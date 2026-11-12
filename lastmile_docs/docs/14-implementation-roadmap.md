# Implementation Roadmap

## Constraints
Team 2–3; 4–6 weeks; Mac/iPhone; zero paid infrastructure/subscriptions; no CI/CD, microservices, Kubernetes or unnecessary Docker.

## Phase 0 — Environment/repository
Verify Xcode, Flutter, Node, Git, Homebrew. Create repo/app folders, `.gitignore`, `.env.example`. Initialize Flutter, Next.js, NestJS. Verify simulator and iPhone LAN connectivity. Install PostgreSQL/PostGIS and Redis after checking existing setup.
Exit: all app surfaces start locally.

## Phase 1 — DB and identity
Finalize exact schema; Prisma migrations; PostGIS; seed pickup zone/service region/admin; Google OAuth and app JWT; role and driver approval.
Exit: passenger and approved driver authenticate correctly.

## Phase 2 — Booking
Geofence/service polygon; one active booking constraint; shared/private creation; idempotency/outbox; history/cancel.
Exit: booking saved exactly once and queued.

## Phase 3 — Matching/routing
Zone batches/leases/claims; OSRM table; candidate groups; permutations; detour; fare; offer snapshot.
Exit: two feasible requests create offer.

## Phase 4 — Offers/dispatch
Passenger accept/reject/timeout; unanimous acceptance; driver eligibility/ETA; atomic reservation; driver rejection; explicit private fallback.
Exit: group obtains one driver or safely fails.

## Phase 5 — Ride/realtime
Arrival/start/complete; Socket.IO auth/rooms; GPS Redis latest + DB snapshots; live map; reconnect recovery.
Exit: tracking works across devices on same Wi-Fi.

## Phase 6 — Payment/admin/reliability
Cash confirmation; admin dashboard; cancellation policy; reconciliation; failure tests.
Exit: full flow and edge cases pass.

## Phase 7 — Demo hardening
Seed accounts/data; verify network/firewall/permissions; routing coverage; concurrency/failure tests; demo script; feature freeze.

## Team split
Three people: backend/data/matching; Flutter; admin/testing/integration. Two people: backend/data/routing/queues and Flutter+admin; share integration/testing.

## Definition of done
Server-side business rule; migration if needed; API/event docs; unit/integration tests; UI loading/error/empty/reconnect states; no secret leakage; docs/ADR updated.
