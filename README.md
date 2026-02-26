# Blaze RPC

**Zero-copy, lock-free JSON-RPC acceleration for Agave validators**

Blaze Sidecar attaches to a real Agave validator and serves `getAccountInfo`, `getSlot` (and more later) with sub-microsecond latency — directly from the live `Bank` state via shared memory.

No serialization. No network hops. No locks in the validator hot path.

## Why Blaze?

- **50–100× faster reads** than vanilla Agave RPC or remote providers (p99 < 2 ms vs 200–650 ms)
- Zero impact on replay / consensus (only hooks on `Bank::freeze`)
- True zero-copy: account data stays in shared mmap
- Lock-free producer (validator) using [anza-xyz/shaq](https://github.com/anza-xyz/shaq)
- Separate sidecar process with jsonrpsee server

Ideal for co-located high-frequency bots, indexers, wallets, MEV searchers, or as an acceleration layer for RPC providers.

## Architecture

       VALIDATOR PROCESS

              ReplayStage thread
                     │
                     │ Bank::freeze()
                     ▼
                Sidecar Hook
                     │
                     │
                     ├──────────────┐
                     │              │
                     ▼              ▼
               rts-alloc       shaq queue
             allocate data    push envelope
             copy account     offset only
                     │              │
                     └──────┬───────┘
                            │
                            ▼

                    SHARED MEMORY REGION
            +--------------------------------+
            | Header                         |
            | shaq ring buffer               |
            | allocator slabs                |
            | account data blobs             |
            +--------------------------------+

                                │
                                ▼

                        SIDECAR PROCESS
                                │
                                ▼
                        shaq consumer
                                │
                                ▼
                        reconstruct pointer
                                │
                                ▼
                        build HashMap index
                                │
                                ▼
                        jsonrpsee RPC server
