# distributed-retrieval-pipeline

Distributed retrieval pipeline for scheduling, resolving, executing, and
recording retrieval tasks across heterogeneous network providers.

The current implementation targets Filecoin-compatible retrieval flows, but the
useful engineering pattern is broader: ingest candidate work, enrich it with
provider metadata, dispatch protocol-specific workers, apply timeouts/retries,
and persist results for downstream analytics.

## System Shape

```
work sources
   |
   v
ingestion + scheduler
   |
   v
provider / location resolver
   |
   v
process manager
   |
   +--> HTTP workers
   +--> Graphsync workers
   +--> Bitswap workers
   |
   v
result emitter
   |
   v
analytics, cache builders, and verification jobs
```

## Core Components

### Ingestion And Scheduler

Collects retrievable items from chain, index, or internal sources and turns
them into deduplicated tasks. The scheduler supports grouping, sampling,
batching, and throttling so large task sets do not overload workers or remote
providers.

### Resolver

Enriches tasks before execution:

- maps provider IDs to libp2p multiaddrs
- extracts public IP and port information
- attaches geolocation and connectivity metadata
- gives workers concrete addresses to attempt

### Process Manager

Runs protocol workers with configured concurrency, restarts failed processes,
and attaches labels to logs. This keeps worker orchestration separate from the
retrieval protocols themselves.

### Task Workers

Protocol-specific workers poll the task queue, execute retrieval attempts, and
record results.

- **HTTP worker**: downloads via HTTP retrieval endpoints.
- **Graphsync worker**: retrieves through Graphsync.
- **Bitswap worker**: retrieves hot data through Bitswap.

Workers use connection, first-byte, and overall timeouts with retry/backoff
behavior.

### Result Emitter

Emits idempotent success/failure records for downstream systems such as cache
builders, analytics services, and verification jobs.

## Side-Node Cache Integration

Successful full retrievals can trigger a cache-building flow:

```
verified full retrieval
      |
      v
full tree builder
      |
      v
window path / side-node cache
      |
      v
future partial retrieval verification
```

The cache path is useful when later clients only need a byte range plus a small
proof path instead of another full-object download. Incorrect cache data fails
root validation, so the cache is treated as untrusted.

## Operational Concerns

- horizontal worker scaling
- task sharding and deduplication
- timeout and retry policy
- provider reachability tracking
- cache-hit and fallback metrics
- correlation IDs from task creation through result emission
- versioned hashing, window, and proof parameters for reproducibility

## Repository Notes

This repo is intentionally closer to infrastructure plumbing than a UI demo.
The interesting parts are the queue boundaries, worker lifecycle, protocol
abstractions, and result records that make retrieval performance measurable.

Related components live under `integration/`, including importers and query
services for retrieval statistics.
