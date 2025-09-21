
# RPCv2 **Accounts** — Architecture

> Standalone **Accounts domain RPC** for Solana RPC v2.  
> Ingests from **Geyser (Yellowstone gRPC)** → materializes **current account state** and **program‑aware indexes** → serves **optimized JSON‑RPC** and **reliable WebSocket** subscriptions.  
> License target: **Apache‑2.0**.

---

## 0) Goals (from the RFP, scoped to Accounts)

- **Decouple** account RPC from the validator; ingest **only** from Geyser (no Turbine/consensus).
- Re‑implement & **optimize** account endpoints:
  - `getAccountInfo`, `getMultipleAccounts`, `getProgramAccounts`, `getTokenAccountsByOwner`, `getLargestTokenAccounts`, `getRecentPrioritizationFees`, and `simulateTransaction` (JIT-capable).
- **Robust subscriptions** for account/slot updates: resumable cursors, dedupe, and overflow/backfill signaling.
- SLOs to target (tunable):
  - **Latency:** p95 ≤ **200 ms** for `getMultipleAccounts` & token queries; p95 ≤ **250 ms** for GPA with prefix filters.
  - **Reliability:** WS delivery ≥ **99.95%** monthly; replayable via resume token.
  - **Efficiency:** ≥ **90%** SPL‑Token reads served from indexes (no full scans).

---

## 1) High‑level Architecture

### ASCII (fallback)

```
Validators (>=2)
   │         ┌──────────────────────────┐
   ├─Geyser──► 1) Ingest + Dedupe       │
   │         └────────────┬─────────────┘
   │                      │  CDC Events
   │         ┌────────────▼─────────────┐
   │         │ 2) CDC Log / Event Bus   │
   │         └───────┬──────────┬───────┘
   │                 │          │
   │     ┌───────────▼───┐  ┌───▼─────────────────┐
   │     │ 3A) State KV  │  │ 3B) Index Builders  │
   │     │ (current acct)│  │  (SPL-Token, GPA)   │
   │     │ Scylla/Rocks  │  │  → ClickHouse       │
   │     └───────┬───────┘  └────────┬────────────┘
   │             │                   │
   │   ┌─────────▼─────────┐   ┌─────▼─────────────┐
   │   │ 4) RPC HTTP       │   │ 5) WS Gateway     │
   │   │ (axum)            │   │ (resume tokens)   │
   │   └────────┬──────────┘   └────────┬──────────┘
   │            │                       │
   └────────────▼───────────────────────▼─────────────► Clients
```

---

## 2) Repository Layout

```
rpcv2-accounts/
  crates/
    ingest-geyser/         # Yellowstone gRPC client, normalizer, dedupe
    cdc-bus/               # Kafka/Redpanda or NATS JetStream producers/consumers
    state-kv/              # ScyllaDB client or sharded RocksDB KV
    index-ch/              # ClickHouse writers, query pushdown for GPA/Token
    rpc-http/              # JSON-RPC server (axum), batching & filter validation
    ws-gateway/            # WebSocket server (resume tokens, dedupe, overflow)
    simulate/              # simulateTransaction (Phase 1 delegate, Phase 2 bank)
    conformance/           # wire-compat, correctness, commitment semantics
    bench/                 # load & latency harness, WS reliability tests
    account-history/       # optional: account versioning and historical state tracking
  deploy/                  # Helm/Terraform, dashboards, alerts, sample configs
```

---

## 3) Components (what, how, tech choices)

### 3.1 `ingest-geyser/` — Yellowstone gRPC client & normalizer
**Purpose:** Connect to multiple Geyser endpoints; receive account/slot updates; normalize to one event shape and dedupe.

**Event shape:**
```rust
pub struct AccountChange {
  pub slot: u64,
  pub write_version: u64,
  pub transaction_index: u32,     // intra-slot ordering for proper sequencing
  pub pubkey: [u8; 32],
  pub owner: [u8; 32],
  pub lamports: u64,
  pub data: bytes::Bytes,         // optional: store blob externally and keep a hash
  pub data_hash: [u8; 32],        // integrity verification for external storage
  pub rent_epoch: u64,
  pub source_id: String,          // which validator feed
  pub bank_hash: [u8; 32],        // fork identification for proper reorg handling
  pub commitment_watermarks: CommitmentWatermarks, // proper watermark tracking
}

pub struct CommitmentWatermarks {
  pub processed_slot: u64,
  pub confirmed_slot: u64,
  pub finalized_slot: u64,
}
```
**Idempotency key:** `(slot, write_version, transaction_index, pubkey)`  
**Fork-aware reorg buffer:** Track `bank_hash` lineage with configurable depth (default: 300 slots) to handle Solana's complex fork scenarios and slot skipping patterns.

**Tech:** Rust (`tokio`, `tonic`/`prost`)  
**Why:** High throughput, low latency, strongly typed gRPC.  
**Alt:** WebSocket relay; not recommended for peak throughput.

---

### 3.2 `cdc-bus/` — Durable Event Log
**Purpose:** Decouple ingestion from serving; allow replay and fan‑out.

- **Option A: Redpanda / Kafka** — partitions, retention, exactly‑once-ish with idempotent writes.
- **Option B: NATS JetStream** — simpler ops; at‑least‑once; lower latency; good for WS.

**Why a CDC log?**  
- Recover from crashes by replay.  
- Multiple consumers (materializers, WS, backfillers) scale independently.  
- Stable ordering via `(slot, write_version)` within a partitioning strategy (e.g., hash(pubkey) to minimize cross‑partition reorders).

---

### 3.3 `state-kv/` — Current Account State (hot path)
**Purpose:** Serve `getAccountInfo`/`getMultipleAccounts` quickly.

- **Schema (KV):** `pubkey -> {owner, lamports, data_hash|ptr, slot, write_version}`
- **Choices:**
  - **ScyllaDB:** distributed, predictable p99 for random reads/writes; operationally proven.
  - **Sharded RocksDB:** embedded speed; you manage shard routing/compaction.
- **Why:** Reads are frequent & random; KV is ideal.

**Tips:**  
- Optional short‑TTL Redis for the hottest pubkeys.  
- Store large `data` blobs off‑heap (object storage) when needed; keep a `data_hash` for integrity.

---

### 3.4 `index-ch/` — Program‑aware Indexes (heavy queries)
**Purpose:** Make `getProgramAccounts` and token queries fast without full scans.

- **ClickHouse tables:**
  - `token_accounts(owner, mint, amount, pubkey, slot)`
  - `mints(mint, decimals, supply, authorities, slot)`
  - `gpa_index(program_id, field_tag, prefix, pubkey, slot)` where `prefix` are short extracted fields used for **memcmp prefix** filters.
- **Partitioning:** by `slot_day` (or time); **ORDER BY** `(program_id, prefix, pubkey, slot)`
- **Why ClickHouse:** Columnar, vectorized, blazingly fast for selective scans and top‑N aggregations.

**Behavior & Security:**  
- Only **exact/prefix** filters are supported for GPA with strict validation.
- **Resource limits:** Max 1MB account data size, max 1000 results per query, max 5 concurrent GPA requests per client.
- **Rate limiting:** Program-specific limits (e.g., 10 req/min for popular programs like Token, 100 req/min for others).
- **Filter validation:** Reject ambiguous filters with detailed `explain` payload suggesting supported alternatives.
- **DoS protection:** Query cost estimation and early termination for expensive operations.

---

### 3.5 `rpc-http/` — JSON‑RPC Server (Accounts domain)
**Endpoints:**  
- `getAccountInfo`, `getMultipleAccounts` (with adaptive batching & duplicate coalescing)  
- `getProgramAccounts` (index‑backed with filter pushdown)  
- `getTokenAccountsByOwner`, `getLargestTokenAccounts`  
- `getRecentPrioritizationFees` (slot/leader‑aware cache)  
- `simulateTransaction` (see 3.7)

**Semantics:**  
- **Commitment‑aware:** respect processed/confirmed/finalized watermarks.  
- **Adaptive batching:** auto‑size batches to hit p95 SLOs.  
- **Explain mode:** when rejecting filters or falling back, return a JSON `explain` object.

**Tech:** Rust (`axum`/`hyper`, `tower`), `moka` (in‑proc cache)  
**Why:** High performance, clean middleware, good ecosystem.

---

### 3.6 `ws-gateway/` — WebSocket Subscriptions
**Topics:** `account`, `program`, `slot`

**Delivery model:**  
- **At‑least‑once** with client dedupe using `{slot, write_version, transaction_index, pubkey}`.  
- **Enhanced resume tokens:** 
```rust
pub struct ResumeToken {
  pub slot: u64,
  pub write_version: u64,
  pub commitment_level: CommitmentLevel,
  pub sequence_number: u64,      // per-subscription ordering guarantee
  pub checkpoint_hash: [u8; 32], // integrity verification for replay
  pub subscription_id: String,   // scoped to specific subscription
}
```
- **Reliable replay:** Resume tokens provide exactly-once replay semantics within the retention window.
- **Overflow handling:** Progressive backoff with detailed overflow metrics and automatic client backfill coordination.

**Tech:** Rust (`tokio-tungstenite`), consumes directly from **CDC** consumer groups.  
**Why:** Shared backbone keeps ordering consistent across HTTP and WS.

---

### 3.7 `simulate/` — Simulation Service (JIT)
**Phase 1 (faster):** Delegate simulate upstream, but **preload** referenced accounts from `state-kv` to avoid cache misses; cache the read‑set for next calls.  
**Phase 2 (local simulation):** Build a **read‑only bank** from latest finalized slot + pending deltas and run simulation locally using Agave runtime crates.

**Phase 2 Limitations & Scope:**
- **Supported:** Basic account reads, lamport transfers, simple SPL Token operations
- **Limitations:** Complex CPI chains, sysvars that change per slot, advanced runtime features
- **Integration:** Requires coordination with Anza team for proper runtime crate usage
- **Fallback:** Unsupported simulation types gracefully fall back to Phase 1 delegation

**Return:** logs, post‑balances, compute units consumed, optional account diffs.

**Why staged:** Ship value early while building deeper simulation capabilities; clear scoping prevents over-promising on complex runtime integration.

---

### 3.8 `conformance/` — Wire‑compat & Correctness
- Golden tests vs. a reference RPC for core endpoints.  
- GPA filter behavior matrix (which filters are supported/rejected).  
- Commitment semantics tests.  
- WS delivery/replay tests.

**Goal:** Anyone can verify correctness quickly.

---

### 3.9 `bench/` — Load & Latency Harness
- Mixed workload generation (read heavy + WS).  
- Metrics: p50/p95/p99, WS drop/retry rate, slot lag, index hit‑rate.  
- 24–48h soak to validate SLOs before releases.

---

### 3.10 `account-history/` — Optional Account Versioning (Future Enhancement)
**Purpose:** Track historical account states for advanced use cases (auditing, time-travel queries, analytics).

```rust
pub trait AccountHistoryStore {
  async fn get_account_at_slot(&self, pubkey: Pubkey, slot: u64) -> Result<Option<Account>>;
  async fn get_account_changes_range(&self, pubkey: Pubkey, start_slot: u64, end_slot: u64) -> Result<Vec<AccountChange>>;
  async fn get_account_version_count(&self, pubkey: Pubkey) -> Result<u64>;
}
```

**Implementation:** Optional ClickHouse tables with compressed historical states.  
**Use cases:** DeFi protocol auditing, account lifecycle analysis, forensic investigation.  
**Scope:** Not in initial RFP but architectural hooks provided for future extension.

### 3.11 `deploy/` — Ops
- Helm/Terraform, multi‑AZ example topology.  
- Prometheus/Grafana dashboards + alert rules (CDC gaps, slot lag, p95 breaches, WS overflow, fork detection).  
- Example configs: validator endpoints, index set, filter policy, reorg buffer sizing.

---

## 4) Data Model & Schemas (minimal)

### 4.1 Enhanced CDC Event
```json
{
  "slot": 274839201,
  "write_version": 7,
  "transaction_index": 42,
  "pubkey": "7s...",
  "owner": "Tokenkeg...",
  "lamports": 2039280,
  "data_b64": "...",
  "data_hash": "a1b2c3...",
  "rent_epoch": 0,
  "source_id": "validator-a",
  "bank_hash": "d4e5f6...",
  "commitment_watermarks": {
    "processed_slot": 274839201,
    "confirmed_slot": 274839185,
    "finalized_slot": 274839153
  }
}
```

### 4.2 KV (Scylla) — current account state
```sql
CREATE TABLE accounts_current (
  pubkey TEXT PRIMARY KEY,
  owner TEXT,
  lamports BIGINT,
  data_hash BLOB,
  data_ptr TEXT,              -- optional external blob pointer
  slot BIGINT,
  write_version BIGINT,
  transaction_index INT,      -- intra-slot ordering
  bank_hash BLOB,             -- fork identification
  rent_epoch BIGINT,
  updated_at TIMESTAMP        -- for TTL and cache management
);

-- Optional hot cache table for frequently accessed accounts
CREATE TABLE accounts_hot_cache (
  pubkey TEXT PRIMARY KEY,
  account_data BLOB,          -- full account data for hot path
  ttl_seconds INT             -- configurable cache expiration
) WITH default_time_to_live = 300;
```

### 4.3 ClickHouse — Program-aware optimized schemas
```sql
-- SPL Token accounts with proper versioning and state tracking
CREATE TABLE spl_token_accounts (
  owner String,
  mint String,
  amount UInt128,
  delegate Nullable(String),
  state Enum8('uninitialized'=0, 'initialized'=1, 'frozen'=2),
  pubkey String,
  slot UInt64,
  write_version UInt64,
  transaction_index UInt32
) ENGINE = ReplacingMergeTree(write_version)
PARTITION BY toYYYYMM(toDate(slot * 400 / 86400))  -- ~400ms per slot
ORDER BY (owner, mint, amount DESC, pubkey);

-- Mints with comprehensive metadata
CREATE TABLE spl_mints (
  mint String,
  decimals UInt8,
  supply UInt128,
  mint_authority Nullable(String),
  freeze_authority Nullable(String),
  is_initialized Bool,
  pubkey String,
  slot UInt64,
  write_version UInt64
) ENGINE = ReplacingMergeTree(write_version)
ORDER BY (mint, slot);

-- Generic GPA index with bounded prefix extraction
CREATE TABLE gpa_index (
  program_id String,
  field_hash UInt64,        -- hash of field position for efficient lookup
  prefix_4 FixedString(4),  -- first 4 bytes for common memcmp filters
  prefix_8 FixedString(8),  -- first 8 bytes for extended filters
  data_size UInt32,         -- account data size for filtering
  pubkey String,
  slot UInt64,
  write_version UInt64
) ENGINE = ReplacingMergeTree(write_version)
PARTITION BY toYYYYMM(toDate(slot * 400 / 86400))
ORDER BY (program_id, field_hash, prefix_4, prefix_8, pubkey);

-- Program-specific optimizations (example: Stake accounts)
CREATE TABLE stake_accounts (
  voter_pubkey String,
  withdrawer_authority String,
  staker_authority String,
  stake_lamports UInt64,
  activation_epoch UInt64,
  deactivation_epoch Nullable(UInt64),
  pubkey String,
  slot UInt64,
  write_version UInt64
) ENGINE = ReplacingMergeTree(write_version)
ORDER BY (voter_pubkey, stake_lamports DESC, pubkey);
```

---

## 5) Request Flows

### 5.1 `getMultipleAccounts`
1. Client calls RPC with a list of pubkeys.  
2. `rpc-http` **coalesces duplicates** and groups reads by shard/partition.  
3. Fetch from **state-kv** (and optional Redis for hottest keys).  
4. Return results with `context.slot` per requested commitment.

**Why fast:** KV lookup per key, batched; minimal CPU.

### 5.2 `getProgramAccounts` (with filters)
1. Client sends filters (owner, **memcmp prefix** over extracted fields).  
2. `rpc-http` validates; **rejects ambiguous** filters with `explain`.  
3. Query **ClickHouse** index to resolve matching pubkeys fast.  
4. Hydrate account bodies from **state-kv** (or join projection).  
5. Return array of accounts.

**Why scalable:** No full scans; all filters map to indexed columns.

---

## 6) Fork-aware Reorg Handling & Exactly‑once Semantics

- **Enhanced idempotency:** `(slot, write_version, transaction_index, pubkey)` key used across CDC → sinks.  
- **Fork lineage tracking:** Maintain `bank_hash` chains to detect and handle complex Solana fork scenarios.
- **Ordering guarantees:** Within partition, sort by `(slot, write_version, transaction_index)`; track per-source watermarks with gap detection.  
- **Intelligent rollback:** 
  - **Buffer depth:** Configurable 300-slot reorg buffer (default) with emergency expansion capability
  - **Rollback detection:** Monitor `bank_hash` discontinuities and slot gaps to trigger replay
  - **State reconciliation:** Atomic rollback of `state-kv` and ClickHouse indexes using write_version-based cleanup
- **Commitment progression:** Track processed/confirmed/finalized watermarks per validator source with Byzantine fault tolerance (require 2/3 majority for finalization).

---

## 7) Tech Stack Summary (and why)

- **Language:** **Rust** (tokio, axum, tonic, tungstenite) — performance + type safety.  
- **CDC Bus:** **Redpanda/Kafka** (or **NATS JetStream**) — replay & fan‑out.  
- **State KV:** **ScyllaDB** (or sharded RocksDB) — random R/W at scale.  
- **Indexes:** **ClickHouse** — columnar speed for selective scans/top‑N.  
- **Cache:** **Redis** (optional) — hottest keys.  
- **Metadata:** **Postgres** — tenants, API keys, config.  
- **Observability:** **OpenTelemetry → Prometheus/Grafana** with Solana-specific metrics:
  ```rust
  pub struct SolanaRpcMetrics {
    pub slots_behind_tip: Gauge,                    // current lag from chain tip
    pub commitment_level_latencies: HistogramVec,   // by processed/confirmed/finalized
    pub account_versions_skipped: Counter,          // reorg-related version conflicts
    pub gpa_filter_rejections: CounterVec,          // by rejection reason
    pub websocket_overflow_events: Counter,         // client backpressure events
    pub index_hit_rate: GaugeVec,                   // by program_id and query type
    pub fork_detection_events: Counter,             // bank_hash discontinuities
    pub state_kv_cache_hit_rate: Gauge,            // Redis/ScyllaDB hit rates
    pub simulation_fallback_rate: Gauge,            // Phase2 → Phase1 fallbacks
    pub cdc_replay_events: Counter,                 // recovery operations
  }
  ```
- **Packaging:** Helm/Terraform; multi‑AZ examples.

---

## 8) Implementation Phases & Risk Mitigation

### Phase 1: Foundation & Core Infrastructure (Weeks 1–8)
**Deliverables:**
- Geyser ingestion with fork-aware reorg handling
- Enhanced CDC log with commitment watermark tracking  
- ScyllaDB state store with proper versioning
- Basic `getAccountInfo/getMultipleAccounts` endpoints
- WebSocket subscriptions with enhanced resume tokens
- Comprehensive observability and alerting

**Risk Mitigation:**
- Start with proven patterns (ScyllaDB over experimental stores)
- Validate reorg handling with testnet fork scenarios
- Build performance benchmarks early with real validator data

**Acceptance:** 48h soak test; p95 ≤ 250 ms at target QPS; WS ≥ 99.9%; clean reorg recovery.

### Phase 2: Program-Aware Indexing (Weeks 9–16)  
**Deliverables:**
- ClickHouse integration with optimized schemas
- SPL Token and generic GPA indexes
- Filter validation with security bounds
- `getProgramAccounts`, `getTokenAccountsByOwner`, largest holders
- Program-specific query optimizations

**Risk Mitigation:**  
- Validate ClickHouse performance with real-world GPA queries
- Build comprehensive filter rejection test matrix
- Partner with RPC providers for query pattern validation

**Acceptance:** 0 full scans on SPL-Token; p95 ≤ 200 ms; 90%+ index hit rate; correctness vs golden dataset.

### Phase 3: Production Hardening (Weeks 17–24)
**Deliverables:**
- Byzantine fault tolerant commitment progression
- Enhanced security and DoS protection
- `simulateTransaction` Phase 1 (delegation with preloading)
- Conformance suite and public benchmarks
- Production deployment guides

**Risk Mitigation:**
- Extensive chaos engineering and failure injection testing
- Security audit of filter validation and rate limiting
- Load testing with RPC provider partnership

**Acceptance:** WS ≥ 99.95% over 7 days; successful chaos drills; security audit passed.

### Phase 4: Advanced Features (Weeks 25–32, Optional)
**Deliverables:** 
- `simulateTransaction` Phase 2 (local bank simulation) 
- Account history tracking (optional)
- Advanced program-specific optimizations
- Multi-region deployment support

**Risk Mitigation:**
- Clear scoping of simulation limitations
- Agave team collaboration for runtime integration
- Gradual rollout with fallback mechanisms

**Acceptance:** Simulation parity on scoped programs; error rate < 0.1%; optional features stable.

---

## 9) Security, Multitenancy, Policy & Risk Mitigation

- **Auth:** API keys with per‑tenant scopes (read, subscribe, program-specific access).  
- **Rate limits/quotas:** Multi-layered rate limiting:
  - Per endpoint (e.g., 100 GPA req/min, 1000 getAccount req/min)
  - Per WebSocket channel (max 10 subscriptions)
  - Per program (Token: 10 req/min, others: 100 req/min)
  - Fair scheduling with priority queues for different client tiers
- **Filter policy:** Only index‑backed GPA filters allowed; comprehensive validation with detailed `explain` responses.  
- **Data integrity:** Content hashes for external blobs; optional signature anchors in CDC for audit trails.
- **DoS protection:** 
  - Query cost estimation and early termination
  - Resource usage monitoring per client
  - Circuit breakers for expensive operations
- **Byzantine fault tolerance:** Require 2/3 validator consensus for finalized commitment progression.

### 9.1 Program-Specific Query Optimizations
```rust
pub enum ProgramSpecificQuery {
  // SPL Token optimizations
  TokenAccountsByMint { mint: Pubkey, limit: usize },
  TokenLargestHolders { mint: Pubkey, limit: usize },
  TokenAccountsByDelegate { delegate: Pubkey },
  
  // Stake program optimizations  
  StakeAccountsByVoter { voter: Pubkey },
  StakeAccountsByWithdrawer { withdrawer: Pubkey },
  ActiveStakeByEpoch { epoch: u64 },
  
  // Common patterns that bypass generic GPA
  AccountsByOwnerAndSize { owner: Pubkey, data_size: u32 },
  RecentlyModifiedAccounts { since_slot: u64, program: Pubkey },
}
```

---

## 10) Local Dev & Testing

- Docker Compose profile with: Redpanda (or NATS), Scylla (or Rocks), ClickHouse, Postgres, Grafana+Prometheus.  
- Fake Geyser generator to replay recorded slots for fast iteration.  
- `conformance/` tests runnable in CI (GitHub Actions).

---

## 11) Configuration (example)

```toml
[geyser]
endpoints = ["grpc://validator-a:10000", "grpc://validator-b:10000"]
commitment = "confirmed"

[cdc]
backend = "redpanda"
brokers = ["broker-1:9092","broker-2:9092"]
topic = "accounts.cdc.v1"

[state_kv]
backend = "scylla"
nodes = ["scylla-1:9042","scylla-2:9042","scylla-3:9042"]

[indexes]
clickhouse_url = "tcp://ch-1:9000"
enable_spl_token = true

[rpc]
listen_addr = "0.0.0.0:8899"
batch_max = 512

[ws]
listen_addr = "0.0.0.0:8900"
resume_window_slots = 300
```

---

## 12) Ecosystem Integration & Competitive Advantages

### 12.1 Integration Strategy
- **RPC Provider Partnerships:** Collaborate with Helius, Triton, QuickNode for real-world validation
- **Anza/Foundation Alignment:** Regular technical reviews and architecture coordination  
- **Community Feedback:** Open-source core components early for ecosystem input
- **Standards Compliance:** Full JSON-RPC compatibility with enhanced reliability guarantees

### 12.2 Key Differentiators vs. Current Solutions
- **Solana-native design:** Built specifically for Solana's unique challenges (forks, commitment levels, slot gaps)
- **Program-aware optimization:** Unlike generic RPC stacks, optimized for actual Solana program patterns
- **Production-ready reliability:** 99.95% uptime SLO with proper Byzantine fault tolerance
- **Comprehensive observability:** Deep Solana-specific metrics and alerting
- **Vendor-neutral:** Open source with multiple deployment options (unlike proprietary solutions)

### 12.3 Learning from Alpamayo & Beyond
**Alpamayo insights applied:**
- ✅ Proven RocksDB patterns adapted for our use case
- ✅ Upstream fallback strategies for reliability  
- ✅ Thread-based storage patterns evolved to async Rust
- ✅ Smart caching and request batching approaches

**Beyond Alpamayo limitations:**
- 🚀 Real-time account state (vs. historical-only focus)
- 🚀 Program-aware indexing (vs. generic block storage)  
- 🚀 WebSocket subscriptions with resume tokens
- 🚀 Advanced simulation capabilities
- 🚀 Fork-aware reorg handling for live data

## 13) License

Target license: **Apache‑2.0** (or MIT) for core code, schemas, conformance suite, and deployment assets.
