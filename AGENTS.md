# AI Coding Agent Instructions and Governance Directives

## On-Demand Aggregation Platform for Last-Mile Connectivity Using Shared Micro-Mobility

---

> [!IMPORTANT]
> **MANDATORY READING FOR ALL AI ASSISTANTS AND CODING AGENTS:**
> You must read and strictly adhere to the architecture specifications located in `docs/` before generating code, scaffolding projects, creating database migrations, or proposing technical modifications. The decisions in `docs/` take absolute precedence over generic coding conventions or outside defaults.

---

## 1. Absolute Architecture Guardrails

1. **Strictly Monolithic (No Microservices):**
   - The backend MUST remain a single **NestJS modular monolith** located in `apps/backend`.
   - Never introduce independent microservices, gRPC service meshes, Kafka/RabbitMQ clusters, or API gateways (Kong/Traefik).
   - Domain modules must communicate via synchronous TypeScript service interfaces in-process, or asynchronously via the **Transactional Outbox** pattern.

2. **PostgreSQL + PostGIS is the Sole Authoritative Truth:**
   - PostgreSQL 16+ is the single source of truth for all bookings, ride groups, driver assignments, financial ledgers, and user accounts.
   - **Redis is strictly an ephemeral cache** for volatile driver GPS fixes (30s TTL) and a broker for BullMQ job queues.
   - Never treat Redis, in-memory Node.js Maps, or Socket.IO state as authoritative. If Redis is flushed, PostgreSQL must retain 100% data integrity.

3. **Backend Authorization is Absolute (Zero Frontend Trust):**
   - The client application mode selection ("Passenger" or "Driver" in Flutter) is strictly a user interface state, NOT a security boundary.
   - The backend MUST enforce Role-Based Access Control (RBAC) on every endpoint.
   - A user cannot execute driver endpoints unless their `driver_profiles` record exists in PostgreSQL with `verification_status = 'APPROVED'`.
   - Never trust client-asserted fares, pickup locations, driver operational statuses, or booking ownership.

4. **Distributed Concurrency and Locking Guardrails:**
   - **Never use in-memory locks** (e.g., `async-mutex`, Node.js variables) to coordinate concurrency across distributed API or worker instances.
   - Concurrency must be coordinated strictly using PostgreSQL primitives:
     - Explicit pessimistic row locks: `SELECT ... FOR UPDATE`.
     - Outbox queue claiming: `SELECT ... FOR UPDATE SKIP LOCKED`.
     - Atomic conditional compare-and-swap updates: `UPDATE driver_profiles SET operational_status = 'RESERVED' WHERE id = :id AND operational_status = 'AVAILABLE'`.
     - Partial unique indexes enforced by the database engine.
   - **Never hold database transactions open** while awaiting external HTTP calls (OSRM), Socket.IO broadcasts, or user input.

5. **Mandatory Idempotency for All State Mutations:**
   - All state-mutating REST endpoints (booking creation, ride cancellations, driver acceptance) must accept and enforce an `Idempotency-Key` header verified against `idempotency_records`.
   - All BullMQ background job consumers must be strictly idempotent: they must query the authoritative PostgreSQL row by UUID and verify current state before executing mutations.

---

## 2. Critical Product and Business Rules

### 2.1 The Matching Paradigm (No Passenger Acceptance Voting)
- **DO NOT implement a post-matching passenger group acceptance round.**
- Do not create passenger voting countdowns, unanimous offer acceptance checks, or passenger rejection handlers.
- Passengers give **upfront consent** to automatic grouping, maximum 4-passenger vehicle capacity, capped fares, and a maximum 25% individual detour during the booking flow (`POST /bookings`).
- The matching algorithm forms a feasible group automatically; the group moves **IMMEDIATELY** into driver dispatch.

### 2.2 Removal of Group Fingerprint Exclusion Tables
- **DO NOT create a dedicated `matching_group_attempts` or `group_fingerprint` table in the MVP database.**
- Because passenger acceptance voting has been eliminated, the loop where passengers repeatedly reject the same group is gone.
- Operational traceability is maintained through standard tables: `matching_batches`, `batch_bookings`, `ride_groups`, `group_members`, and `driver_dispatch_offers`.

### 2.3 Driver Dispatch and Multi-Offer Rules
- **Strict 1-to-1 Group Dispatch:** A ride group is offered to **EXACTLY ONE driver at a time** with a 20-second timeout. Never broadcast a group to multiple drivers concurrently.
- **Driver Multi-Offer Receipt:** An active, available driver MAY receive multiple pending dispatch offers for different groups.
- **Atomic Driver Reservation:** The moment a driver accepts an offer, an atomic transaction reserves the driver, assigns the group, marks the accepted offer `ACCEPTED`, withdraws the driver's other pending offers, and marks competing offers for that group `WITHDRAWN`.
- **Driver Rejection / Timeout Invariant:**
  - If a driver explicitly rejects (`POST /drivers/offers/:id/reject`) or times out (20s elapses), **THE PASSENGER GROUP REMAINS INTACT**.
  - Passengers are NOT returned to matching.
  - The rejecting driver is excluded **strictly for the current group version and dispatch attempt**.
  - The group cascades immediately to the next ranked eligible driver.
  - If the group configuration changes (e.g., a member cancels), the group version increments, and the driver becomes eligible again.

### 2.4 Cancellation and Late Fee Rules
- **Cancellation Before Driver Assignment:** Free cancellation (₹0 fee). The remaining group is recalculated: if $\ge 2$ passengers and detour $\le 25\%$, the group is preserved; otherwise, it is dissolved and compliant members are rematched, **strictly preserving their original booking creation timestamps**.
- **Cancellation After Driver Assignment:** Assesses a **fixed ₹30 late cancellation fee** inserted into `cancellation_fees` as `OUTSTANDING`. Exempt driver inactivity $> 5\text{m}$, platform errors, and administrative waivers.
- **Cash-Only Recovery on Next Ride:** Surcharge outstanding fees on the passenger's subsequent ride. Collect and mark `COLLECTED` atomically when the driver confirms cash receipt. **A cancellation fee must never be collected twice.**

---

## 3. Database Invariants to Enforce

When generating Prisma migrations or executing raw SQL queries, you must guarantee the following 11 database invariants:

1. **One Active Booking per Passenger:** Enforced via partial unique index:
   `CREATE UNIQUE INDEX idx_unique_active_passenger_booking ON bookings (passenger_id) WHERE status IN ('REQUESTED', 'MATCHING', 'DISPATCHING', 'CONFIRMED', 'IN_PROGRESS');`
2. **One Active Group Membership per Booking:** A booking belongs to at most one active group:
   `CREATE UNIQUE INDEX idx_unique_active_booking_in_group ON group_members (booking_id) WHERE is_active_member = true;`
3. **One Active Assignment per Driver:** Enforced via partial unique index:
   `CREATE UNIQUE INDEX idx_unique_active_driver_assignment ON ride_assignments (driver_id) WHERE status IN ('ASSIGNED', 'ARRIVED', 'IN_PROGRESS');`
4. **One Active Assignment per Group:** Enforced via partial unique index:
   `CREATE UNIQUE INDEX idx_unique_active_group_assignment ON ride_assignments (ride_group_id) WHERE status IN ('ASSIGNED', 'ARRIVED', 'IN_PROGRESS');`
5. **Single Pending Offer per Group:** A group has at most one pending driver offer:
   `CREATE UNIQUE INDEX idx_unique_single_pending_offer_per_group ON driver_dispatch_offers (ride_group_id) WHERE status = 'PENDING';`
6. **Scoped Driver Offer Exclusion:** Scoped to group version and dispatch attempt:
   `CREATE UNIQUE INDEX idx_unique_driver_offer_per_group_version ON driver_dispatch_offers (ride_group_id, group_version, dispatch_attempt_id, driver_id);`
7. **No Duplicate Fee Collection:** A cancellation fee can only be settled once:
   `CREATE UNIQUE INDEX idx_unique_collected_booking_fee ON cancellation_fees (collected_booking_id) WHERE status = 'COLLECTED';`
8. **Group Capacity Limits:** A shared group must maintain between 2 and 4 passengers while active for dispatch.
9. **Ownership Protection:** A passenger can never cancel or query another passenger's booking.
10. **Driver Eligibility:** Only administrator-approved (`verification_status = 'APPROVED'`), online (`is_online = true`), unsuspended drivers can receive or accept offers.
11. **PostGIS Geofencing:** Pickup coordinates must fall within 150m of the PDEU zone center; drop-off coordinates must reside strictly within the Gandhinagar service region polygon.

---

## 4. Environment and Financial Guardrails

1. **Strict ₹0 Budget Constraint:**
   - Do not integrate paid third-party APIs (Google Directions, Mapbox SDK, Twilio, SendGrid, Stripe, Razorpay).
   - Use local OSRM on OpenStreetMap data, free OSM raster tiles via `flutter_map`, and cash settlement.
2. **Mac Host Machine Safety:**
   - **Do not alter global macOS configuration:** Do not modify the system `PATH`, change system Python `/usr/bin/python3`, modify Homebrew system permissions, or reconfigure global network proxies.
   - Keep all credentials, database passwords, and secrets inside local `.env` files (ignored by Git). Commit `.env.example` placeholders only.
3. **No Premature DevOps / CI/CD:**
   - Do not add GitHub Actions CI/CD workflows, Docker Compose requirements, or Kubernetes manifests to the MVP deliverable.
4. **Do Not Fabricate Pending Decisions:**
   - Do not hardcode assumed values for items listed as **PENDING** in [docs/ARCHITECTURE_DECISIONS.md](./docs/ARCHITECTURE_DECISIONS.md) (such as exact base fare tariffs, exact material change thresholds, or final token lifespans). Prompt the user or mark them as configurable parameters.

---

*Adhere to these instructions strictly across all implementation tasks.*

