# Architecture Decision Records (ADR)

| ADR | Decision | Status | Why / trade-off |
|---|---|---|---|
| 001 | NestJS modular monolith | Approved | Small team and transactional workflows; less ops than microservices, but requires module discipline |
| 002 | Flutter mobile + Next.js admin | Agreed | Mobile-first + web admin; two frontend stacks |
| 003 | PostgreSQL + PostGIS + Prisma | Approved | ACID, geospatial and type-safe access; some raw SQL needed |
| 004 | Redis + BullMQ | Approved | Delayed/retried jobs; adds Redis operations |
| 005 | Transactional outbox | Approved | Prevent lost job publication; at-least-once means idempotent consumers |
| 006 | REST + Socket.IO | Approved | Clear commands plus live updates; reconnect needs REST reconciliation |
| 007 | Redis latest GPS + DB snapshots | Approved | Reduce DB writes; latest live location can be stale during Redis outage |
| 008 | One batch per pickup zone | Approved | Avoid overlapping matching; serializes one zone |
| 009 | Greedy grouping + permutation route order | Agreed | Explainable, bounded for four dropoffs; not globally optimal |
| 010 | READ COMMITTED + locks/conditional updates/constraints | Approved | Correctness without SERIALIZABLE overhead; careful locking required |
| 011 | Google OAuth + app JWT | Agreed | Google identity, app-owned authorization; OAuth config needed |
| 012 | Admin approval for drivers | Approved | Trust and safety; manual onboarding |
| 013 | Cash-only MVP | Approved | Avoid gateway fees/refunds/settlement; manual reconciliation |
| 014 | Explicit private fallback | Approved | Preserves passenger consent; extra state/interaction |
| 015 | Preserve feasible group on cancellation | Approved | Avoid unnecessary group cancellation; requires recomputation |
| 016 | Late-cancellation strike/cooldown | Agreed direction | Avoid money/payment complexity; exact thresholds/exemptions need confirmation |
| 017 | One active booking per passenger | Approved | Prevent duplicate demand; no multiple queued destinations |
| 018 | Same-Wi-Fi local demo | Approved | Zero infrastructure cost; Mac must stay on same network |
| 019 | ₹0 infrastructure/subscription constraint | Hard constraint | Free tiers may sleep/throttle; local fallback |
| 020 | Exact DB schema deferred to DB initialization | Approved deferral | Environment setup first; schema remains implementation gate |
| 021 | No CI/CD in MVP | Explicitly excluded | Focus on core feature; manual release process |

## Detailed algorithm decision
- One zone-level batch/lease.
- Greedy candidate grouping with bounded shortlist.
- OSRM Table matrix once per candidate group; enumerate at most 24 drop-off permutations.
- Max detour initial 25%; max trip duration pending.
- Shared fare proportional to solo fare estimates; base tariff/rounding pending.
- Driver eligibility then ETA ranking; atomic reservation.
- Advanced marginal cost and global bipartite matching deferred.

## Change protocol
For any approved decision change, explain rationale, migration/API/event/test impact, then update this ADR and relevant topic docs.
