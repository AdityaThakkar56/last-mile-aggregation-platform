# Technical Requirements Document (TRD)

## Technology baseline
| Layer | Technology | Rationale |
|---|---|---|
| Mobile | Flutter/Dart | Single mobile codebase, iOS-first |
| iOS toolchain | Xcode | Required for simulator, build, signing and device deployment |
| Admin | Next.js/TypeScript | Web administration |
| Backend | NestJS/TypeScript | Modular structure, guards, gateways, worker entry points |
| API | REST | Commands/queries and Swagger |
| Live | Socket.IO | GPS and ride/offer push |
| Database | PostgreSQL | ACID source of truth |
| Geospatial | PostGIS | Geofence and service polygon |
| ORM | Prisma | Type-safe access and migrations |
| Queue | Redis + BullMQ | Background processing, delayed jobs and retries |
| Routing | OSRM + OSM data | Road matrix and route geometry |
| Map UI | flutter_map / Leaflet-compatible tiles | Avoid mandatory paid map SDK |
| Flutter libs | Dio, Provider, go_router, flutter_secure_storage, geolocator, socket_io_client, google_sign_in | Networking, state, navigation, secure storage, GPS, sockets, OAuth |

## Architecture constraints
- Modular monolith, not microservices.
- API and worker are separate runnable entry points from same NestJS codebase.
- Local development runs separate processes.
- If free hosting cannot sustain separate services, combine roles in one deployment without changing logical separation.
- No Docker requirement, Nx/Turborepo, Kubernetes or CI/CD in MVP.
- Same-Wi-Fi local demo is initial deployment target.
- Verify any future free hosting terms; do not assume free always-on services.

## Non-functional requirements
- **Correctness:** no duplicate active booking/assignment; no ride confirmation before driver acceptance; no silent private fallback.
- **Reliability:** outbox recovery; idempotent jobs; client REST reconciliation; explicit failure states.
- **Performance:** short transactions; one OSRM table matrix per candidate group rather than one request per permutation; at most 24 orders for four dropoffs.
- **GPS:** 3–5 s updates, 15–30 s snapshots (initial defaults).
- **Security:** server-side DTO validation, RBAC, resource ownership, secret hygiene.
- **Cost:** ₹0 target; local open-source stack; public OSM/OSRM endpoints are not assumed production infrastructure.

## Environment setup
Verify before installing: Xcode, command-line tools, Homebrew, Flutter/Dart, VS Code, Node LTS/npm, Git. Later install PostgreSQL+PostGIS and Redis. OSRM setup after basic routing integration. Prefer native Apple Silicon ARM64. Keep secrets in ignored `.env`; avoid unnecessary global PATH/system Python changes.

## Pending
Exact versions, exact Prisma schema, OSRM region/data import, tile source, cloud provider, OAuth client IDs/redirect scheme, token expiry values.
