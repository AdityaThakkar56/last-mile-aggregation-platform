# Documentation Map and Status

## Status vocabulary
- **Approved:** explicitly accepted.
- **Agreed direction:** selected approach; implementation details remain open.
- **Proposed/default:** suggested initial value/policy, not fully locked.
- **Pending:** deferred until setup/implementation.
- **Future:** excluded from MVP.

## Decision snapshot
| Area | Decision | Status |
|---|---|---|
| Product | Shared last-mile aggregation from common pickup | Approved |
| Pickup | PDEU zone, geofence; initial 150 m radius | Concept approved; radius pilot default |
| Service area | Bounded destination polygon, PostGIS | Approved |
| Mobile/admin | Flutter + Next.js | Agreed |
| Backend | NestJS modular monolith | Approved |
| Data | PostgreSQL + PostGIS + Prisma | Approved |
| Queue | Redis + BullMQ; matching/dispatch/notifications queues | Approved |
| Reliable events | Transactional outbox | Approved |
| Real-time | Socket.IO | Approved |
| GPS | Redis latest location + periodic PostgreSQL snapshots | Approved |
| Routing | OSM data + OSRM | Agreed |
| Matching | Zone-level batches, one zone lease, booking claims | Approved |
| Grouping | Greedy candidate grouping + feasibility | Agreed |
| Route order | OSRM Table matrix + enumerate up to 24 permutations | Approved |
| Fare | Proportional solo-fare allocation | Formula approved; tariff pending |
| Dispatch | Eligible-driver filter + ETA ranking + atomic reservation | Approved |
| Concurrency | READ COMMITTED + locks + conditional updates + constraints | Approved |
| Authentication | Google OAuth identity + app JWT/refresh tokens | Agreed |
| Driver onboarding | Admin approval | Approved |
| Payment | Cash only in MVP | Approved |
| Cancellation | Preserve feasible remaining group, else rematch | Approved |
| Late cancellation | Strike/cooldown rather than money | Agreed direction; thresholds pending |
| Demo | Same Wi-Fi; Mac hosts services | Approved |
| Budget | ₹0 subscriptions/infrastructure target | Hard constraint |
| Exact schema | Deferred until DB initialization | Pending by choice |
| Cloud provider | Not selected | Pending |
| CI/CD | Excluded from MVP | Explicitly excluded |

## Change control
A change to an approved rule must explain rationale, affected modules/APIs/data/jobs/tests, add an ADR, and update all affected documents.
