# On-Demand Aggregation Platform for Last-Mile Connectivity Using Shared Micro-Mobility

[![Architecture Status](https://img.shields.io/badge/Architecture-Approved%20%26%20Complete-success)](./docs/ARCHITECTURE_DECISION_RECORDS.md)
[![Target Budget](https://img.shields.io/badge/Budget-%E2%82%B90%20(Zero%20Cost)-blue)](./docs/DEPLOYMENT_AND_ENVIRONMENT.md)
[![Deployment](https://img.shields.io/badge/Deployment-Local%20Host%20(Apple%20Silicon%20Mac)-orange)](./docs/DEPLOYMENT_AND_ENVIRONMENT.md)
[![Design](https://img.shields.io/badge/Design-NestJS%20Modular%20Monolith-red)](./docs/SYSTEM_ARCHITECTURE.md)
[![Platforms](https://img.shields.io/badge/Mobile-Flutter%20(iOS%20%26%20Android)-purple)](./docs/PROJECT_OVERVIEW.md)

---

## 1. Executive Summary and Problem Statement

Suburban university campuses and transit hubs—such as **Pandit Deendayal Energy University (PDEU)** in Gandhinagar, Gujarat—suffer from severe "last-mile" transportation inefficiencies. Hundreds of students, faculty, and daily commuters exit transit gates simultaneously heading along common corridors (e.g., Bhaijipura, Kudasan, Koba Circle, Raysan, Infocity, Sector 1–30).

### The Traditional Commute Dilemma
- **High Individual Cost:** Commuters take private point-to-point autos or cabs costing ₹80–₹150 per ride.
- **Low Vehicle Utilization:** Autos frequently carry only a single commuter, worsening road congestion, emissions, and wait times.
- **Sidewalk Chaos:** Informal street haggling and manual coordination waste 15–30 minutes at campus gates.

### The Platform Solution
This platform automates real-time demand aggregation for shared micro-mobility vehicles (4-passenger electric rickshaws, bike taxis, and micro-shuttle vans):
1. **Geofenced Campus Aggregation:** Validates commuters at dedicated campus transit hubs via strict 150-meter backend GPS geofences using PostGIS.
2. **Upfront Consent & Automatic Grouping:** Commuters give upfront consent to grouping, capacity limits ($\le 4$ passengers), capped fares, and maximum 25% individual detours at booking. Fragile post-matching passenger voting rounds are completely eliminated.
3. **OSRM Permutation Route Solver:** Evaluates road-network travel-time matrices ($k! \le 24$ permutations) via a local Open Source Routing Machine (OSRM) instance to select the optimal drop-off sequence that minimizes vehicle travel time while respecting detour limits.
4. **Proportional Solo-Fare Allocation:** Divides shared trip costs equitably based on individual benchmark solo travel costs, saving commuters 20%–50% on every ride.
5. **Sequential 1-to-1 Driver Dispatch:** Dispatches formed groups to available, verified drivers one at a time with a 20-second timer, atomically locking driver reservations.
6. **Mid-Trip Breakdown Resilience:** Automatically manages vehicle breakdowns, snapshots immutable manifests, dispatches replacement drivers, applies 10% discounts on resumption, or aborts with ₹0 fare and full cash refunds.
7. **Cash-Only Settlement & ₹30 Surcharge Ledger:** Audits physical cash collections and manages post-assignment late cancellation and no-show penalties through an automated next-ride surcharge recovery ledger.

---

## 2. System Architecture and Technology Stack

The entire architecture is designed to execute locally on an **Apple Silicon Mac host** with physical mobile devices and simulators connecting over a shared local Wi-Fi router under a **strict ₹0 budget constraint**:

```mermaid
flowchart TB
    subgraph Clients [Client Applications - Cross Platform]
      P_Mob[Flutter Mobile App - Passenger Mode\nProvider + go_router + flutter_map]
      D_Mob[Flutter Mobile App - Driver Mode\ngeolocator + socket_io_client]
      A_Web[Next.js 14+ Admin Dashboard\nTypeScript + Tailwind CSS]
    end

    subgraph Backend [Modular Monolith - Single Codebase]
      API[NestJS API Process\nREST Endpoints (/api/v1) + Socket.IO Gateway (/realtime)]
      WRK[NestJS Background Worker\nBullMQ Processors + Outbox Engine]
    end

    subgraph Persistence [Local Infrastructure - Mac Host]
      PG[(PostgreSQL 16 + PostGIS 3.4\nAuthoritative ACID Store)]
      RD[(Redis 7.2\nBullMQ Queues + 30s GPS TTL)]
      OSRM[OSRM Routing Server\nOpenStreetMap Gandhinagar Road Network]
    end

    P_Mob & D_Mob & A_Web -->|HTTP / WebSockets| API
    API -->|Prisma + Raw SQL| PG
    API -->|ioredis Cache| RD
    API & WRK -->|HTTP REST| OSRM
    WRK -->|Outbox Claim & Prisma| PG
    WRK -->|BullMQ Jobs & Leases| RD
```

### Technology Matrix

| Layer | Technologies Selected | Rationale & Governance Rule |
|---|---|---|
| **Mobile Applications** | **Flutter 3.22+, Dart 3.4+**, Provider, go_router, Dio, flutter_secure_storage, geolocator, socket_io_client, flutter_map | Single codebase supporting both Android and iOS as official first-class targets. Initial development prioritized on VS Code + iOS Simulator + iPhone. |
| **Admin Dashboard** | **Next.js 14+ (App Router)**, TypeScript, Tailwind CSS, Leaflet | Responsive administrative portal for driver vetting, fleet monitoring, fee waivers, and demo simulation orchestration. |
| **Backend Monolith** | **NestJS 10+ Modular Monolith**, TypeScript 5+ | Single codebase exposing dual runnable entry points (`start:api` and `start:worker`). Microservices are strictly prohibited. |
| **Primary Persistence** | **PostgreSQL 16+, PostGIS 3.4+**, Prisma ORM | Sole authoritative single source of truth for all users, bookings, groups, trips, and financial ledgers. Enforces 11+ partial unique indexes. |
| **Ephemeral Cache & Queues**| **Redis 7.2+, BullMQ 5+**, ioredis | Ephemeral GPS telemetry cache (30s TTL) and job queue broker. If flushed, PostgreSQL retains 100% data integrity. |
| **Road Routing Engine** | **OSRM 5.27+ (Open Source Routing Machine)** | Local routing daemon running on Gandhinagar OpenStreetMap data. Unlimited free queries; sub-millisecond matrix calculations. |
| **Authentication** | **Google OAuth 2.0 (OpenID Connect)** | Verified against Google JWKS; issues 15-minute JWT access tokens and 30-day rotating refresh tokens with reuse detection. |
| **Financial Settlement** | **Physical Cash Settlement** | Cash collected upon drop-off; decoupled from ride completion; uncollected ₹30 fees surcharged on subsequent rides. |

---

## 3. Authoritative Documentation Index

This repository contains an exhaustive, implementation-ready architectural documentation suite designed to serve as the single source of truth for engineering teams and AI coding agents:

| Document | Primary Domain and Specifications Covered |
|---|---|
| [**AGENTS.md**](./AGENTS.md) | **Mandatory AI Agent Directives**, architectural guardrails, database invariants, and non-negotiable coding rules. |
| [**docs/PROJECT_OVERVIEW.md**](./docs/PROJECT_OVERVIEW.md) | **Project Overview & PRD:** Problem statement, user personas, journey maps, approved business rules, functional and non-functional requirements. |
| [**docs/HIGH_LEVEL_DESIGN.md**](./docs/HIGH_LEVEL_DESIGN.md) | **High-Level Design (HLD):** C4 Level 1–3 architecture diagrams, end-to-end data flows, container models, and network topology. |
| [**docs/SYSTEM_ARCHITECTURE.md**](./docs/SYSTEM_ARCHITECTURE.md) | **Detailed Software Architecture:** 16 domain module deep dives, dual process topology, transactional outbox pattern, and failure handling matrix. |
| [**docs/DATABASE_DESIGN.md**](./docs/DATABASE_DESIGN.md) | **Relational Database Design & ERD:** Complete entity specifications, 5 focused Mermaid ERDs, data dictionary, and 11+ exact raw SQL invariants. |
| [**docs/API_SPECIFICATION.md**](./docs/API_SPECIFICATION.md) | **REST & Realtime API Contract:** Base path `/api/v1`, 25 endpoint areas, DTO envelopes, error catalogue, and idempotency headers. |
| [**docs/MATCHING_AND_ROUTING.md**](./docs/MATCHING_AND_ROUTING.md) | **Matching & Permutation Solver:** Greedy candidate clustering, OSRM matrix calls, $k! \le 24$ drop-off optimizer, 25% detour filter, and 1-to-1 driver dispatch. |
| [**docs/FARE_AND_CANCELLATION.md**](./docs/FARE_AND_CANCELLATION.md) | **Fare Engine & Cancellation Ledger:** Tariff (₹20 base, ₹10/km), proportional allocation formula, deterministic whole-rupee rounding, waiting schedule, and ₹30 late fee recovery. |
| [**docs/STATE_MACHINES.md**](./docs/STATE_MACHINES.md) | **State Machines & Transitions:** Formal transition matrices, allowed/forbidden paths, and Mermaid diagrams for all 8 core entities. |
| [**docs/REALTIME_SOCKET_DESIGN.md**](./docs/REALTIME_SOCKET_DESIGN.md) | **Realtime Socket.IO Gateway:** Single-use 60s socket tickets, room architecture, event catalogue, 3–5s GPS streaming, and reconnect synchronization. |
| [**docs/AUTHENTICATION_AND_SECURITY.md**](./docs/AUTHENTICATION_AND_SECURITY.md) | **Authentication & Session Security:** Google OAuth JWKS verification, 15m JWT access tokens, 30d rotating refresh tokens, token family reuse detection, and zero-trust RBAC. |
| [**docs/FAILURE_HANDLING_AND_RECOVERY.md**](./docs/FAILURE_HANDLING_AND_RECOVERY.md) | **Incident & Disaster Recovery:** Mid-trip breakdown handling, replacement dispatch, 10% resumption discount, ₹0 abortion, and 10 failure edge cases. |
| [**docs/DEPLOYMENT_AND_ENVIRONMENT.md**](./docs/DEPLOYMENT_AND_ENVIRONMENT.md) | **Deployment & Demonstration Runbook:** Apple Silicon Mac host setup, iOS and Android networking aliases, environment contracts, and disaster recovery. |
| [**docs/DEMO_SIMULATION_ARCHITECTURE.md**](./docs/DEMO_SIMULATION_ARCHITECTURE.md) | **Demo Simulation Architecture:** Interactive Next.js Control Center, virtual passenger/driver agent runners, GPS route playback, and disruption injections. |
| [**docs/ARCHITECTURE_DECISION_RECORDS.md**](./docs/ARCHITECTURE_DECISION_RECORDS.md) | **Master ADR Register:** Comprehensive log of 31 approved architectural decisions, rejected alternatives, trade-offs, and pending specifications. |
| [**docs/IMPLEMENTATION_ROADMAP.md**](./docs/IMPLEMENTATION_ROADMAP.md) | **Implementation Roadmap & Gates:** 13-stage implementation plan with explicit dependency gates and mandatory Stage 9 lock. |

---

## 4. Repository Structure

```text
.
├── AGENTS.md                          # Mandatory AI coding agent instructions and directives
├── README.md                          # Project overview and master documentation index
│
├── docs/                              # Authoritative system design specifications
│   ├── PROJECT_OVERVIEW.md            # Vision, personas, user journeys, and PRD requirements
│   ├── HIGH_LEVEL_DESIGN.md           # C4 Level 1-3 diagrams, containers, and data flows
│   ├── SYSTEM_ARCHITECTURE.md         # Modular monolith deep dive, outbox, and locking
│   ├── DATABASE_DESIGN.md             # PostgreSQL schema, 5 Mermaid ERDs, and SQL invariants
│   ├── API_SPECIFICATION.md           # REST API endpoints (/api/v1) and DTO specifications
│   ├── MATCHING_AND_ROUTING.md        # Greedy clustering, OSRM matrices, and 24-permutation solver
│   ├── FARE_AND_CANCELLATION.md       # Tariff, proportional fare allocation, and late fee ledger
│   ├── STATE_MACHINES.md              # 8 entity state machines, transition matrices, and diagrams
│   ├── REALTIME_SOCKET_DESIGN.md      # Socket.IO gateway (/realtime), tickets, and GPS telemetry
│   ├── AUTHENTICATION_AND_SECURITY.md # Google OAuth JWKS, JWT rotation, and zero-trust RBAC
│   ├── FAILURE_HANDLING_AND_RECOVERY.md # Breakdown handling, replacement dispatch, and failure matrix
│   ├── DEPLOYMENT_AND_ENVIRONMENT.md  # Local Mac demonstration runbook and network addressing
│   ├── DEMO_SIMULATION_ARCHITECTURE.md # Simulation engine, virtual agents, and Demo Control Center
│   ├── ARCHITECTURE_DECISION_RECORDS.md # Master ADR log (31 approved decisions + open items)
│   └── IMPLEMENTATION_ROADMAP.md      # 13 implementation stages and verification gates
│
└── apps/                              # Implementation source trees (Deferred to Stage 1)
    ├── backend/                       # NestJS Modular Monolith (API & Worker entrypoints)
    │   ├── prisma/                    # Schema, migrations, and PostGIS seed
    │   └── src/                       # 16 domain modules and infrastructure
    ├── mobile/                        # Flutter iOS and Android mobile application
    │   └── lib/                       # Passenger and Driver feature flows
    └── admin/                         # Next.js TypeScript administration portal
        └── app/                       # Dashboard, vetting, fee reviews, and simulation
```

---

## 5. Development Setup Prerequisites

### Local Demonstration Hardware and Tools
- **Host Machine:** Apple Silicon Mac (macOS Sonoma / Sequoia).
- **Client Devices:** Physical iPhone and/or Android smartphones connected to the same Wi-Fi router.
- **System Binaries & Tools:**
  - Xcode 15+ and Command Line Tools.
  - Flutter SDK 3.22+ and Dart 3.4+.
  - Node.js 20+ (LTS) and npm.
  - PostgreSQL 16+ with PostGIS 3.4+ (`brew install postgresql@16 postgis`).
  - Redis 7.2+ (`brew install redis`).
  - OSRM Backend (`brew install osrm-backend`).

### Project Status and Working Rules
- **Current Phase:** Architecture Consolidation, System Design, and Technical Documentation **100% COMPLETE**.
- **Next Phase:** Stage 1 Implementation (Database initialization, Prisma migrations, and baseline service bootstrapping).
- **Rule:** **No application coding or scaffolding is to begin without strictly adhering to this documentation pack.**

---

*On-Demand Aggregation Platform for Last-Mile Connectivity Using Shared Micro-Mobility*
