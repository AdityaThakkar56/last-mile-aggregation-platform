# Architecture Decision Records (ADRs) and Decision Register

## On-Demand Aggregation Platform for Last-Mile Connectivity Using Shared Micro-Mobility

---

## 1. Decision Status Classification Framework

Every architectural, technical, and business decision is classified under one of four unambiguous governance statuses:
- **APPROVED:** Explicitly agreed, verified, and locked architectural baseline. AI agents and developers must strictly adhere to these decisions without deviation.
- **PROPOSED:** Recommended implementation pattern subject to empirical validation during local testing.
- **PENDING SPECIFICATION / OPEN DECISION:** Open business, regulatory, or client-configuration parameter awaiting external deployment confirmation. Must **never** be fabricated by AI coding agents.
- **DEFERRED:** Explicitly excluded from the MVP scope to protect the 4–6 week timeline and ₹0 budget.

---

## 2. Master Decision Classification Matrix

| Reference | Domain | Decision Summary | Governance Status | Implementation Boundary |
|---|---|---|---|---|
| **ADR-001** | Backend Architecture | NestJS Modular Monolith over Microservices | **APPROVED** | `apps/backend` |
| **ADR-002** | Mobile Framework | Single Flutter Codebase for Both Android and iOS | **APPROVED** | `apps/mobile` |
| **ADR-003** | Development Priority | iOS Simulator as Primary Initial Development Environment | **APPROVED** | Development Workflow |
| **ADR-004** | Backend Runtime | NestJS 10+ with Dual Process Entrypoints (API & Worker) | **APPROVED** | `main.ts` / `worker.main.ts` |
| **ADR-005** | Authoritative Storage| PostgreSQL 16+ & PostGIS 3.4+ as Sole Source of Truth | **APPROVED** | `prisma/schema.prisma` |
| **ADR-006** | Cache & Queues | Redis 7.2+ Strictly for Ephemeral GPS & BullMQ Queues | **APPROVED** | `InfrastructureModule` |
| **ADR-007** | Routing Engine | Local OSRM on OpenStreetMap Data (No Custom Graph/A*) | **APPROVED** | `RoutingModule` (Port 5000) |
| **ADR-008** | Group Optimization | Greedy Clustering + Exhaustive Permutations ($k! \le 24$) | **APPROVED** | `MatchingModule` |
| **ADR-009** | Vehicle Capacity | Shared Group Capacity 2 to 4 Passengers with Preferences | **APPROVED** | `BookingsModule` |
| **ADR-010** | Matching Deadlines | 60s Batch Interval with 120s Max Waiting Deadline | **APPROVED** | `MatchingModule` |
| **ADR-011** | Cost Allocation | Proportional Solo-Fare Allocation Formula | **APPROVED** | `FaresModule` |
| **ADR-012** | Fare Ceiling | Private Estimated Fare as Shared Fare Ceiling | **APPROVED** | `FaresModule` |
| **ADR-013** | Quote Validity | Exactly 2-Minute Fare Quote Expiry Window | **APPROVED** | `fare_quotes` table |
| **ADR-014** | Waiting Charges | First 5m Free, ₹5/min with Ceiling Rounding Schedule | **APPROVED** | `TripsModule` |
| **ADR-015** | Late Penalties | Fixed ₹30 Late Cancellation and No-Show Fees | **APPROVED** | `cancellation_fees` table |
| **ADR-016** | No-Show Policy | Flat ₹30 Fee with Zero Additional Waiting Charge | **APPROVED** | `TripsModule` |
| **ADR-017** | Driver Dispatch | Sequential 1-to-1 Offers with 20s Countdown Timers | **APPROVED** | `DispatchModule` |
| **ADR-018** | Multi-Offer Receipt | Drivers May Receive Multiple Offers for Different Groups | **APPROVED** | `DispatchModule` |
| **ADR-019** | Asynchronous Pub/Sub| Transactional Outbox Pattern for Guaranteed Jobs | **APPROVED** | `outbox_events` table |
| **ADR-020** | Data Authority | PostgreSQL is Authoritative; Redis Flushing Loses No Data | **APPROVED** | Persistence Architecture |
| **ADR-021** | Interface Roles | REST for Authoritative Mutations; Sockets for Notification | **APPROVED** | API / Gateway Architecture |
| **ADR-022** | Breakdown Policy | Mid-Trip Incident Logging and Replacement Dispatch | **APPROVED** | `IncidentsModule` |
| **ADR-023** | Resumption Discount | 10% Fare Discount for Onboard Passengers on Resumption | **APPROVED** | `FaresModule` |
| **ADR-024** | Breakdown Abortion | Onboard Passengers Pay ₹0 if Replacement Unavailable | **APPROVED** | `IncidentsModule` |
| **ADR-025** | Early Drop-Off | On-Route Drop-Off Permitted with Original Fare Unchanged | **APPROVED** | `TripsModule` |
| **ADR-026** | Settlement Model | Cash-Only Settlement Independent of Ride Completion | **APPROVED** | `cash_collections` table |
| **ADR-027** | Demonstration Host | Apple Silicon Mac Host for ₹0 Local Demo Environment | **APPROVED** | Deployment Architecture |
| **ADR-028** | Simulation Engine | Dedicated Demo Control Center & Virtual Agent Runners | **APPROVED** | `DEMO_SIMULATION_ARCHITECTURE.md` |
| **ADR-029** | Fingerprint Removal | Permanent Removal of Group Fingerprint Table in MVP | **APPROVED** | Database Schema |
| **ADR-030** | Private Conversion | No Automatic Private Conversion; Explicit Consent Only | **APPROVED** | `BookingsModule` |
| **ADR-031** | Route Alteration | No Mid-Trip Shared Destination Alterations in MVP | **APPROVED** | `TripsModule` |
| **OPEN-01** | Production OAuth | Production Google Client IDs and Web Redirect URIs | **OPEN DECISION**| `.env` Production Config |
| **OPEN-02** | Legal Review | Commercial Review of ₹30 Late Fee under Indian Consumer Law | **OPEN DECISION**| Post-MVP Legal Review |

---

## 3. Comprehensive Architecture Decision Records (ADRs)

### ADR-001: NestJS Modular Monolith over Microservices
- **Category:** Backend Architecture
- **Problem:** The team consists of 2–3 students building an aggregation platform in 4–6 weeks. Microservices introduce network latency, distributed transaction deadlocks (Saga pattern), gRPC orchestration, and operational complexity that would jeopardize project delivery.
- **Alternatives Considered:**
  1. *Microservices (NestJS + gRPC/Kafka):* Rejected due to distributed state complexity, multi-repo overhead, and deployment friction.
  2. *Unstructured Express.js:* Rejected due to lack of architectural guardrails and high risk of spaghetti code.
- **Decision:** Architect the backend as a strictly cohesive **Modular Monolith** using **NestJS 10+** and **TypeScript 5+**.
- **Reasoning:** NestJS provides enterprise-grade dependency injection, clear module boundaries, and synchronous in-process service calls while guaranteeing strict ACID database transactions in a single PostgreSQL instance.
- **Trade-offs:** Domain modules share runtime memory and database connection pools. Developers must strictly respect module boundaries without circular dependencies.
- **Status:** **APPROVED**

---

### ADR-002: Single Flutter Codebase for Android and iOS
- **Category:** Frontend Mobile Architecture
- **Problem:** Commuters and drivers operate both Android and iOS devices. Developing separate native Swift and Kotlin applications exceeds team capacity. Mobile web apps suffer from poor background GPS tracking.
- **Alternatives Considered:**
  1. *Separate Native Apps (Swift + Kotlin):* Rejected; doubles frontend implementation workload.
  2. *React Native:* Considered, but Flutter provides superior rendering consistency via Impeller and robust background GPS plugins.
- **Decision:** Build a single cross-platform mobile codebase using **Flutter 3.22+ and Dart 3.4+**, supporting both Android and iOS as first-class official project targets.
- **Reasoning:** Delivers 60fps native performance, battery-efficient background GPS via `geolocator`, and reliable WebSocket streaming.
- **Trade-offs:** Requires maintaining platform-specific configurations (Info.plist for iOS, AndroidManifest.xml for Android).
- **Status:** **APPROVED**

---

### ADR-003: iOS Simulator as Initial Primary Development Environment
- **Category:** Development Workflow
- **Problem:** Setting up, configuring, and testing both mobile toolchains (Xcode + Android Studio) simultaneously on day 1 creates tooling distraction and slows backend integration.
- **Alternatives Considered:**
  1. *Android Studio / Emulator First:* Rejected; team's primary workstation is Apple Silicon Mac where iOS Simulator boots instantly with minimal memory footprint.
  2. *Both Simultaneously:* Slows initial sprint velocity.
- **Decision:** Prioritize **VS Code / Antigravity IDE with Xcode and the iOS Simulator** as the primary initial development and demonstration environment, alongside physical iPhone validation. Android Studio and Emulator testing will follow in Phase 2.
- **Reasoning:** Xcode runs with near-zero latency on Apple Silicon, provides superior hardware simulation, and accelerates rapid UI/API iteration. Both platforms remain equal official targets.
- **Trade-offs:** Android-specific device permission testing is sequenced into the secondary verification phase.
- **Status:** **APPROVED**

---

### ADR-004: Dual Process Roles (API & Worker) from Shared NestJS Codebase
- **Category:** Backend Runtime Topology
- **Problem:** Running compute-intensive OSRM permutation calculations and 20s dispatch countdown timers in the main HTTP event loop blocks incoming REST requests and introduces WebSocket latency spikes.
- **Alternatives Considered:**
  1. *Single Node.js Process:* Vulnerable to event loop starvation during heavy batch calculations.
  2. *Separate Repositories:* Creates duplicate DTOs, entity definitions, and database schemas.
- **Decision:** Deploy two separate logical process roles from the same NestJS codebase: `api` (`main.ts`) and `worker` (`worker.main.ts`), communicating via PostgreSQL and Redis.
- **Reasoning:** Decouples user-facing API latency from background compute while sharing 100% of data models, schemas, and utility libraries.
- **Trade-offs:** Requires running two Node processes locally. (Fallback: unified single process supported for resource-constrained laptops).
- **Status:** **APPROVED**

---

### ADR-005: PostgreSQL 16+ & PostGIS 3.4+ as Authoritative Single Source of Truth
- **Category:** Data Persistence
- **Problem:** The platform manages complex geospatial geofences (150m radius, service polygons), concurrent driver reservations, and financial ledgers. Non-relational databases (MongoDB) lack mature multi-record ACID transactions and spatial indexing.
- **Alternatives Considered:**
  1. *MongoDB:* Rejected; weak spatial containment joins and lack of partial unique indexes.
  2. *MySQL 8:* Spatial functions are less feature-rich and performant than PostGIS.
- **Decision:** Standardize on **PostgreSQL 16+ with PostGIS 3.4+** as the sole authoritative persistence store.
- **Reasoning:** Absolute ACID consistency, partial unique indexes enforcing business invariants at the database engine level, and native spatial functions (`ST_DWithin`, `ST_Contains`).
- **Trade-offs:** Requires managing raw SQL queries alongside Prisma for PostGIS and pessimistic row locks.
- **Status:** **APPROVED**

---

### ADR-006: Redis 7.2+ Strictly for Ephemeral GPS and BullMQ Job Queues
- **Category:** Caching and Queuing
- **Problem:** Writing high-frequency driver GPS fixes (every 3–5 seconds) to PostgreSQL exhausts WAL disk write bandwidth.
- **Alternatives Considered:**
  1. *Writing all GPS to PostgreSQL:* Saturates disk I/O and creates massive database bloat.
  2. *In-Memory Node.js Maps:* Lost on process restart; cannot coordinate across API and Worker processes.
- **Decision:** Utilize **Redis 7.2+** strictly for ephemeral data: caching latest driver GPS with a 30s TTL, holding distributed matching leases, and backing BullMQ job queues.
- **Reasoning:** In-memory speed prevents database write saturation; BullMQ provides rock-solid delayed timers. PostgreSQL remains 100% authoritative if Redis crashes.
- **Trade-offs:** Introduces Redis as a local daemon dependency.
- **Status:** **APPROVED**

---

### ADR-007: Local OSRM on OpenStreetMap Data (No Custom Graph/A*)
- **Category:** Routing Engine
- **Problem:** Commercial routing APIs (Google Maps Platform, Mapbox) require billing credit cards and incur per-query charges that violate the ₹0 budget. Building a custom road graph and A* solver in TypeScript is an unnecessary, error-prone distraction.
- **Alternatives Considered:**
  1. *Google Directions API:* Prohibitively expensive; violates ₹0 budget constraint.
  2. *Custom A* Road Graph:* Massive complexity; reinvents road-routing wheel.
- **Decision:** Host a local instance of **OSRM (Open Source Routing Machine 5.27+)** using OpenStreetMap regional road network extracts covering Gandhinagar and PDEU.
- **Reasoning:** Unlimited free routing queries, sub-millisecond table matrices, accurate road turn restrictions, and zero subscription costs.
- **Trade-offs:** Static OSM road network does not reflect real-time live traffic congestion.
- **Status:** **APPROVED**

---

### ADR-008: Greedy Clustering + Exhaustive Permutations ($k! \le 24$)
- **Category:** Matching Algorithm
- **Problem:** Solving the full Vehicle Routing Problem (VRP) across an unbounded pool is NP-hard and computationally prohibitive for a fast-turnaround MVP.
- **Alternatives Considered:**
  1. *Global Integer Linear Programming (MIP):* Exponential solve time; solver licensing hurdles.
  2. *Greedy Insertion Heuristic:* Trapped in sub-optimal local minima; misses better drop-off permutations.
- **Decision:** Implement a **greedy spatial clustering heuristic** around the oldest seed booking, fetch an OSRM travel-time matrix, and **exhaustively evaluate all $k!$ drop-off permutations in memory** ($4! = 24$ maximum evaluations), filtering out any candidate with individual detour $> 25\%$.
- **Reasoning:** For $k \le 4$, evaluating 24 permutations takes $<0.1\text{ ms}$ in Node.js, guaranteeing the true road-network optimum while remaining fast, explainable, and robust.
- **Trade-offs:** Heuristic and bounded rather than globally system-wide optimal across the entire city.
- **Status:** **APPROVED**

---

### ADR-009: Group Capacity 2 to 4 Passengers with Group Preferences
- **Category:** Product & Business Rules
- **Problem:** Vehicles must maintain safe passenger loads while respecting commuter pooling preferences.
- **Alternatives Considered:**
  1. *Unbounded Group Size:* Violates physical vehicle seating capacity.
  2. *Forced Fixed 4-Pax Groups:* Causes commuters to wait excessively if only 2 or 3 riders are traveling along the corridor.
- **Decision:** Allow shared groups of **2, 3, or 4 passengers**. Allow commuters to choose their minimum group size preference (2, 3, or 4; default: 2), and enforce that all members' preferences must be satisfied.
- **Reasoning:** Balances vehicle seat utilization with waiting deadlines and commuter autonomy.
- **Trade-offs:** Preference matching slightly constrains combinatorial pool size.
- **Status:** **APPROVED**

---

### ADR-010: 60s Batch Interval with 120s Maximum Waiting Deadline
- **Category:** Matching Deadlines
- **Problem:** Commuters leaving campus gates cannot wait indefinitely for pooling matches.
- **Alternatives Considered:**
  1. *Continuous Real-Time Matching:* Generates suboptimal 2-person groups immediately rather than waiting 30 seconds for a full 4-person carpool.
  2. *5-Minute Batches:* Unacceptable commuter wait time leading to booking abandonment.
- **Decision:** Execute zone matching batches every **60 seconds**. Enforce a hard maximum waiting deadline of **120 seconds** from original booking creation. The deadline never resets during rematching.
- **Reasoning:** 60s allows sufficient demand aggregation at campus gates; 120s provides an acceptable, predictable SLA for student commuters.
- **Trade-offs:** Adds an initial batch accumulation latency of up to 60 seconds before first dispatch.
- **Status:** **APPROVED**

---

### ADR-011: Proportional Solo-Fare Allocation Formula
- **Category:** Financial Engineering
- **Problem:** Splitting the total shared vehicle fare equally penalizes passengers disembarking early, while pure distance splitting ignores shared detour costs.
- **Alternatives Considered:**
  1. *Equal Split ($G / N$):* Highly unfair to short-distance commuters.
  2. *Shapley Value Allocation:* Computationally complex ($2^k$ calculations) and difficult to explain to commuters.
- **Decision:** Allocate costs proportionally based on benchmark solo reference fares:
  $$F_i = G \times \frac{S_i}{\sum S_j}$$
- **Reasoning:** Mathematically transparent, guarantees that every commuter saves 20%–40% compared to a solo ride, and reconciles exactly with total driver earnings.
- **Trade-offs:** Requires pre-calculating solo direct road distances for all members.
- **Status:** **APPROVED**

---

### ADR-012: Private Estimated Fare as Shared Fare Ceiling
- **Category:** Consumer Protection
- **Problem:** If unexpected traffic detours occur, shared pooling fares could theoretically exceed the cost of taking a private auto, causing customer outrage.
- **Alternatives Considered:**
  1. *No Ceiling:* Exposes commuters to financial uncertainty.
  2. *Arbitrary Fixed Ceiling (e.g. ₹50):* Financially infeasible for long-distance trips.
- **Decision:** Lock the passenger's **private estimated fare** as their immutable **agreed fare ceiling**. If an allocated fare exceeds this ceiling, the group is rejected.
- **Reasoning:** Guarantees that a shared commuter will **never** pay more than the private direct fare quote.
- **Trade-offs:** Rejects groups where high detours cannot be absorbed within the ceiling.
- **Status:** **APPROVED**

---

### ADR-013: Exactly 2-Minute Fare Quote Expiry Window
- **Category:** Financial Policy
- **Problem:** Commuters holding stale fare quotes for hours could create bookings under obsolete tariff or road network conditions.
- **Alternatives Considered:**
  1. *30-Second Quotes:* Too short; creates user stress and frequent expiration errors.
  2. *15-Minute Quotes:* Road congestion or tariff adjustments make estimates stale.
- **Decision:** Set fare quote validity to **exactly 2 minutes (120 seconds)**. Expiry of the quote does not invalidate an already-created booking.
- **Reasoning:** Provides ample time for commuters to review pricing and select group preferences while maintaining fresh road estimates.
- **Trade-offs:** Users who hesitate past 2 minutes must request a fresh quote.
- **Status:** **APPROVED**

---

### ADR-014: First 5m Free, ₹5/min Waiting Charge with Ceiling Rounding Schedule
- **Category:** Operational Policy
- **Problem:** Drivers waiting excessively at campus gates lose earning time; passengers require a reasonable grace period to exit campus gates.
- **Alternatives Considered:**
  1. *Immediate Waiting Charge:* Frustrates students walking through campus turnstiles.
  2. *Zero Waiting Charge:* Drivers abandon rides due to uncompensated idle time.
- **Decision:** First **5 minutes are free**. After 5 minutes, charge **₹5 per started minute** ($W_i = 5 \times \lceil\max(0, T_i - 5)\rceil$) up to 10 minutes.
- **Reasoning:** Balanced grace period; ceiling rounding enforces accountability; individual charge kept separate from shared group fare.
- **Trade-offs:** Requires precise timestamping between driver arrival and individual passenger OTP verification.
- **Status:** **APPROVED**

---

### ADR-015: Fixed ₹30 Late Cancellation and No-Show Fees
- **Category:** Financial Penalties
- **Problem:** Commuters who cancel after driver dispatch or fail to show up inflict economic damage on drivers.
- **Alternatives Considered:**
  1. *₹0 Penalty:* Encourages reckless bookings and driver churn.
  2. *Full Fare Penalty:* Excessive for student budgets and causes severe dispute volume.
- **Decision:** Implement a **fixed ₹30.00 penalty** for cancellations occurring after driver assignment and for no-shows exceeding 10 minutes.
- **Reasoning:** ₹30 represents ~50% of an average solo fare, deterring frivolous cancellations while remaining fair to students.
- **Trade-offs:** Requires an audit ledger and administrative fee waiver dispute mechanism.
- **Status:** **APPROVED**

---

### ADR-016: No-Show Flat ₹30 Fee with Zero Additional Waiting Charge
- **Category:** Financial Policy
- **Problem:** Stacking waiting charges on top of no-show fees creates punitive penalties (₹30 + ₹25 = ₹55) that alienate campus commuters.
- **Alternatives Considered:**
  1. *Stacking Fees:* Charging both ₹30 no-show fee and ₹25 waiting fee.
- **Decision:** When a passenger is marked `NO_SHOW` after 10 minutes, assess a **flat ₹30.00 fee only**. Do **NOT** additionally charge waiting fees.
- **Reasoning:** Keeps penalties clear, predictable, and defensible before university administration.
- **Trade-offs:** Driver does not receive incremental waiting fees beyond the flat ₹30 recovery.
- **Status:** **APPROVED**

---

### ADR-017: Sequential 1-to-1 Driver Dispatch with 20s Countdown
- **Category:** Dispatch Architecture
- **Problem:** Broadcasting ride groups to all drivers concurrently creates race conditions, phantom rides, and driver frustration.
- **Alternatives Considered:**
  1. *Broadcast Dispatch:* High race condition frequency and driver dissatisfaction.
  2. *60-Second Sequential Timer:* Too slow; cascades exhaust commuter deadlines.
- **Decision:** Dispatch groups to **exactly one driver at a time** with a strict **20-second countdown timer**.
- **Reasoning:** Eliminates race conditions; 20 seconds is sufficient for a driver to glance at an offer while ensuring rapid cascades.
- **Trade-offs:** Requires an atomic reservation transaction upon acceptance.
- **Status:** **APPROVED**

---

### ADR-018: Driver Multi-Offer Receipt for Different Groups
- **Category:** Driver Utilization
- **Problem:** Artificially restricting a driver to one offer across the entire city when multiple groups form nearby starves the driver of options.
- **Alternatives Considered:**
  1. *Single Pending Offer per Driver:* Lowers system matching throughput.
- **Decision:** An active, available driver **MAY receive multiple pending dispatch offers for different groups**. Acceptance of one group atomically withdraws other offers.
- **Reasoning:** Maximizes driver utilization while ensuring groups are never disputed across accepted drivers.
- **Trade-offs:** Requires an atomic multi-table withdrawal transaction.
- **Status:** **APPROVED**

---

### ADR-019: Transactional Outbox Pattern for Guaranteed Job Publication
- **Category:** Reliability Architecture
- **Problem:** Writing domain state to PostgreSQL and publishing to BullMQ/Redis sequentially causes dual-write failures if Redis or Node crashes between writes.
- **Alternatives Considered:**
  1. *Two-Phase Commit (2PC):* Heavy and unsupported across Node.js/Redis.
  2. *Direct Dual Writes:* Unacceptable risk of orphaned database records.
- **Decision:** Implement the **Transactional Outbox Pattern** using an `outbox_events` table polled by a worker via `SELECT ... FOR UPDATE SKIP LOCKED`.
- **Reasoning:** Guarantees at-least-once message publication and decoupled background processing.
- **Trade-offs:** Requires consumers to be strictly idempotent.
- **Status:** **APPROVED**

---

### ADR-020: PostgreSQL is Authoritative; Redis Flushing Loses No Data
- **Category:** Storage Strategy
- **Problem:** Relying on Redis for persistent business state creates data loss risks during crashes or cache evictions.
- **Alternatives Considered:**
  1. *Redis-Primary Architecture:* In-memory durability risks.
- **Decision:** **PostgreSQL is the sole authoritative store of record.** If Redis is completely flushed (`FLUSHALL`) or restarted, PostgreSQL retains 100% data integrity.
- **Reasoning:** Enterprise-grade reliability and zero financial data loss.
- **Trade-offs:** High-frequency GPS fixes must be batched before persisting durable breadcrumbs.
- **Status:** **APPROVED**

---

### ADR-021: REST for Authoritative Mutations; Sockets for Notification Distribution
- **Category:** API & Realtime Design
- **Problem:** Executing state mutations over WebSockets introduces weak idempotency, difficult error handling, and security bypass risks.
- **Alternatives Considered:**
  1. *Full WebSocket RPC:* Poor REST tooling compatibility and complex error handling.
- **Decision:** **All state mutations execute via REST (`/api/v1`)**. Socket.IO (`/realtime`) is strictly downstream for notifications and telemetry streaming.
- **Reasoning:** Clean HTTP status codes, standard idempotency headers, and strict RBAC interceptors.
- **Trade-offs:** Clients maintain two transport connections (HTTP and WebSocket).
- **Status:** **APPROVED**

---

### ADR-022: Mid-Trip Driver Breakdown and Replacement Handover Policy
- **Category:** Incident Management
- **Problem:** Vehicle breakdowns during active trips leave commuters stranded and create immense platform liability.
- **Alternatives Considered:**
  1. *Instant Trip Cancellation:* Leaves students stranded on highways.
  2. *Manual Admin Phone Coordination:* Slow and unscalable.
- **Decision:** Mark trip `'INTERRUPTED'`, capture immutable manifest snapshot, and automatically search and dispatch an eligible replacement driver ranked by OSRM ETA.
- **Reasoning:** Safeguards student safety and automates vehicle recovery.
- **Trade-offs:** Requires managing handover states and manifest snapshots.
- **Status:** **APPROVED**

---

### ADR-023: 10% Interruption Discount on Trip Resumption
- **Category:** Customer Compensation
- **Problem:** Commuters subjected to a mid-trip vehicle breakdown experience delay and inconvenience.
- **Alternatives Considered:**
  1. *Zero Discount:* High passenger dissatisfaction.
  2. *Full 100% Free Ride on Resumption:* Economically penalizes the replacement driver who completed the journey.
- **Decision:** Apply an automatic **10% discount** off final ride fares for passengers onboard at the moment of breakdown when a replacement safely resumes the trip.
- **Reasoning:** Acknowledges commuter inconvenience while ensuring the replacement driver earns a viable fare.
- **Trade-offs:** Requires an explicit fare adjustment calculation and financial audit record.
- **Status:** **APPROVED**

---

### ADR-024: ₹0 Fare and Full Refund if Breakdown Cannot Be Resumed
- **Category:** Customer Protection
- **Problem:** If no replacement driver is available, commuters must find alternative transit after experiencing a breakdown.
- **Alternatives Considered:**
  1. *Pro-rated Distance Charge:* Unethical to charge passengers when stranded midway.
- **Decision:** If replacement dispatch fails, mark trip `'ABORTED'`. **All onboard passengers pay ₹0.00**. Issue a full cash refund if cash was handed over. Platform absorbs cost.
- **Reasoning:** Total customer fairness and protection under service failure.
- **Trade-offs:** Platform incurs operational loss.
- **Status:** **APPROVED**

---

### ADR-025: Shared Early Drop-Off Policy (Original Fare Unchanged)
- **Category:** Shared Transit Policy
- **Problem:** A passenger demanding to disembark early on a shared route could disrupt other passengers' fares or route sequences.
- **Alternatives Considered:**
  1. *Dynamic Pro-rated Recalculation:* Unfairly inflates remaining passengers' fares mid-trip.
  2. *Prohibiting Early Drop-Off:* Unsafe and legally indefensible.
- **Decision:** Disembarkation allowed only at a safe point already on the vehicle's agreed route. Journey marked `'COMPLETED'`. **Original allocated fare remains unchanged.** Other passengers' fares and route geometry remain untouched.
- **Reasoning:** Preserves financial predictability for all other commuters while allowing safe passenger egress.
- **Trade-offs:** Early-disembarking passenger pays the full originally agreed fare.
- **Status:** **APPROVED**

---

### ADR-026: Cash-Only Settlement Independent of Ride Completion
- **Category:** Financial Operations
- **Problem:** Micro-mobility in suburban India operates overwhelmingly in physical cash. Requiring digital UPI gateway integrations incurs fees and credit card requirements.
- **Alternatives Considered:**
  1. *UPI / Razorpay Integration:* Incurs transaction fees and merchant KYC requirements; violates ₹0 budget constraint.
- **Decision:** Implement **Cash-Only settlement** upon drop-off, decoupled from vehicle trip completion. Uncollected fees are tracked in a dedicated surcharge ledger.
- **Reasoning:** 100% adherence to ₹0 budget and reflects physical transport reality.
- **Trade-offs:** Requires trust in drivers confirming cash; uncollected fees remain outstanding until subsequent rides.
- **Status:** **APPROVED**

---

### ADR-027: Apple Silicon Mac Host for ₹0 Local Demo Environment
- **Category:** Deployment Infrastructure
- **Problem:** Cloud hosting (AWS, GCP, Render) incurs subscription fees, credit card authorisations, or sleep-mode throttles that ruin live evaluator demos.
- **Alternatives Considered:**
  1. *Free Cloud Tiers (Render / Fly.io):* Free tiers lack PostGIS extensions, sleep after 15m idle, and throttle WebSockets.
- **Decision:** Run all backend services (PostgreSQL, PostGIS, Redis, OSRM, NestJS API/Worker, Next.js Admin) locally on an Apple Silicon Mac host over local Wi-Fi.
- **Reasoning:** 100% free, zero cloud dependencies, ultra-low latency, and impervious to external internet outages.
- **Trade-offs:** Requires manual local daemon setup via Homebrew.
- **Status:** **APPROVED**

---

### ADR-028: Dedicated Demo Simulation Architecture and Control Center
- **Category:** Evaluation & Testing
- **Problem:** Evaluators cannot physically drive micro-mobility vehicles across Gandhinagar during a 20-minute project presentation.
- **Alternatives Considered:**
  1. *Static Mock UI Screenshots:* Fails to prove real-time algorithmic execution.
  2. *Mocked Backend Responses:* Does not test actual database concurrency or OSRM routing.
- **Decision:** Build a dedicated **Demo Simulation Architecture** with an interactive Next.js control center, virtual agent pool, and GPS playback engine executing against the real backend.
- **Reasoning:** Proves real-time algorithmic capability and data integrity under live evaluator control.
- **Trade-offs:** Requires maintaining virtual client simulation runners.
- **Status:** **APPROVED**

---

### ADR-029: Permanent Removal of Group Fingerprint Table in MVP
- **Category:** Database Optimization
- **Problem:** Previous designs included a `matching_group_attempts` table with SHA-256 member hashes to prevent re-offering rejected groups.
- **Alternatives Considered:**
  1. *Retaining Fingerprint Table:* Adds unnecessary table writes and index maintenance.
- **Decision:** **Permanently remove the group fingerprint table from the MVP database.**
- **Reasoning:** With the elimination of passenger acceptance voting (ADR-030), commuters cannot reject formed groups, rendering fingerprint tracking completely redundant.
- **Trade-offs:** None for MVP; full traceability is retained in `matching_batches` and `driver_dispatch_offers`.
- **Status:** **APPROVED**

---

### ADR-030: No Automatic Shared-to-Private Conversion without Explicit Consent
- **Category:** Consumer Consent
- **Problem:** Automatically converting a failed shared booking into a private ride surprises commuters with a 2x higher fare.
- **Alternatives Considered:**
  1. *Automatic Conversion:* Disregards student budget constraints.
- **Decision:** If shared matching fails, the system may offer a separately priced private ride, but **requires explicit passenger acceptance**. Automatic conversion is strictly forbidden.
- **Reasoning:** Protects student commuters from unintended financial commitments.
- **Trade-offs:** Requires user interaction if shared pooling is unavailable.
- **Status:** **APPROVED**

---

### ADR-031: No Mid-Trip Shared Destination Alteration in MVP
- **Category:** Scope & Routing Invariant
- **Problem:** Allowing passengers to change destinations mid-trip in a shared carpool requires dynamic real-time re-permutation of all downstream stops and recalculation of all other passengers' fares.
- **Alternatives Considered:**
  1. *Dynamic Mid-Trip Shared Re-Routing:* Extreme complexity; risks violating other passengers' detours and fare ceilings.
- **Decision:** Shared ride destinations are **immutable once boarded**. Mid-trip destination alterations are permitted **strictly for private rides**.
- **Reasoning:** Protects compliant shared commuters from unexpected detour delays and fare disputes.
- **Trade-offs:** Commuters who realize they picked the wrong stop must use the early drop-off policy.
- **Status:** **APPROVED**

---

## 4. Pending Specifications and Open Decisions Register

The following items are deliberately classified as **OPEN DECISION** or **PENDING SPECIFICATION**. Developers and AI coding agents must treat them as configuration-driven and must **NEVER hardcode assumed values**:

1. **OPEN-01 (Production Google OAuth Client IDs):** Final iOS Bundle ID, Android SHA-256 fingerprint, and web OAuth Client IDs for post-evaluation store releases. *(Currently configured via `.env` placeholders)*.
2. **OPEN-02 (Post-MVP Legal Review of ₹30 Late Fee):** Legal compliance review under Indian Consumer Protection (E-Commerce) Rules 2020 regarding late-fee disclosure before commercial rollout. *(Requires university legal / facility review)*.
3. **OPEN-03 (Exact Mathematical Formula for Estimating Point Within Shared Range):** While the range `[₹20, Private Estimated Fare]` is approved, the exact formula for showing a single "expected point" within that range (e.g. median vs 3-pax historical average) remains open as an empirical tuning parameter.

---

*End of Architecture Decision Records (ADRs) and Decision Register*
