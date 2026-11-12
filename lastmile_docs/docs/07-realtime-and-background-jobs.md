# Real-Time Events and Background Jobs

## Socket.IO
Use for GPS, booking/offer updates, dispatch offer notifications and ride state events. It is not durable truth; after reconnect, fetch REST state.

### Rooms
- `user:{userId}`
- `ride:{rideId}`
- `driver:{driverId}`
- `admin:monitoring`
Server authorizes room membership. Do not trust arbitrary room IDs from clients.

### Events
| Event | Direction | Purpose |
|---|---|---|
| `booking:status` | Server→Passenger | Booking state |
| `offer:new` | Server→Passenger | New offer |
| `offer:updated` | Server→Passenger | Revised offer |
| `offer:expired` | Server→Passenger | Expiry |
| `ride:confirmed` | Server→Passenger | Driver accepted |
| `driver:offer:new` | Server→Driver | Dispatch offer |
| `driver:offer:expired` | Server→Driver | Driver timeout |
| `driver:location` | Driver→Server | GPS update |
| `driver:location` | Server→Passenger | Validated broadcast |
| `ride:status` | Server→Passenger/Driver | Arrived/start/complete/cancel |

Include event ID/version, entity ID and server timestamp. Ignore stale event versions and reconcile with REST.

## GPS strategy
1. Driver emits every 3–5 seconds during active ride.
2. Gateway authenticates and checks driver assignment.
3. Validate coordinates and timestamp; reject stale/out-of-order updates.
4. Write latest location to Redis with TTL.
5. Broadcast to authorized ride room.
6. Persist PostgreSQL snapshot every 15–30 seconds plus important events.
7. On reconnect, fetch current ride and latest known location via REST.

For future multiple API instances, use Socket.IO Redis adapter.

## BullMQ queues
- `matching`: `process-zone-batch`, `rematch-booking`, `expire-booking`.
- `dispatch`: `find-drivers`, `expire-driver-offer`.
- `notifications`: `send-notification`.
- Reconciliation jobs: reclaim expired leases, inspect offers/reservations, republish outbox.

Initially one worker process may consume all queues; split only if needed.

## Transactional outbox
1. Write business change + outbox event in one PostgreSQL transaction.
2. Publisher reads pending outbox.
3. Enqueue BullMQ job.
4. Mark outbox published.
5. On Redis/publisher failure, leave pending and retry.
6. Duplicate publication is possible; processor must be idempotent.

Outbox is a durable publication mechanism, not a second competing business queue.

## Job rules
- Pass IDs, fetch current DB state.
- Use deterministic job IDs where appropriate.
- Bounded retries/exponential backoff.
- BullMQ stalled-job recovery plus DB leases/claims.
- Never rely on in-memory locks.
- Recheck cancellation/status before executing work.
