# AI Agent Instructions

Read the relevant `docs/` files before implementing features.

## Fixed product rules
- One active booking per passenger; enforce in PostgreSQL.
- Passenger selects SHARED or PRIVATE. Only SHARED enters matching.
- Never silently convert SHARED to PRIVATE; explicit consent is required.
- Initial capacity: 4 paying passengers plus driver.
- Group size 2–4; every passenger must accept the same offer version.
- Defaults: 60 s matching window, 120 s maximum shared wait from original booking creation, 20 s offer timeout, 25% maximum detour, 150 m pickup geofence (pilot values; configurable).
- Rematching does not reset original wait deadline.
- A changed group/route/fare requires a new offer and fresh acceptance from everyone.
- Group acceptance is not ride confirmation. Confirm only after driver accepts and is atomically assigned.
- Reject this group is distinct from cancel request.
- Driver requires admin approval before going online.
- Cash only in MVP; ride completion is independent from payment confirmation.
- Group cancellation: preserve remaining group only if all feasibility constraints still pass; otherwise rematch.
- Late-cancellation strike/cooldown is the agreed direction; exact thresholds/exemptions need final confirmation.
- PostgreSQL is source of truth. Redis is queue/ephemeral GPS state.
- Socket.IO is not durable truth; reconnect via REST.

## Architecture constraints
- NestJS modular monolith, not microservices.
- API and workers are separate runnable roles from same codebase; combine deployment if free hosting requires.
- REST for commands/queries, Socket.IO for live events, BullMQ for background jobs.
- PostgreSQL + PostGIS + Prisma; Redis + BullMQ; OSRM.
- Zero paid infrastructure/subscriptions. Local same-Wi-Fi demo is fallback.
- No CI/CD, Docker, Kubernetes, Nx or Turborepo in MVP unless explicitly approved.

## Engineering rules
- Backend validates DTOs, roles, ownership, fare, state transitions and geospatial constraints.
- Use short transactions, conditional updates, row locks and unique/partial indexes for races.
- Never hold DB transaction during OSRM/network calls or user wait.
- Use idempotency keys for retry-sensitive operations.
- Use transactional outbox for durable job/event publication.
- BullMQ processors must be idempotent and fetch authoritative state by ID.
- Never trust client-provided role, fare, driver state or booking state.
- Keep secrets out of Git; commit `.env.example` only.
- Use UTC timestamps, UUIDs and `/api/v1`.
- Update relevant docs and ADRs when decisions change.

## Do not invent pending decisions
Exact Prisma schema, fare tariff/rounding, max trip duration, candidate shortlist size, objective λ, cloud provider/free-tier, OAuth client configuration and final cancellation thresholds remain pending. Ask or mark assumptions.
