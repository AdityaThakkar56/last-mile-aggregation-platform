# On-Demand Aggregation Platform for Last-Mile Connectivity

GitHub-ready documentation pack for the shared last-mile mobility platform.

## Read in this order
1. `docs/00-documentation-map-and-status.md`
2. `docs/01-product-requirements-prd.md`
3. `docs/02-technical-requirements-trd.md`
4. `docs/03-high-level-design.md`
5. `docs/04-low-level-design.md`
6. `docs/05-algorithms-and-matching.md`
7. `docs/06-api-contract.md`
8. `docs/07-realtime-and-background-jobs.md`
9. `docs/08-data-model-and-database-plan.md`
10. `docs/09-end-to-end-request-flows.md`
11. `docs/10-failure-handling-and-recovery.md`
12. `docs/11-security-and-privacy.md`
13. `docs/12-deployment-and-demo-runbook.md`
14. `docs/13-testing-and-quality-plan.md`
15. `docs/14-implementation-roadmap.md`
16. `docs/15-architecture-decision-records.md`
17. `docs/16-glossary.md`
18. `AGENTS.md`

## Status
This pack records the design discussions. It distinguishes approved decisions from proposed defaults and pending implementation details. Exact Prisma schema/migrations, fare tariff, max trip duration, cloud provider and final OAuth configuration remain pending.

## Project baseline
- Flutter/Dart passenger and driver app; Next.js/TypeScript admin.
- NestJS modular monolith; REST + Socket.IO.
- PostgreSQL + PostGIS + Prisma.
- Redis + BullMQ + transactional outbox.
- OSRM + OpenStreetMap-compatible maps.
- Local Mac-hosted same-Wi-Fi demo, target ₹0 infrastructure/subscription spend.
- No microservices, Docker requirement, Nx/Turborepo or CI/CD in MVP.
