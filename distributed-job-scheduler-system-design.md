# Production Distributed Job Scheduler

## 1. Requirements

### Functional requirements

1. Submit immediate, delayed, recurring, and dependency-based jobs.
2. Dispatch work to heterogeneous worker pools using resource and placement constraints.
3. Support retries, timeouts, cancellation, priorities, rate limits, and deadlines.
4. Prevent two healthy workers from owning the same attempt intentionally.
5. Expose status, progress, logs, history, and searchable metadata.
6. Support cron schedules with explicit time zones and daylight-saving behavior.
7. Handle workflows/DAGs and fan-out/fan-in dependencies.
8. Provide dead-lettering and manual replay.

### Non-functional requirements

- Schedule tens of millions of jobs per day with second-level dispatch latency.
- Remain available across machine and availability-zone failures.
- Never lose an accepted job.
- Provide at-least-once execution; clearly define idempotency expectations.
- Isolate tenants and prevent starvation.
- Scale the control plane independently from worker execution.

Exactly-once side effects are out of scope because the scheduler cannot atomically commit arbitrary external work. It provides fencing and idempotency tools instead.

## 2. Scale assumptions and capacity

Assume:

- 100 million job runs/day: about 1,160/second average and 20,000/second peak.
- 10 million recurring definitions.
- 5 million simultaneously running attempts.
- 200,000 worker nodes across many pools.
- 1 KB average job metadata and 20 state transitions per run.
- 30 days of searchable hot history and one year archived.

```text
100M runs/day x 20 transitions x 300 bytes
= about 600 GB/day of unreplicated event history
```

Payloads and logs belong in object storage; queue rows contain references. Large scheduled bursts, such as midnight cron jobs, matter more than average throughput.

## 3. High-level architecture

```mermaid
flowchart LR
    C[Clients / Workflow Service] --> API[Job API]
    API --> DB[(Sharded Job Store)]
    API --> OUT[(Transactional Outbox)]
    OUT --> BUS[(Durable Event Log)]
    T[Timer / Cron Shards] --> DB
    T --> BUS
    BUS --> Q[Ready Queue Partitions]
    S[Scheduler Fleet] --> Q
    S --> M[Worker Membership / Capacity]
    S --> W[Worker Agents]
    W --> HB[Lease / Heartbeat Service]
    HB --> DB
    W --> OBJ[(Payloads and Logs)]
    R[Reconciler] --> DB
    R --> Q
    API --> IDX[(History Search Index)]
```

The durable job store is authoritative. Ready queues accelerate dispatch but are reconstructable. Schedulers decide placement; workers execute untrusted or isolated workloads.

## 4. Data model and state machine

```text
JobDefinition
- job_id
- tenant_id
- command_or_handler
- payload_uri
- schedule
- timezone
- retry_policy
- timeout
- priority
- constraints
- revision

JobRun
- run_id
- job_id
- scheduled_for
- state
- remaining_dependencies
- attempt_count
- deadline
- idempotency_key
- version

Attempt
- attempt_id
- run_id
- number
- worker_id
- lease_epoch
- lease_expires_at
- started_at / finished_at
- result_uri
- failure_class

Dependency
- parent_run_id
- child_run_id
- condition: SUCCESS | COMPLETION

StateEvent
- run_id
- sequence
- from_state / to_state
- actor
- timestamp
```

Core run transitions:

```text
SCHEDULED -> READY -> DISPATCHED -> RUNNING -> SUCCEEDED
                     |             |-> RETRY_WAIT -> READY
                     |             |-> FAILED
                     |             |-> TIMED_OUT
                     |             |-> CANCELED
                     |-> READY (lease expires before start)
```

Every transition uses compare-and-swap on `version` and appends a state event in the same transaction.

## 5. Core APIs and worker protocol

```http
POST /v1/jobs
Idempotency-Key: client-uuid

{
  "handler": "generate-invoice:v3",
  "payload_uri": "obj://payloads/...",
  "run_at": "2026-10-13T09:00:00Z",
  "timeout_seconds": 900,
  "retry": {"max_attempts": 5, "backoff": "exponential"},
  "constraints": {"pool": "finance", "cpu": 2, "memory_mb": 4096}
}
```

```http
POST   /v1/jobs/{job_id}:pause
POST   /v1/jobs/{job_id}:resume
POST   /v1/runs/{run_id}:cancel
POST   /v1/runs/{run_id}:retry
GET    /v1/runs/{run_id}
GET    /v1/runs?tenant_id=...&state=FAILED
GET    /v1/runs/{run_id}/events
```

Workers use a bidirectional gRPC stream:

```text
Register(capabilities, pool, resources)
Offer(attempt_id, lease_epoch, payload_ref, deadline)
Accept(attempt_id, lease_epoch)
Heartbeat(attempt_id, lease_epoch, progress)
Complete(attempt_id, lease_epoch, result_ref)
```

Every worker mutation includes the lease epoch. A stale worker receives an explicit fenced error.

## 6. Job submission flow

1. The API authenticates the tenant, validates limits and schedule syntax, and stores large payloads separately.
2. In one transaction it inserts the definition/run, idempotency record, audit event, and outbox record.
3. Immediate jobs enter `READY`; future jobs enter `SCHEDULED`.
4. An outbox publisher emits ready events to the appropriate queue partition.
5. The client receives the stable job/run ID. A retry with the same key and equivalent request returns it; a different request is rejected.

The outbox closes the gap between committing a job and publishing it. A reconciler periodically scans authoritative `READY` rows without active dispatch records and republishes them.

## 7. Timers, cron, and delayed jobs

Partition timers by a hash of `job_id` and time bucket. Each timer shard owns a renewable, fenced lease and scans an indexed window such as:

```text
WHERE shard = ? AND next_fire_time <= now + lookahead
```

Near-term timers enter a hierarchical timing wheel or priority heap; far-future timers remain in the database. On firing, a conditional transaction creates a uniquely keyed run `(job_id, scheduled_for)` and computes the next occurrence.

Cron definitions store the original expression, IANA time zone, and explicit daylight-saving policy:

- Missing local time: skip or run at the next valid instant.
- Repeated local time: run once or twice.
- Misfire: catch up, coalesce to one run, or skip.

Clock skew is bounded with synchronized clocks, but database compare-and-swap—not local time alone—prevents double creation.

## 8. Dispatch and placement

Ready queues are partitioned by pool and priority band. Scheduler instances consume partitions under leases and maintain a soft cache of worker capacity.

Placement filters by:

- Required runtime, labels, region, and compliance zone.
- CPU, memory, GPU, and ephemeral disk.
- Tenant concurrency and rate limits.
- Affinity/anti-affinity and data locality.
- Deadline and priority.

Then it scores eligible workers by remaining capacity, locality, fragmentation, and recent failure rate. The scheduler creates an `Attempt` and transitions `READY -> DISPATCHED` before offering it. If the worker does not accept promptly, the offer expires and the run is requeued.

Use deficit round robin or weighted fair queuing across tenants, with reserved capacity for critical classes. Aging prevents low-priority starvation.

## 9. Execution, leases, retries, and cancellation

After accepting, the worker starts an isolated process/container and transitions the attempt to `RUNNING`. It renews a short lease and reports progress. Missing heartbeats cause the attempt to become lost; after a grace period the reconciler increments the lease epoch and retries.

Retries distinguish:

- Transient infrastructure or retryable application errors.
- Permanent validation/application failures.
- Scheduler cancellation and deadline expiration.

Backoff uses exponential delay plus jitter and respects a maximum attempts/elapsed-time budget. Cancellation is cooperative first: mark `CANCEL_REQUESTED`, notify the worker, then terminate after a grace period. A completion racing with cancellation wins only through a documented conditional transition.

Worker code receives `run_id`, `attempt_id`, and a stable business idempotency key. For side effects, it should use an idempotency table or transactional inbox in the destination system.

## 10. Workflow and dependency handling

A DAG submission validates acyclicity and creates dependency edges. Each child stores `remaining_dependencies`. When a parent reaches a terminal state, an idempotent dependency consumer records that edge as satisfied and atomically decrements the counter. The child becomes `READY` exactly when the count reaches zero and its conditions hold.

For massive fan-out, do not transact across all children. Emit partitioned parent-completion events and use batched conditional updates. A workflow aggregator computes summary state from durable child transitions. Compensation is application-defined; the scheduler must not pretend arbitrary steps can be rolled back.

## 11. Consistency and concurrency

Strong consistency is required for:

- Run state transitions.
- Attempt ownership and lease epochs.
- Uniqueness of scheduled occurrences and idempotency keys.
- Dependency counters.

Queue delivery, search indexing, logs, and progress are eventually consistent. Queue messages are hints containing IDs; consumers re-read authoritative state and ignore obsolete messages.

Fencing is essential. A lease timestamp alone fails if a paused scheduler resumes. Monotonically increasing `lease_epoch` values let the database, worker, and downstream resource reject stale owners.

## 12. Partitioning and storage

Shard job/run rows by `tenant_id` plus hashed `job_id` to distribute large tenants. Keep a run and its attempts/events together. Separate indexes serve `(state, next_action_time, shard)` scans without globally querying all jobs.

Ready queues partition by execution pool and priority, with virtual partitions for rebalancing. Timer buckets partition by fire time and hash. Payloads, results, and logs use immutable object keys and checksums.

Move terminal event histories to columnar object-storage files after the hot period. Keep compact status rows online and index selected fields in a search engine. Search is never authoritative for scheduling.

## 13. Reliability and failure handling

| Failure | Handling |
|---|---|
| API response lost after commit | Idempotency key returns original job |
| Outbox publish fails | Publisher retries; reconciler republishes |
| Ready message duplicated | Conditional state transition ignores obsolete message |
| Scheduler crashes after dispatch | Attempt lease expires; fenced retry |
| Worker completes after lease loss | Stale epoch rejects completion |
| Worker dies after external side effect | Retry may repeat; handler idempotency is required |
| Timer owner fails | New fenced owner scans overlap window |
| Queue unavailable | Accepted jobs remain in authoritative store |
| Search index lags | Status lookup reads job store |
| Poison job repeatedly crashes workers | Retry budget, quarantine, and dead-letter queue |
| Zone fails | Replicated metadata elects owners in another zone |

Run continuous invariants: no `RUNNING` attempt with an expired unhandled lease, no `READY` job absent from reconciliation, and no child with a negative dependency count.

## 14. Multi-region design

Assign each tenant or job to a home region for state transitions and timer ownership. Stateless APIs route writes home; workers may run elsewhere if latency and data policy permit. Metadata replicates synchronously across availability zones and asynchronously to a disaster-recovery region.

During regional failure:

1. Stop or fence old-region lease issuance.
2. Verify replication position against the accepted recovery-point objective.
3. Promote the secondary and increment regional epochs.
4. Reconstruct ready queues and timer windows from authoritative rows.
5. Treat old workers as stale until they re-register.

Active-active ownership of one run is avoided. Globally submitted jobs can be routed by tenant home while read APIs use replicas.

## 15. Security and isolation

- Authenticate users and workers with short-lived identities and mutual TLS.
- Authorize job types, worker pools, payloads, cancellations, and replay separately.
- Encrypt metadata, payloads, results, and logs with KMS-managed keys.
- Run workloads in sandboxed containers or microVMs with least-privilege service accounts.
- Keep secrets out of job payloads; inject scoped secret references at execution.
- Enforce tenant quotas for submissions, concurrency, CPU time, storage, and retries.
- Validate handler allowlists and prevent arbitrary control-plane command injection.
- Audit definition changes, dispatch, cancellation, secret access, and manual retries.

## 16. Observability

Track:

- Submission rate, validation failures, and idempotent replays.
- Schedule-to-ready, ready-to-dispatch, and dispatch-to-start latency.
- Queue depth and oldest age by pool, priority, and tenant.
- Running attempts, lease renewals, lost workers, and fenced stale writes.
- Success, retry, timeout, cancellation, and dead-letter rates by job type.
- Timer lag, cron misfires, and reconciler repairs.
- Worker resource utilization and placement failures.
- Regional replication lag and failover readiness.

The primary SLI is deadline attainment or schedule-to-start latency, not merely API uptime. Synthetic jobs should verify submission through execution and result retrieval.

## 17. Capacity protection and operations

Admission control rejects or delays workloads beyond tenant quotas. Per-handler circuit breakers can pause jobs when a dependency is failing, avoiding a retry storm. Operators need safe controls to pause a definition, drain a pool, reduce concurrency, quarantine a version, replay dead letters, and inspect every state transition.

Deploy scheduler changes gradually because placement bugs have fleet-wide blast radius. Version worker protocols and allow mixed-version operation during rollout.

## 18. Major trade-offs

### At-least-once versus exactly-once execution

At-least-once survives worker ambiguity. Exactly-once external effects require cooperation from the destination, so expose stable idempotency keys and fencing instead of making an impossible promise.

### Push versus worker polling

Push gives low latency and informed placement. Polling is simpler through firewalls and naturally applies backpressure. A long-lived worker stream with server offers combines both.

### Database timers versus timing wheels

Database indexes are durable but expensive at high frequency. Timing wheels are efficient but volatile. Keep future truth in the database and cache only the near horizon.

### Central versus decentralized scheduling

Central schedulers optimize placement but can bottleneck. Partitioned schedulers by pool retain global policy within a manageable domain.

### Strict priority versus fairness

Strict priority can starve tenants. Weighted fair queues with reserved emergency capacity provide predictable isolation.

## 19. Recommended interview presentation

A clear 45-minute sequence is:

1. Clarify job types, latency, retry semantics, and side-effect guarantees.
2. Estimate run/transition rates, concurrent attempts, and burst patterns.
3. Draw the authoritative store, outbox, ready queues, schedulers, and workers.
4. Walk through submission, dispatch, heartbeat, completion, and retry.
5. Deep-dive into cron timers and fencing tokens.
6. Explain idempotency and why execution is at least once.
7. Cover fair scheduling, dependency counters, and reconciliation.
8. Finish with regional failover, sandboxing, and trade-offs.

The central design insight is:

> Keep job state authoritative in a strongly consistent store, use queues as rebuildable dispatch accelerators, and combine renewable leases with monotonically increasing fencing epochs so failures can cause retries without allowing stale owners to mutate state.
