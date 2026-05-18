# Distributed Retrieval Pipeline Demo

This walkthrough presents the repo as distributed infrastructure for retrieval
task scheduling, execution, retries, and result aggregation.

![Distributed retrieval pipeline demo](assets/screenshots/retrieval-pipeline-demo.png)

## Flow Chart

```mermaid
flowchart LR
    A[Work sources] --> B[Ingestion]
    B --> C[Scheduler]
    C --> D[Provider resolver]
    D --> E[Process manager]
    E --> F[HTTP worker]
    E --> G[Graphsync worker]
    E --> H[Bitswap worker]
    F --> I[Result emitter]
    G --> I
    H --> I
    I --> J[Analytics and cache builders]
```

## Sequence Diagram

```mermaid
sequenceDiagram
    participant Source as Work Source
    participant Scheduler as Scheduler
    participant Resolver as Resolver
    participant Worker as Protocol Worker
    participant Store as Result Store
    participant Downstream as Analytics/Cache Jobs

    Source->>Scheduler: enqueue retrievable item
    Scheduler->>Resolver: enrich provider metadata
    Resolver-->>Scheduler: addresses + geo/connectivity
    Scheduler->>Worker: dispatch task with timeout policy
    Worker-->>Store: success/failure record
    Store-->>Downstream: idempotent result event
```

## Task Entities

```mermaid
erDiagram
    RETRIEVAL_TASK ||--|| PROVIDER_METADATA : enriched_by
    RETRIEVAL_TASK ||--o{ WORKER_ATTEMPT : executes
    WORKER_ATTEMPT ||--|| RESULT_RECORD : emits
    RESULT_RECORD ||--o{ DOWNSTREAM_JOB : triggers

    RETRIEVAL_TASK {
      string piece
      string payload
      string module
      string status
    }
    WORKER_ATTEMPT {
      string protocol
      int timeout_ms
      int retry_count
    }
```

## Sample Result Record

```json
{
  "task_id": "piece-1842",
  "provider": "f01234",
  "protocol": "http",
  "success": true,
  "duration_ms": 2104,
  "bytes_read": 1048576,
  "retries": 0,
  "emitted_at": "2026-05-17T20:30:00Z"
}
```
