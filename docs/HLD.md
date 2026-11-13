# High-Level Design (HLD)

## On-Demand Aggregation Platform for Last-Mile Connectivity Using Shared Micro-Mobility

---

## 1. System Context Diagram (C4 Level 1)

The System Context diagram illustrates the high-level boundary of the Last-Mile Micro-Mobility Platform, identifying external actors, client applications, and external identity and map tile providers:

```mermaid
flowchart TB
    subgraph External Actors
      P[Passenger\nPDEU Student / Commuter]
      D[Driver\nMicro-Mobility E-Rickshaw Partner]
      A[Campus Administrator\nTransport & Safety Facility Officer]
    end

    subgraph External Cloud Services
      GOOG[Google Identity Services\nOAuth 2.0 / OpenID Connect JWKS]
      OSM_T[OpenStreetMap Tile Server\nFree Leaflet Map Tiles]
    end

    subgraph Last-Mile Mobility System Boundary [Last-Mile Platform System Boundary - ₹0 Local Stack]
      SYS[On-Demand Shared Aggregation Platform\nNestJS Modular Monolith + Next.js Admin\nPostgreSQL/PostGIS + Redis + Local OSRM]
    end

    P -->|Searches rides, books shared trip, tracks vehicle, pays cash| SYS
    D -->|Receives 1-to-1 offers, streams GPS, completes multi-drop trip| SYS
    A -->|Approves drivers, monitors active rides, reviews fee waivers| SYS

    SYS -->|Verifies Google ID Tokens against JWKS| GOOG
    P -.->|Fetches map render tiles directly| OSM_T
    D -.->|Fetches map render tiles directly| OSM_T
    A -.->|Fetches map render tiles directly| OSM_T
```

---

## 2. Container Diagram (C4 Level 2)

The Container diagram decomposes the platform boundary into executable runtime containers, data stores, background workers, and client applications:

```mermaid
flowchart TB
    subgraph Clients [Client Applications Layer]
      P_App[Passenger Mobile App\nFlutter / iOS & Android\nProvider + go_router + flutter_map]
      D_App[Driver Mobile App\nFlutter / iOS & Android\nGeolocator + socket_io_client]
      A_Web[Admin Dashboard\nNext.js 14+ / TypeScript\nTailwind CSS + Server Actions]
    end

    subgraph Application Runtime [Application Services Layer - Shared NestJS Codebase]
      API_Proc[API Process Container\nNestJS Modular Monolith\nREST Controllers + Socket.IO Gateway]
      WRK_Proc[Worker Process Container\nNestJS Background Engine\nBullMQ Job Consumers + Outbox Publisher]
    end

    subgraph Data Stores [Persistence & Caching Layer]
      PG[(Authoritative Database\nPostgreSQL 16 + PostGIS 3.4\nSpatial SRID 4326 + ACID Store)]
      RD[(In-Memory Cache & Broker\nRedis 7.2\nBullMQ Queues + 30s GPS TTL)]
    end

    subgraph Routing Engine [Local Routing Engine Container]
      OSRM[OSRM Engine\nC++ Routing Binary\nOpenStreetMap Gandhinagar Road Network]
    end

    P_App -->|HTTPS REST /api/v1| API_Proc
    P_App -->|WSS Socket.IO| API_Proc
    D_App -->|HTTPS REST /api/v1| API_Proc
    D_App -->|WSS GPS Stream / Events| API_Proc
    A_Web -->|HTTPS REST /api/v1| API_Proc

    API_Proc -->|Prisma ORM + PostGIS Raw SQL| PG
    API_Proc -->|ioredis Cache / Socket Session State| RD
    API_Proc -->|HTTP OSRM Matrix / Route| OSRM

    WRK_Proc -->|Transactional Outbox Reader + Prisma| PG
    WRK_Proc -->|BullMQ Queue Consumer & Locks| RD
    WRK_Proc -->|HTTP OSRM Matrix API| OSRM
```

---

## 3. Modular Monolith Component Architecture (C4 Level 3)

The backend is organized into 14 distinct domain modules within a single NestJS application. Each module has strictly bounded responsibilities, owned entities, and well-defined export contracts.

```mermaid
flowchart TD
    subgraph Core Modules [Identity & Administration Modules]
      Auth[AuthModule\nGoogle JWT, Refresh Sessions]
      Users[UsersModule\nProfiles, Roles, Status]
      Drivers[DriversModule\nOnboarding, Verification, Vetting]
      Vehicles[VehiclesModule\nVehicle Metadata, Seating Capacity]
      Admin[AdminModule\nApprovals, Operations, Fee Waivers]
    end

    subgraph Operational Modules [Ride Lifecycle & Geospatial Modules]
      Bookings[BookingsModule\nBooking Lifecycle, Active Invariant]
      Locations[LocationsModule\nPostGIS Geofencing, GPS Ingestion]
      RoutingModule[RoutingModule\nOSRM Table Matrix, Route Geometry]
      FaresModule[FaresModule\nProportional Cost Formula, Ledgers]
    end

    subgraph Algorithmic Engine [Matching & Dispatching Engine]
      Matching[MatchingModule\nZone Leases, Permutation Optimizer]
      Dispatch[DispatchModule\n1-to-1 Driver Dispatch, 20s Timers]
    end

    subgraph Infrastructure Modules [Cross-Cutting & Persistence Modules]
      Infra[InfrastructureModule\nPrisma, Redis, Outbox Publisher]
      Notif[NotificationsModule\nSocket.IO Gateway, Push Service]
      Payments[PaymentsModule\nCash Settlement, Fee Collections]
    end

    Auth --> Users
    Drivers --> Users
    Drivers --> Vehicles
    Admin --> Drivers
    Admin --> Bookings

    Bookings --> Users
    Bookings --> Locations
    Bookings --> FaresModule

    Matching --> Bookings
    Matching --> RoutingModule
    Matching --> FaresModule

    Dispatch --> Matching
    Dispatch --> Drivers
    Dispatch --> RoutingModule

    Locations --> Infra
    Payments --> Bookings
    Payments --> FaresModule

    Matching --> Infra
    Dispatch --> Infra
    Infra --> Notif
```

---

## 4. Core Architectural Patterns

### 4.1 Modular Monolith with Domain-Driven Design (DDD)
- **Elimination of Distributed Transaction Overhead:** The platform coordinates bookings, matching groups, driver reservations, and payment ledgers inside atomic PostgreSQL transactions without distributed two-phase commits.
- **Strict Module Encapsulation:** Modules do not execute direct foreign database writes. If `MatchingModule` needs to claim a booking, it invokes `BookingsService.claimBookingForGroup()`, ensuring domain validation rules remain unbroken.
- **Unified Codebase:** Code duplication is eliminated; backend DTOs, database models, and enum definitions are shared between API and Worker processes.

### 4.2 Dual Process Execution Model
The backend compiles into a single set of artifacts but executes in two separate runtime roles:
1. **API Process (`main.ts`):** 
   - Handles client HTTP REST traffic and Socket.IO WebSocket connections.
   - Executes fast, synchronous OLTP database operations (User login, Booking creation, Driver acceptance, GPS ingestion).
   - Enqueues background work strictly via the Transactional Outbox.
2. **Worker Process (`worker.main.ts`):** 
   - Runs BullMQ job consumers and scheduled cron tasks.
   - Executes heavy, asynchronous compute tasks: running 60s matching batch algorithms, evaluating 24 permutation route matrices via OSRM, managing 20s driver dispatch countdowns, and running stale lease reconciliations.
   - Does not expose public HTTP ports.

### 4.3 Transactional Outbox Pattern
To eliminate dual-write hazards between PostgreSQL and Redis, all asynchronous operations commit an outbox event within the primary business transaction:
1. State mutation and outbox event insertion occur atomically in PostgreSQL.
2. A lightweight background poller in the worker process reads `PENDING` outbox records using `SELECT ... FOR UPDATE SKIP LOCKED`.
3. The poller enqueues corresponding jobs into BullMQ.
4. Upon successful Redis acknowledgment, the outbox record is marked `PUBLISHED`.
5. If the process dies, uncommitted Redis jobs are safely recovered and re-enqueued on restart.

### 4.4 Ephemeral vs. Authoritative Data Segregation
- **Authoritative Data (PostgreSQL 16 + PostGIS):** Bookings, Ride Groups, Group Memberships, Driver Assignments, Fares, Cancellation Fee Ledgers, Cash Payment Records, and User Identities.
- **Ephemeral Data (Redis 7.2):** High-frequency driver GPS locations (30s TTL), BullMQ queue job states, distributed matching batch locks (leases), and active Socket.IO connection session maps.

---

## 5. End-to-End System Data Flows

### 5.1 Ingress Booking Data Flow

```mermaid
sequenceDiagram
    autonumber
    actor P as Passenger (Flutter)
    participant API as NestJS API Process
    participant PG as PostgreSQL / PostGIS
    participant OB as Outbox Publisher
    participant RD as Redis / BullMQ

    P->>API: POST /api/v1/bookings (pickupZoneId, destLat, destLng, type="SHARED")
    API->>PG: Begin ACID Transaction (READ COMMITTED)
    API->>PG: Check Active Booking Invariant (SELECT WHERE passenger_id AND status active)
    API->>PG: ST_DWithin Geofence Check (150m from PDEU Center)
    API->>PG: ST_Contains Destination Polygon Check
    API->>PG: INSERT INTO bookings (status="REQUESTED", type="SHARED")
    API->>PG: INSERT INTO outbox_events (eventType="BOOKING_CREATED", status="PENDING")
    API->>PG: COMMIT
    API-->>P: 201 Created (bookingId, status="REQUESTED")

    OB->>PG: SELECT PENDING outbox_events FOR UPDATE SKIP LOCKED
    OB->>RD: Enqueue matching job in 'matching' queue
    OB->>PG: UPDATE outbox_events SET status = 'PUBLISHED'
```

### 5.2 Matching and Permutation Routing Data Flow

```mermaid
sequenceDiagram
    autonumber
    participant WRK as Worker Process
    participant RD as Redis (Lease / Lock)
    participant PG as PostgreSQL
    participant OSRM as Local OSRM Engine
    participant OB as Outbox Publisher

    WRK->>RD: Acquire Zone Matching Lease (key: "lease:matching:pdeu", ttl: 30s)
    WRK->>PG: BEGIN
    WRK->>PG: SELECT eligible bookings WHERE zone='PDEU' AND status='REQUESTED' FOR UPDATE
    WRK->>PG: INSERT INTO matching_batches (status="PROCESSING")
    WRK->>PG: COMMIT
    
    Note over WRK,OSRM: Permutation Routing & Feasibility Evaluation
    WRK->>OSRM: POST /table/v1/driving (Pickup + Candidate Destinations)
    OSRM-->>WRK: Travel Time & Distance Matrix (N x N)
    WRK->>WRK: Enumerate drop-off permutations (k! <= 24)
    WRK->>WRK: Calculate individual detours (discard routes > 25% detour)
    WRK->>WRK: Select route minimizing vehicle travel time
    WRK->>OSRM: GET /route/v1/driving (Optimal Permutation Stops)
    OSRM-->>WRK: Full Route Polyline Geometry & Distance
    
    WRK->>PG: BEGIN
    WRK->>PG: INSERT INTO ride_groups (status="DISPATCHING", route_geometry, total_fare)
    WRK->>PG: INSERT INTO group_members (allocating proportional fares)
    WRK->>PG: UPDATE bookings SET status = 'MATCHING'
    WRK->>PG: INSERT INTO outbox_events (eventType="GROUP_FORMED_DISPATCH_TRIGGERED")
    WRK->>PG: COMMIT
    WRK->>RD: Release Zone Matching Lease
```

### 5.3 Sequential 1-to-1 Driver Dispatch Data Flow

```mermaid
sequenceDiagram
    autonumber
    participant WRK as Worker Process (Dispatch Engine)
    participant PG as PostgreSQL
    participant RD as Redis (Driver Locations & Timers)
    participant SIO as Socket.IO Gateway
    actor D1 as Driver 1 (Flutter)
    actor D2 as Driver 2 (Flutter)

    WRK->>PG: Find eligible drivers (approved, online, available, capacity >= group pax)
    WRK->>RD: Fetch latest driver GPS coordinates
    WRK->>WRK: Rank drivers by OSRM ETA to PDEU Pickup Hub
    
    Note over WRK,D1: 1-to-1 Dispatch: Group offered ONLY to Driver 1
    WRK->>PG: INSERT INTO driver_dispatch_offers (groupId, driverId=D1, status="PENDING", timeout_at=NOW()+20s)
    WRK->>SIO: Push driver:offer:received to D1 socket room
    WRK->>RD: Schedule BullMQ Delayed Job: 'expire-driver-offer' in 20s
    SIO-->>D1: Offer Alert with 20s Countdown Timer

    alt Driver 1 Accepts within 20s
        D1->>PG: POST /drivers/offers/:id/accept (Executes Atomic Acceptance Transaction)
        Note over PG: Lock group, conditionally update D1 to RESERVED, assign D1 to Ride
        PG-->>D1: 200 OK (Assigned)
        SIO-->>D1: Withdraw other pending offers
    else Driver 1 Rejects or Times Out (20s expires)
        Note over WRK,PG: Mark D1 offer REJECTED / EXPIRED. Group stays INTACT!
        WRK->>PG: UPDATE driver_dispatch_offers SET status = 'EXPIRED' WHERE id = offer1
        Note over WRK: Driver 1 excluded for this group version. Cascades to Driver 2!
        WRK->>PG: INSERT INTO driver_dispatch_offers (groupId, driverId=D2, status="PENDING")
        WRK->>SIO: Push driver:offer:received to D2 socket room
        SIO-->>D2: Offer Alert with 20s Countdown Timer
    end
```

### 5.4 Real-Time Telemetry and GPS Ingestion Data Flow

```mermaid
sequenceDiagram
    autonumber
    actor D as Driver (Flutter)
    participant SIO as Socket.IO Gateway
    participant RD as Redis (Ephemeral Cache)
    participant PG as PostgreSQL (Durable Snapshots)
    actor P as Passengers (Flutter)

    loop Every 3-5 seconds while Trip is ACTIVE
        D->>SIO: Emit driver:location:update (lat, lng, heading, speed, timestamp)
        SIO->>SIO: Verify Driver Active Assignment & Authenticity
        SIO->>RD: SET driver:location:<driverId> (JSON payload, EX 30)
        SIO->>SIO: Broadcast to room: ride:<rideId> (driver:location:broadcast)
        SIO-->>P: Live Vehicle Marker Updates on Map
        
        opt Every 15-30s or on Stop Drop-off
            SIO->>PG: INSERT INTO ride_locations (ride_id, location, recorded_at)
        end
    end
```

### 5.5 Cash Reconciliation and Cancellation Fee Recovery Data Flow

```mermaid
sequenceDiagram
    autonumber
    actor P as Passenger (Flutter)
    actor D as Driver (Flutter)
    participant API as NestJS API Process
    participant PG as PostgreSQL
    participant SIO as Socket.IO Gateway

    Note over P,API: Scenario: Passenger has an outstanding ₹30 late cancellation fee
    P->>API: GET /passengers/cancellation-fees/outstanding
    API-->>P: { hasOutstandingFee: true, amount: 30.00 }
    
    P->>API: POST /bookings (Creates new ride)
    Note over P,API: Booking confirms total cash due = Current Fare (₹50) + Fee (₹30) = ₹80
    
    Note over D,P: Ride executes. Vehicle arrives at Passenger Drop-off.
    D->>API: POST /rides/:id/complete
    P->>D: Hands ₹80 physical cash to driver
    
    D->>API: POST /payments/:id/confirm-cash (amountCollected: 80.00)
    API->>PG: BEGIN ACID Transaction
    API->>PG: INSERT INTO cash_payment_records (booking_id, amount=80.00, status="PAID")
    API->>PG: UPDATE cancellation_fees SET status = 'COLLECTED', collected_at = NOW(), collected_booking_id = :newBookingId WHERE passenger_id = :pId AND status = 'OUTSTANDING'
    API->>PG: UPDATE bookings SET payment_status = 'PAID' WHERE id = :newBookingId
    API->>PG: COMMIT
    
    API-->>D: 200 OK (Payment Confirmed)
    API->>SIO: Broadcast payment:status (status="PAID") to passenger room
    SIO-->>P: Receipt Screen Displayed: Fare ₹50 + Prior Fee ₹30 = ₹80 Settled
```

---

## 6. Module Communication Matrix

To preserve modular architectural purity, the table below documents the permitted communication mechanism between domain modules:

| Calling Module | Target Module | Communication Mechanism | Payload / Purpose | Consistency Guarantee |
|---|---|---|---|---|
| `BookingsModule` | `LocationsModule` | Synchronous Service Call | Validates passenger coordinates against PostGIS geofence | Immediate / Strong |
| `BookingsModule` | `FaresModule` | Synchronous Service Call | Fetches solo fare estimate and checks outstanding cancellation fees | Immediate / Strong |
| `BookingsModule` | `InfrastructureModule` | Transactional Outbox (PostgreSQL) | Writes `BOOKING_CREATED` event inside primary booking transaction | Atomic / Guaranteed |
| `MatchingModule` | `RoutingModule` | Synchronous Service Call | Calls OSRM Table matrix API to evaluate travel-time permutations | Immediate In-Memory |
| `MatchingModule` | `FaresModule` | Synchronous Service Call | Applies proportional fare allocation formula across group members | Immediate / Strong |
| `MatchingModule` | `BookingsModule` | Synchronous Service Call | Atomically updates booking statuses from `REQUESTED` to `MATCHING` | Immediate / Strong |
| `MatchingModule` | `InfrastructureModule` | Transactional Outbox (PostgreSQL) | Writes `GROUP_FORMED_DISPATCH_TRIGGERED` event | Atomic / Guaranteed |
| `DispatchModule` | `DriversModule` | Synchronous Service Call | Filters eligible drivers and ranks candidates by ETA | Immediate / Strong |
| `DispatchModule` | `InfrastructureModule` | BullMQ Job Queue (Redis) | Enqueues 20-second delayed timeout job (`handle-driver-offer-timeout`) | At-Least-Once Delivery |
| `DispatchModule` | `NotificationsModule` | Asynchronous Event via Outbox | Dispatches `driver:offer:received` Socket.IO event to driver | Eventual Real-Time |
| `LocationsModule` | `NotificationsModule` | In-Memory Gateway Broadcast | Broadcasts high-frequency GPS coordinate stream to `ride:<id>` room | Ephemeral Best-Effort |
| `PaymentsModule` | `FaresModule` | Synchronous Service Call | Updates `cancellation_fees` status from `OUTSTANDING` to `COLLECTED` | Immediate / Strong |
| `AdminModule` | `DriversModule` | Synchronous Service Call | Manages driver onboarding approval, rejection, and suspension | Immediate / Strong |

---

*End of High-Level Design (HLD)*

