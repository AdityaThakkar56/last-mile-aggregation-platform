# Failure Handling and Recovery

## Principle
Recover wherever possible; fail safely whenever correctness of booking, fare, group or assignment cannot be guaranteed.

## Failure matrix
| Failure | Immediate behavior | Recovery |
|---|---|---|
| Redis down | Booking can persist with outbox; jobs/GPS delayed | Republish outbox when Redis returns |
| OSRM down | Pause route-dependent matching; no fabricated fare/detour | Bounded retries + circuit breaker |
| PostgreSQL down | Reject writes/state transitions; no confirmation/assignment | Retry with idempotency key; reconcile |
| Driver disconnect | Keep active ride; mark GPS stale | Reconnect; admin alert if prolonged |
| API crash | DB state remains | Restart; clients reconnect and fetch REST |
| Worker crash | Job may retry/stall | BullMQ recovery + DB lease + idempotent processor |
| Duplicate request | Return same idempotent result | Unique idempotency key |
| Duplicate acceptance | Return existing response | Conditional update + unique response |
| Delayed GPS | Ignore older timestamp/sequence | Keep newest valid location |
| Driver rejects | Release reservation | Try next eligible driver |
| No driver | Do not confirm | Notify and offer private fallback with consent |
| Same batch processed twice | Inspect batch, claims and existing offers | Lease, DB constraints, idempotency |

## Matching recovery
Persist batch state PENDING/PROCESSING/COMPLETED/FAILED, lease expiry, attempt count, last error and timestamps. Reclaimed worker must inspect existing offers/claims before doing work.

## Redis outage
PostgreSQL remains source of truth. Booking/outbox can commit; matching/dispatch waits. GPS may be stale. Do not confirm assignment without durable coordination. Republish pending outbox when Redis returns.

## OSRM outage
Use timeout, bounded backoff and CLOSED/OPEN/HALF-OPEN circuit breaker. Keep batch only until original wait deadline. Do not use straight-line distance as invisible substitute. Offer fallback/cancel if routing unavailable beyond deadline.

## PostgreSQL outage
Fail closed for booking acceptance, driver assignment, trip transitions and payment updates. Never report success before commit. Reconcile outbox/leases/offers after recovery.

## Driver disconnect
Do not auto-reassign during active trip. Keep IN_PROGRESS, show stale location timestamp, reconnect and restore GPS. Admin may intervene explicitly.

## Reconciliation
Inspect expired leases, stuck batches, expired offers, driver reservations past deadline, accepted groups without dispatch, unpublished outbox events, completed rides with payment pending. Use auditable state transitions; never guess active trip state.

## Concurrency
READ COMMITTED plus row locks, conditional updates, unique constraints. Keep transactions short, lock rows consistently, bounded retries for safe transient deadlocks. No SERIALIZABLE requirement for MVP.

## Future horizontal scaling
PostgreSQL constraints coordinate instances; BullMQ workers can be distributed; Socket.IO Redis adapter enables cross-instance broadcasts; no in-memory lock/session dependency.
