# Architecture Decision Records (ADRs) and Decision Log

## On-Demand Aggregation Platform for Last-Mile Connectivity Using Shared Micro-Mobility

---

## 1. Decision Status Classification Framework

To guarantee strict architectural integrity and prevent undocumented assumptions, every technical and business decision is classified under one of four unambiguous governance statuses:

- **APPROVED:** Explicitly accepted, verified, and locked architectural baseline. AI agents and developers must strictly adhere to these decisions without deviation.
- **PROPOSED:** Recommended initial implementation pattern or pilot default. Subject to validation during local prototype testing.
- **PENDING:** Open business or regulatory parameter awaiting explicit stakeholder confirmation, tariff modeling, or legal review. **Must not be fabricated by AI coding agents.**
- **DEFERRED:** Explicitly excluded from the MVP scope to protect the 4–6 week timeline and ₹0 budget.

---

## 2. Master Decision Classification Matrix

| Reference | Domain | Decision Summary | Governance Status | Implementation Boundary |
|---|---|---|---|---|
| **ADR-001** | Backend Architecture | NestJS Modular Monolith over Microservices | **APPROVED** | `apps/backend` |
| **ADR-002** | Process Topology | Dual Process Roles (API & Worker) from Shared Codebase | **APPROVED** | `main.ts` / `worker.main.ts` |
| **ADR-003** | Frontend Topology | Flutter Single Codebase Mobile & Next.js Admin | **APPROVED** | `apps/mobile`, `apps/admin` |
| **ADR-004** | Authoritative Storage| PostgreSQL 16+ & PostGIS 3.4+ as Single Source of Truth | **APPROVED** | `prisma/schema.prisma` |
| **ADR-005** | Cache & Queues | Redis 7.2+ for Ephemeral GPS & BullMQ Queues | **APPROVED** | `InfrastructureModule` |
| **ADR-006** | Message Reliability | Transactional Outbox Pattern for Asynchronous Jobs | **APPROVED** | `outbox_events` table |
| **ADR-007** | Routing Engine | Local OSRM on OpenStreetMap Data | **APPROVED** | `RoutingModule` (Port 5000) |
| **ADR-008** | Matching Paradigm | Removal of Passenger Group Acceptance in Favor of Upfront Consent | **APPROVED** | `BookingsModule`, `MatchingModule` |
| **ADR-009** | Group Deduplication | Removal of Dedicated Fingerprint Tables in MVP | **APPROVED** | Database Schema |
| **ADR-010** | Driver Dispatch | Sequential 1-to-1 Group Offers with Driver Multi-Offer Receipt | **APPROVED** | `DispatchModule` |
| **ADR-011** | Driver Exclusion | Scoped Driver Rejection/Timeout Exclusion per Group Version | **APPROVED** | `driver_dispatch_offers` |
| **ADR-012** | Concurrency Locking | Atomic Driver Acceptance via Row-Level Locking & Conditional CAS | **APPROVED** | `DispatchService.acceptOffer()` |
| **ADR-013** | Cancellation Engine | State-Dependent Cancellation & Feasible Group Preservation | **APPROVED** | `BookingsService.cancel()` |
| **ADR-014** | Late Cancellation Fee| ₹30 Fixed Late Cancellation Fee Assessed After Driver Assignment | **APPROVED** | `cancellation_fees` table |
| **ADR-015** | Fee Recovery | Separate Cash Ledger Recovered as Surcharge on Next Ride | **APPROVED** | `PaymentsModule` |
| **ADR-016** | Payment Mode | Cash-Only Settlement Independent of Ride Completion | **APPROVED** | `cash_payment_records` |
| **ADR-017** | GPS Telemetry | Ephemeral 3–5s Redis TTL with 15–30s PostgreSQL Snapshots | **APPROVED** | `LocationsModule`, Socket.IO |
| **ADR-018** | Booking Invariant | Exactly One Active Booking per Passenger via Partial Unique Index | **APPROVED** | PostgreSQL Database Engine |
| **ADR-019** | Matching Batching | Zone-Level Batches Protected by Distributed Redis Leases | **APPROVED** | `MatchingBatchProcessor` |
| **ADR-020** | Route Permutations | Permutation Solver ($k! \le 24$) with Max 25% Detour Filter | **APPROVED** | `PermutationSolver` |
| **ADR-021** | Cost Allocation | Proportional Solo-Fare Allocation Formula | **APPROVED** | `FaresService` |
| **ADR-022** | Infrastructure Cost | Strict ₹0 Budget & Same-Wi-Fi Mac-Hosted Demonstration | **APPROVED** | Deployment Architecture |
| **ADR-023** | DevOps Boundaries | Exclusion of CI/CD, Mandatory Docker, and Kubernetes | **APPROVED** | Development Workflow |
| **ADR-024** | DB Isolation Level | PostgreSQL `READ COMMITTED` with Explicit Row Locking | **APPROVED** | Prisma / Database Config |
| **ADR-025** | Identity & Auth | Google OAuth 2.0 with Platform JWT Access/Refresh Rotation | **APPROVED** | `AuthModule` |
| **PEND-01**| Pricing Policy | Exact Base Fare Tariff and Per-Kilometer Rate | **PENDING** | `FaresModule` |
| **PEND-02**| Pricing Policy | Exact Shared Total Fare Derivation Function | **PENDING** | `FaresModule` |
| **PEND-03**| Pricing Policy | Minimum Ride Fare Threshold | **PENDING** | `FaresModule` |
| **PEND-04**| Pricing Policy | Integer Fare Rounding & Remainder Allocation Rule | **PENDING** | `FaresModule` |
| **PEND-05**| Matching Policy | Maximum Total Multi-Stop Trip Duration Threshold | **PENDING** | `MatchingModule` |
| **PEND-06**| Matching Policy | Candidate Booking Shortlist Size for Greedy Clustering | **PENDING** | `MatchingModule` |
| **PEND-07**| Routing Policy | Advanced Detour Penalty Coefficient ($\lambda$) in Objective | **PENDING** | `MatchingModule` |
| **PEND-08**| Consumer Rights | Exact Numerical Threshold for "Material Route/Fare Change" | **PENDING** | `BookingsModule` |
| **PEND-09**| Identity Config | Final Google OAuth Client IDs and Universal Redirect URI | **PENDING** | `.env` Configuration |
| **PEND-10**| Security Policy | Final Access & Refresh Token Expiry Configuration | **PENDING** | `AuthModule` |
| **PEND-11**| Legal & Compliance | Applicable Cancellation Fee Consumer Protection Review | **PENDING** | Legal Review |
| **PEND-12**| Cloud Hosting | Production Cloud Provider Selection (Post-MVP) | **PENDING** | Infrastructure |

---

## 3. Detailed Architecture Decision Records (ADR Log)

### ADR-001: Modular Monolithic Architecture with NestJS
- **Context:** The development team consists of 2–3 CSE students building an on-demand shared micro-mobility platform in 4–6 weeks.
- **Problem:** Microservices introduce distributed transactions, service discovery, distributed tracing, and complex deployments that jeopardize delivery within the timeline and ₹0 budget.
- **Decision:** Architect the backend as a cohesive **Modular Monolith** using **NestJS 10+** and **TypeScript 5+**.
- **Rationale:** NestJS provides enterprise-grade dependency injection, strict module boundaries, and architectural structure while allowing transactional ACID consistency within a single PostgreSQL database.
- **Alternatives Considered & Rejected:**
  - *Microservices:* Rejected due to distributed transaction overhead, network latency, and operational complexity.
  - *Unstructured Express.js:* Rejected due to lack of architectural guardrails and high risk of spaghetti code.
- **Trade-offs:** Modules share runtime memory and database connection pools; developers must maintain strict discipline and never bypass module boundaries.
- **Status:** **APPROVED**

---

### ADR-002: Dual Process Roles (API & Worker) from Shared Codebase
- **Context:** The system must handle high-throughput synchronous REST requests, persistent WebSockets, and compute-heavy batch matching algorithms.
- **Problem:** Running OSRM permutation calculations and 20s dispatch timers in the HTTP event loop blocks API requests and causes socket latency spikes.
- **Decision:** Deploy two separate logical processes from the same NestJS codebase:
  1. `api`: Exposes REST endpoints and Socket.IO gateways.
  2. `worker`: Consumes BullMQ queues (matching, dispatch, notifications) and runs schedulers.
- **Rationale:** Decouples user-facing latency from heavy background compute while sharing data models, DTOs, and utility libraries.
- **Trade-offs:** Requires running two Node processes locally or in production. (Fallback: can execute within a unified single process on constrained hardware).
- **Status:** **APPROVED**

---

### ADR-003: Flutter Multi-Role Client & Next.js Admin Architecture
- **Context:** Mobile users require an iOS-first responsive app, while campus facility officers require a desktop administration panel.
- **Problem:** Building separate native iOS and Android apps exceeds team capacity; building mobile-responsive web apps results in poor GPS and background performance.
- **Decision:** Implement a single **Flutter / Dart** cross-platform codebase supporting both Passenger and Driver role flows. Build the administration portal in **Next.js 14+ (App Router)** with **TypeScript** and **Tailwind CSS**.
- **Rationale:** Flutter provides native 60fps performance, battery-efficient location tracking via `geolocator`, and reliable WebSockets. Next.js delivers rapid, responsive dashboard development.
- **Trade-offs:** Maintaining two frontend codebases (Dart for mobile, TypeScript for admin web).
- **Status:** **APPROVED**

---

### ADR-004: PostgreSQL 16+ & PostGIS 3.4+ as Authoritative ACID Source of Truth
- **Context:** The platform manages complex geospatial geofencing, concurrent driver assignments, and financial cash ledgers.
- **Problem:** Non-relational databases (MongoDB) lack mature multi-record ACID transactions and spatial indexing needed for high-concurrency mobility.
- **Decision:** Standardize on **PostgreSQL 16+** with the **PostGIS 3.4+** spatial extension as the sole authoritative persistence store. Use Prisma ORM for standard models and controlled raw SQL for PostGIS functions and pessimistic row locking.
- **Rationale:** Uncompromised ACID durability, native spatial containment functions (`ST_DWithin`, `ST_Contains`), and partial unique indexes prevent illegal states at the database engine level.
- **Trade-offs:** Requires learning PostGIS SQL functions and managing manual raw SQL queries for complex spatial math.
- **Status:** **APPROVED**

---

### ADR-005: Redis 7.2+ for Ephemeral GPS Caching and BullMQ Job Coordination
- **Context:** Drivers emit GPS coordinates every 3–5 seconds; matching batches and dispatch timeouts require distributed delayed scheduling.
- **Problem:** Writing high-frequency GPS fixes to PostgreSQL exhausts database write-ahead log (WAL) bandwidth and disk I/O.
- **Decision:** Use **Redis 7.2+** strictly for ephemeral in-memory data (storing latest driver GPS with 30s TTL) and as the backing broker for **BullMQ** job queues.
- **Rationale:** In-memory speed prevents database write saturation; BullMQ provides rock-solid delayed job scheduling and retries.
- **Trade-offs:** Introduces Redis as an operational dependency; PostgreSQL must remain fully authoritative if Redis restarts.
- **Status:** **APPROVED**

---

### ADR-006: Transactional Outbox Pattern for Asynchronous Message Dispatch
- **Context:** State changes in PostgreSQL (booking creation, group formation) must reliably trigger BullMQ background jobs and Socket.IO events.
- **Problem:** Writing to PostgreSQL and Redis sequentially creates dual-write failure vulnerabilities (if Redis fails, the job is lost).
- **Decision:** Implement the **Transactional Outbox Pattern**. All domain state changes insert an `outbox_events` record within the primary ACID transaction. A dedicated background worker reads and enqueues events into BullMQ using `SELECT ... FOR UPDATE SKIP LOCKED`.
- **Rationale:** Guarantees at-least-once job publication even across server crashes and network partitions.
- **Trade-offs:** Requires job consumers to be strictly idempotent to handle potential duplicate enqueueing.
- **Status:** **APPROVED**

---

### ADR-007: OpenStreetMap & Local OSRM Engine for Road-Network Routing
- **Context:** Routing, detour evaluation, and travel time matrices must accurately reflect road geometry rather than straight-line distance.
- **Problem:** Commercial APIs (Google Maps Directions / Distance Matrix) are prohibitively expensive and require billing credit cards, violating the ₹0 budget.
- **Decision:** Host a local instance of **OSRM (Open Source Routing Machine)** utilizing an OpenStreetMap regional extract covering Gandhinagar and PDEU.
- **Rationale:** Unlimited free API queries, sub-millisecond table matrix calculations, turn-by-turn polyline geometry, and zero subscription costs.
- **Trade-offs:** Requires pre-processing OSM routing data files locally; does not incorporate real-time dynamic traffic congestion.
- **Status:** **APPROVED**

---

### ADR-008: Complete Removal of Passenger Group Acceptance Round in Favor of Upfront Consent
- **Context:** Earlier architecture required unanimous post-matching passenger voting to accept a proposed group, route, and fare.
- **Problem:** Passenger acceptance voting created severe combinatorial failure cascades: 1 hesitating passenger broke the entire group, exhausted the 120s wait deadline, and left drivers waiting idle.
- **Decision:** **Permanently eliminate the passenger group acceptance round.** Passengers explicitly consent to automated grouping, capacity limits ($\le 4$), maximum detour ($\le 25\%$), and proportional fare allocation upfront at the time of booking. Formed groups proceed immediately to driver dispatch.
- **Rationale:** Dramatically accelerates dispatch speed, eliminates fragile multi-party voting, and drastically simplifies the state machine.
- **Trade-offs:** Requires strict adherence to disclosed detour and fare caps to prevent material changes without commuter consent.
- **Status:** **APPROVED**

---

### ADR-009: Removal of Dedicated Group Fingerprint Exclusion Table in MVP
- **Context:** Earlier designs included a `matching_group_attempts` table to store SHA-256 group member fingerprints (`hash(P1+P2+P3)`) to prevent re-offering rejected groups.
- **Problem:** With the removal of passenger acceptance voting (ADR-008), passengers no longer reject formed groups, rendering fingerprint tracking redundant.
- **Decision:** **Do not implement a dedicated group fingerprint or attempt exclusion table in the MVP database.** Retain ordinary operational logging (`matching_batches`, `batch_bookings`, `ride_groups`, `driver_dispatch_offers`).
- **Rationale:** Eliminates unnecessary table writes, index maintenance, and database complexity.
- **Trade-offs:** None for the MVP; fingerprinting can be re-evaluated if real-time group renegotiation is added in future versions.
- **Status:** **APPROVED**

---

### ADR-010: Sequential 1-to-1 Group Dispatch with Multi-Offer Driver Capability
- **Context:** Formed ride groups must be assigned to an available, verified micro-mobility driver.
- **Problem:** Broadcast dispatching to all drivers creates race conditions, phantom rides, and driver frustration.
- **Decision:**
  1. A formed group is offered to **EXACTLY ONE driver at a time** with a 20-second countdown timer.
  2. A driver **MAY RECEIVE MULTIPLE PENDING OFFERS** for different groups if they are the closest candidate for both.
- **Rationale:** Ensures drivers have choice while guaranteeing that groups are never disputed across multiple accepted drivers.
- **Trade-offs:** Requires an atomic reservation transaction to immediately withdraw other pending offers when a driver accepts one group.
- **Status:** **APPROVED**

---

### ADR-011: Scoped Driver Rejection/Timeout Exclusion Scoped to Group Version
- **Context:** When a driver declines or fails to respond to an offer within 20 seconds, the group must cascade to the next eligible driver.
- **Problem:** Permanent blacklists prevent drivers from being re-considered if a group reconfigures; lack of exclusion causes repeated offers to the same unwilling driver.
- **Decision:** Exclude the rejecting or timed-out driver **strictly for the current group version and dispatch attempt**. If the group membership changes (e.g., a member cancels), the group version increments, and the driver becomes eligible again.
- **Rationale:** Prevents annoying offer loops while maximizing driver utilization when ride parameters change.
- **Trade-offs:** Requires tracking `group_version` on `ride_groups` and scoping unique indexes to `(ride_group_id, group_version, dispatch_attempt_id, driver_id)`.
- **Status:** **APPROVED**

---

### ADR-012: Atomic Driver Assignment via Row-Level Locking and Conditional Updates
- **Context:** When a driver accepts a dispatch offer, multiple race conditions can occur (offer timed out, another driver accepted, or the driver accepted two offers simultaneously).
- **Problem:** Application-level validation is vulnerable to race conditions under concurrent execution.
- **Decision:** Execute driver acceptance inside a strict PostgreSQL ACID transaction using **`SELECT ... FOR UPDATE`** on `ride_groups` and a **conditional atomic update (`UPDATE driver_profiles SET operational_status = 'RESERVED' WHERE operational_status = 'AVAILABLE'`)**.
- **Rationale:** Guarantees that exactly one driver wins the assignment race without phantom bookings or double-booking drivers.
- **Trade-offs:** Requires wrapping acceptance logic in raw SQL transactions within Prisma.
- **Status:** **APPROVED**

---

### ADR-013: Passenger Cancellation Workflow with Feasible Group Preservation
- **Context:** Passengers may cancel ride requests at various stages of the booking lifecycle.
- **Problem:** Cancelling an entire 4-passenger group because 1 passenger cancels frustrates the compliant passengers.
- **Decision:**
  - If cancellation occurs *before* driver assignment: Cancel passenger booking with ₹0 fee. Recalculate remaining members. If $\ge 2$ passengers and detour $\le 25\%$, preserve group and continue dispatch. If invalid, dissolve group and return compliant members to matching with **original booking timestamps preserved**.
  - If cancellation occurs *after* driver assignment: Assess a ₹30 late fee, recalculate route for the driver, and update navigation stops.
- **Rationale:** Maximizes trip completion rates and driver earnings while treating compliant passengers fairly.
- **Trade-offs:** Requires dynamic on-the-fly route re-optimization and fare re-allocation upon cancellation.
- **Status:** **APPROVED**

---

### ADR-014: ₹30 Fixed Late Cancellation Fee Assessed After Driver Assignment
- **Context:** Passengers cancelling after a driver has accepted and committed time, fuel, and vehicle capacity inflict economic harm on the driver.
- **Problem:** Zero-penalty cancellation encourages irresponsible booking; excessive penalties discourage platform adoption.
- **Decision:** Implement a **fixed ₹30 late-cancellation fee** assessed whenever a passenger cancels after a driver has been assigned. Exempt platform errors, driver delays $> 5$ minutes, and administrative waivers.
- **Rationale:** ₹30 aligns with student budgets (50% of an average solo fare) while compensating drivers and enforcing accountability.
- **Trade-offs:** Requires maintaining a dedicated financial audit ledger and handling exemption dispute workflows.
- **Status:** **APPROVED**

---

### ADR-015: Separate Cash Ledger Recovered as Surcharge on Next Ride
- **Context:** The MVP operates without digital payment gateways; cancellation fees cannot be auto-debited from credit cards.
- **Problem:** Uncollected cancellation fees must be recovered reliably in cash without causing transaction reconciliation deadlocks.
- **Decision:** Maintain an authoritative `cancellation_fees` table (`OUTSTANDING` $\rightarrow$ `COLLECTED`). Surcharge outstanding fees on the passenger's subsequent ride. Show the breakdown upfront before booking confirmation. Mark fee collected atomically when the driver confirms cash on the subsequent ride.
- **Rationale:** Fully compatible with a 100% cash-based transport ecosystem while eliminating bad debt.
- **Trade-offs:** If a student never uses the platform again, the ₹30 fee remains uncollected.
- **Status:** **APPROVED**

---

### ADR-016: Strict Cash-Only Payment Workflow Independent of Trip Completion
- **Context:** Physical cash is handed to drivers upon drop-off.
- **Problem:** Tying vehicle trip completion to payment confirmation prevents drivers from ending the ride if payment takes extra time, and complicates multi-passenger drop-offs.
- **Decision:** Decouple ride completion (`POST /rides/:id/complete`) from cash payment confirmation (`POST /payments/:id/confirm-cash`). The driver marks the ride completed when passengers disembark, and subsequently confirms cash receipt per passenger.
- **Rationale:** Reflects physical last-mile operational reality and allows auditable tracking of payment disputes.
- **Trade-offs:** Requires two distinct state transitions in the driver app.
- **Status:** **APPROVED**

---

### ADR-017: Ephemeral GPS Telemetry (3–5s Redis TTL) with Periodic PostgreSQL Snapshots (15–30s)
- **Context:** Real-time tracking requires smooth vehicle movement on passenger maps without crippling database I/O.
- **Problem:** Writing 20 updates per minute per active vehicle into PostgreSQL creates disk bottlenecks.
- **Decision:** Ingest driver GPS fixes over Socket.IO every 3–5 seconds. Cache latest fix in Redis with a 30-second TTL and broadcast directly to the ride's socket room. Persist durable breadcrumb snapshots into `ride_locations` in PostgreSQL every **15 to 30 seconds** or upon passenger drop-off.
- **Rationale:** Delivers buttery-smooth 60fps tracking to mobile clients while reducing database write loads by 80%.
- **Trade-offs:** If Redis crashes, fine-grained 3-second breadcrumbs from the last 15 seconds are lost; macro-level trip audit trails in PostgreSQL remain intact.
- **Status:** **APPROVED**

---

### ADR-018: Strict Single Active Booking Invariant per Passenger via Partial Unique Index
- **Context:** Commuters must not flood the matching engine by creating concurrent duplicate requests.
- **Problem:** Application-level checks (`if (await findActive())`) are subject to race conditions under rapid double-tapping.
- **Decision:** Enforce the invariant at the PostgreSQL engine level using a **partial unique index**:
  ```sql
  CREATE UNIQUE INDEX idx_unique_active_passenger_booking 
  ON bookings (passenger_id) 
  WHERE status IN ('REQUESTED', 'MATCHING', 'DISPATCHING', 'CONFIRMED', 'IN_PROGRESS');
  ```
- **Rationale:** Mathematically guarantees that a passenger can never possess more than one active booking regardless of network retries or client concurrency.
- **Trade-offs:** Requires database migration and careful status transition management.
- **Status:** **APPROVED**

---

### ADR-019: Zone-Level Batch Matching with Distributed Zone Lease
- **Context:** Bookings accumulating at the PDEU pickup hub must be processed periodically in batches.
- **Problem:** Multiple worker instances executing matching simultaneously across the same passenger pool create duplicate groups and race conditions.
- **Decision:** Process matching in discrete zone batches protected by an atomic distributed Redis lock (`lease:matching:pdeu`, TTL 30s). Capture an authoritative cutoff timestamp $T_{\text{cutoff}}$ at the start of each run.
- **Rationale:** Serializes matching execution cleanly, eliminates overlapping clusters, and guarantees deterministic route optimization.
- **Trade-offs:** Adds a maximum 60-second batch accumulation latency before grouping occurs.
- **Status:** **APPROVED**

---

### ADR-020: Greedy Candidate Grouping with Permutation-Based Drop-Off Optimization ($k! \le 24$)
- **Context:** Finding optimal vehicle routes for multi-passenger carpools is an NP-hard Vehicle Routing Problem (VRP).
- **Problem:** Full global VRP solvers are overly complex for an MVP with small vehicle capacities ($k \le 4$).
- **Decision:** Use a greedy spatial heuristic to select candidate clusters of 2–4 passengers from the PDEU hub. Fetch a single travel-time matrix via OSRM Table API and evaluate all $k!$ drop-off permutations locally in-memory ($4! = 24$ maximum evaluations). Discard permutations with individual detour $> 25\%$; pick the sequence minimizing total vehicle travel time.
- **Rationale:** Extremely fast (sub-millisecond evaluation), easily explainable to university evaluators, and guaranteed to find the true road-network optimum for up to 4 stops.
- **Trade-offs:** Does not guarantee global system-wide Pareto optimality across dozens of simultaneous bookings.
- **Status:** **APPROVED**

---

### ADR-021: Proportional Solo-Fare Allocation Formula
- **Context:** The total shared trip fare must be split fairly among passengers travelling different distances along the corridor.
- **Problem:** Equal splitting (splitting evenly by $N$) penalizes commuters disembarking early; distance-only splitting ignores detour delays.
- **Decision:** Allocate costs proportionally based on benchmark solo fares:
  $$F_i = F_{\text{shared\_total}} \times \frac{\text{solo\_fare}_i}{\sum_{j=1}^{k} \text{solo\_fare}_j}$$
- **Rationale:** Mathematically sound, transparent to passengers, ensures everyone saves 25%–40% compared to a solo ride, and guarantees that total collected fares equal the driver's negotiated earnings.
- **Trade-offs:** Requires calculating benchmark solo fares for every passenger before grouping.
- **Status:** **APPROVED**

---

### ADR-022: Strict ₹0 Infrastructure Budget & Local Same-Wi-Fi Demonstration Architecture
- **Context:** The university minor project operates under a hard financial constraint of ₹0.00.
- **Problem:** Cloud hosting (AWS, GCP, Render) incurs subscription fees, credit card authorizations, or sleep-mode throttles that ruin live evaluator demos.
- **Decision:** Run all backend services (PostgreSQL, PostGIS, Redis, OSRM, NestJS API/Worker, Next.js Admin) locally on an Apple Silicon Mac host. Connect physical mobile client devices (iPhones/Android) over a shared local Wi-Fi router using the Mac's LAN IP address (`http://192.168.1.50:3000`).
- **Rationale:** 100% free, zero cloud dependencies, ultra-low latency, and resilient to campus internet outages (except Google OAuth login).
- **Trade-offs:** Requires configuring macOS firewall rules and binding servers to `0.0.0.0`.
- **Status:** **APPROVED**

---

### ADR-023: Exclusion of CI/CD, Mandatory Docker, and Cloud Microservices from MVP
- **Context:** Project timeline is strictly 4–6 weeks for 2–3 students.
- **Problem:** Configuring GitHub Actions CI/CD pipelines, Docker container orchestration, and multi-service networking consumes 30%+ of project time without adding user-facing functional value.
- **Decision:** Run native binaries locally via Homebrew and npm scripts (`brew services start postgresql@16 redis`, `npm run start:dev`). Exclude CI/CD pipelines, mandatory Dockerfiles, and Kubernetes from the MVP deliverable.
- **Rationale:** Maximizes developer velocity and focuses 100% of team effort on algorithmic correctness, data consistency, and user experience.
- **Trade-offs:** Manual setup required for new development machines (mitigated by detailed runbooks).
- **Status:** **APPROVED**

---

### ADR-024: READ COMMITTED Isolation Level with Selective Row Locking over SERIALIZABLE
- **Context:** High-concurrency operations (driver acceptance, batch claiming) require database transaction isolation.
- **Problem:** PostgreSQL `SERIALIZABLE` isolation throws unpredictable serialization aborts (`40001 serialization_failure`), requiring complex retry loops that degrade real-time dispatch performance.
- **Decision:** Standardize on PostgreSQL default **`READ COMMITTED`** isolation combined with **explicit pessimistic row-level locking (`SELECT ... FOR UPDATE`)**, atomic conditional updates, and partial unique indexes.
- **Rationale:** Delivers complete transactional safety and race-condition prevention without retry storms or lock escalation deadlocks.
- **Trade-offs:** Developers must explicitly declare `FOR UPDATE` on critical rows rather than relying on automatic serializable checking.
- **Status:** **APPROVED**

---

### ADR-025: Google OAuth Identity Verification with Platform JWT Session Rotation
- **Context:** Commuters and drivers authenticate via Google on mobile clients.
- **Problem:** Relying on client-asserted email or identity allows trivial spoofing.
- **Decision:** Flutter mobile sends the Google ID token to `/api/v1/auth/google`. NestJS cryptographically validates the token against Google's public JWKS certificates. The backend maps Google's immutable `sub` identifier to an internal `users` UUID and issues short-lived platform JWT access tokens (15m) and cryptographically random refresh tokens (7d).
- **Rationale:** Enterprise-grade security, zero password management overhead, and immediate session revocation capability.
- **Trade-offs:** Requires an active internet connection for initial Google authentication during demos.
- **Status:** **APPROVED**

---

## 4. Pending Architectural Decisions Log

The following parameters are deliberately classified as **PENDING**. AI coding agents and developers must treat them as configuration-driven and must **NEVER hardcode assumed values**:

1. **PEND-01 (Exact Base Fare Tariff):** Base boarding charge and per-kilometer rate for Gandhinagar micro-mobility corridor. *(Proposed default: ₹30 base + ₹12/km)*.
2. **PEND-02 (Shared Total Fare Derivation):** Mathematical curve deriving $F_{\text{shared\_total}}$ from route distance and passenger count. *(Proposed default: $F_{\text{shared\_total}} = \text{base} + (\text{rate} \times \text{distance}) \times 1.25$)*.
3. **PEND-03 (Minimum Ride Fare Threshold):** Minimum absolute fare floor below which no passenger can be charged. *(Proposed default: ₹25.00)*.
4. **PEND-04 (Fare Rounding and Remainder Allocation):** Exact rule for rounding fractional paise and assigning remainder pennies to the longest or shortest trip. *(Proposed default: round to nearest integer ₹, assign remainder to highest solo fare)*.
5. **PEND-05 (Maximum Total Multi-Stop Trip Duration):** Absolute upper bound on journey length to prevent excessive multi-stop journeys. *(Proposed default: 45 minutes)*.
6. **PEND-06 (Candidate Booking Shortlist Size):** Maximum spatial cluster size loaded by greedy grouping algorithm. *(Proposed default: 8 candidate bookings)*.
7. **PEND-07 (Advanced Matching Penalty Coefficient $\lambda$):** Detour penalty weighting factor in future multi-objective optimization function. *(Status: PROPOSED / POST-MVP)*.
8. **PEND-08 (Material Route/Fare Change Threshold):** Exact percentage or rupee increase triggering a mandatory passenger cancellation exemption without penalty. *(Proposed default: $> 15\%$ fare increase or $> 10$ minute delay)*.
9. **PEND-09 (Google OAuth Client Configuration):** Production iOS Bundle ID, Android SHA-1 fingerprint, and web OAuth client IDs. *(Deferred to client configuration)*.
10. **PEND-10 (Token Expiry Configuration):** Exact token lifespans. *(Pilot defaults: 15 minutes access, 7 days refresh)*.
11. **PEND-11 (Cancellation Fee Legal Review):** Compliance review of ₹30 late cancellation fee policy under Indian consumer protection regulations. *(Requires university facility / legal review)*.
12. **PEND-12 (Production Cloud Provider):** Selection of free-tier or university-sponsored cloud hosting provider (Fly.io, Render, AWS Educate) for post-demo deployment. *(Deferred until local evaluation is complete)*.

---

*End of Architecture Decision Records (ADRs) and Decision Log*

