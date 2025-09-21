### Problem
As Solana continues to scale, the demand for fast, reliable and ergonomic access to on-chain data has grown substantially. Individual teams and RPC providers alike struggle to serve the high-throughput workloads necessary to power Solana applications with the existing stack. Many opt to build custom modifications and implementations that do not become upstreamed or open source.

Currently, stock account-based queries rely heavily on outdated, monolithic implementations of the JSON RPC API, unmaintained indexes and a brittle geyser system that was never designed for scale, modularity or ergonomics. As a result, serving account data at the tip of the chain remains an operational bottleneck and a source of frustration for developers.

Account-based RPC endpoints such as getProgramAccounts frequently time out or fail under load, especially when interacting with high-demand programs like Raydium, Pump and Jupiter
Geyser plugins today require heavy monkey-patching and offer limited flexibility for advanced filtering and deduplication
Account queries are deeply coupled to legacy implementations with no clear path to partial domain execution or optimized indexes
Many RPC providers are forced to roll their own custom indexes and replication solutions to serve account data reliably, going as far as running RPCs specific to certain endpoints for certain programs (e.g. getLargestTokenAccounts only for Meteora)

### Proposed Solution
Develop a standalone accounts domain RPC service. This should be a separate RPC decoupled from the validator codebase, fed exclusively from a geyser stream without requiring directly receiving shreds from turbine.
Re-implement all existing JSON-RPC calls such as simulateTransaction  , getRecentPrioritizationFees and getAccountInfo
Build a simulateTransaction JSON-RPC call that allows for JIT account state or simulation that loads from existing bank state
Optimize existing account endpoints. Develop high-performance implementations of getProgramAccounts, getTokenAccountsByOwner and getMultipleAccounts with advanced filtering and batching
Optimize the existing SPL token queries to reduce dependency on generic GPA calls, with restrictions on ambiguous filters that lead to expensive full scans.
Implement robust websocket subscriptions for account changes and slot updates with improved event reliability compared to current geyser implementations.