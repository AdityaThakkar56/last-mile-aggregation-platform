# Deployment and Environment Architecture Specification

## On-Demand Aggregation Platform for Last-Mile Connectivity Using Shared Micro-Mobility

---

## 1. Primary Demonstration Environment (Local Apple Silicon Mac)

The platform is designed to execute completely on an **Apple Silicon Mac host (macOS Sonoma / Sequoia)** running on a shared local Wi-Fi router. This guarantees 100% adherence to the **strict ₹0 budget constraint** while eliminating external cloud dependencies, latency spikes, and evaluation-day internet outages.

```mermaid
flowchart TB
    subgraph HostMac [Apple Silicon Mac - Host Machine]
      subgraph Daemons [Native Local Infrastructure Daemons]
        PG[(PostgreSQL 16 + PostGIS 3.4\nPort 5432 - Bound to 127.0.0.1)]
        RD[(Redis 7.2 Server\nPort 6379 - Bound to 127.0.0.1)]
        OSRM[OSRM Routing Server\nPort 5000 - Bound to 127.0.0.1]
      end

      subgraph BackendServices [NestJS Modular Monolith Runtime]
        API[NestJS API Process\nPort 3000 - Bound to 0.0.0.0]
        WRK[NestJS Background Worker\nInternal Job Runner]
      end

      subgraph AdminPortal [Next.js Runtime]
        Admin[Next.js 14+ Admin Dashboard\nPort 3001 - Bound to 0.0.0.0]
      end
    end

    subgraph LocalWiFi [Shared Local Wi-Fi Router - Subnet e.g. 192.168.1.0/24]
      WIFIRouter((Wi-Fi Access Point))
    end

    subgraph ClientDevices [First-Class Testing & Evaluation Clients]
      iOS_Sim[Xcode iOS Simulator\nConnects to http://localhost:3000]
      iPhone_Phys[Physical iPhone Devices\nConnect to http://192.168.1.50:3000]
      Android_Sim[Android Emulator (Phase 2)\nConnects to http://10.0.2.2:3000]
      Android_Phys[Physical Android Devices (Phase 2)\nConnect to http://192.168.1.50:3000]
      Admin_Browser[Admin Web Browser\nConnects to http://localhost:3001]
    end

    Daemons <--> BackendServices
    BackendServices <--> LocalWiFi
    AdminPortal <--> LocalWiFi

    LocalWiFi <--> iPhone_Phys
    LocalWiFi <--> Android_Phys
    iOS_Sim --> API
    Android_Sim --> API
    Admin_Browser --> Admin
```

---

## 2. Mobile Development and Platform Target Priority

### 2.1 Cross-Platform Mobile Strategy
- **Shared Codebase:** Built using **Flutter / Dart**, providing a single unified mobile codebase for both passenger and driver features.
- **Both Android and iOS are Official Project Targets:** Both operating systems are first-class targets for the platform. Neither platform is an afterthought or deferred.

### 2.2 Phased Development and Tooling Setup Priority
To maximize development velocity during the 4–6 week minor project timeline, client tooling is prioritized as follows:
- **Phase 1 (Current Immediate Priority):**
  - **Primary IDE:** VS Code / Antigravity IDE.
  - **Primary Development & Testing Environment:** **Xcode and the iOS Simulator** (running natively on Apple Silicon).
  - **Physical Device Validation:** Physical iPhone running iOS 17+ connected via Wi-Fi for live GPS and touch validation.
- **Phase 2 (Subsequent Verification Phase):**
  - **Tooling:** Android Studio and the Android Emulator will be installed and configured.
  - **Physical Device Validation:** Physical Android smartphone testing across local Wi-Fi.

---

## 3. Simulator and Device Networking Architecture

Mobile clients require network access to the NestJS API running on the Mac host. Because network addressing varies significantly across simulators and physical hardware, URLs **must be configurable via runtime environment variables** (`API_BASE_URL`), never hardcoded:

```mermaid
flowchart LR
    iOSSim[iOS Simulator] -->|http://localhost:3000| MacHost[Mac Host API]
    AndrSim[Android Emulator] -->|http://10.0.2.2:3000| MacHost
    PhysPhone[Physical iPhone / Android] -->|http://192.168.1.50:3000| MacHost
```

### 3.1 Network Addressing Rules

| Client Environment | Addressing Strategy | Base REST URL | Socket.IO URL | Notes |
|---|---|---|---|---|
| **Xcode iOS Simulator** | Shared host network loopback. | `http://localhost:3000/api/v1` | `ws://localhost:3000/realtime` | Shares host network directly; `localhost` resolves to the Mac. |
| **Android Emulator** | Special emulator host alias. | `http://10.0.2.2:3000/api/v1` | `ws://10.0.2.2:3000/realtime` | Standard Android emulator gateway alias mapping to host `127.0.0.1`. |
| **Physical iPhone / Android**| Shared Wi-Fi local IP. | `http://<MAC_LAN_IP>:3000/api/v1` | `ws://<MAC_LAN_IP>:3000/realtime` | Mac LAN IP (e.g. `192.168.1.50`) assigned by local Wi-Fi router. |
| **Next.js Admin Browser** | Host loopback. | `http://localhost:3000/api/v1` | `ws://localhost:3000/realtime` | Accesses backend on port 3000; web portal runs on 3001. |

### 3.2 Network Security and Binding Guardrails
- **Publicly Exposed on LAN (`0.0.0.0`):**
  - NestJS API (`port 3000`) and Next.js Admin (`port 3001`) must bind to `0.0.0.0` so physical phones on the same Wi-Fi subnet can connect.
  - macOS Application Firewall must permit incoming connections to Node.js.
- **Strictly Localhost Only (`127.0.0.1`):**
  - PostgreSQL (`5432`), Redis (`6379`), and OSRM (`5000`) must strictly bind to `127.0.0.1`.
  - Direct external database connections from mobile clients are blocked at the operating system network boundary.

---

## 4. Environment Variables and Configuration Contract

Configuration is decoupled using environment files (`.env`). Secrets and passwords must **never be committed to Git**. A documented `.env.example` file is maintained in each application root.

### 4.1 Backend Environment Contract (`apps/backend/.env`)
```bash
# Node Environment
NODE_ENV=development
PORT=3000

# Database Configuration (PostgreSQL 16 + PostGIS)
DATABASE_URL="postgresql://lastmile_user:lastmile_secure_pass@127.0.0.1:5432/lastmile_db?schema=public"

# Redis Configuration (BullMQ & GPS Cache)
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=""

# OSRM Routing Engine
OSRM_BACKEND_URL="http://127.0.0.1:5000"

# Authentication & JWT Secrets
JWT_ACCESS_SECRET="dev-access-secret-minimum-32-chars-long!"
JWT_ACCESS_EXPIRY=900
REFRESH_TOKEN_MAX_DAYS=30

# Google OAuth Client IDs
GOOGLE_CLIENT_ID_IOS="your-ios-client-id.apps.googleusercontent.com"
GOOGLE_CLIENT_ID_ANDROID="your-android-client-id.apps.googleusercontent.com"
GOOGLE_CLIENT_ID_WEB="your-web-client-id.apps.googleusercontent.com"

# Matching & Tariff Pilot Defaults
MATCHING_BATCH_INTERVAL_MS=60000
MAX_SHARED_WAIT_DEADLINE_MS=120000
DRIVER_OFFER_TIMEOUT_SEC=20
MAX_DETOUR_PERCENT=25.00
BASE_FARE_PAISE=2000
PER_KM_RATE_PAISE=1000

# Simulation & Demo Mode Flag
DEMO_MODE=true
```

### 4.2 Mobile Environment Contract (`apps/mobile/.env`)
```bash
# Dynamically configurable base URL
API_BASE_URL="http://192.168.1.50:3000/api/v1"
SOCKET_BASE_URL="ws://192.168.1.50:3000/realtime"
APP_ENV="development"
```

---

## 5. Local Demonstration Startup Runbook

To launch the complete system for live demonstration on an Apple Silicon Mac:

### 5.1 Step 1: Start Native Background Services
```bash
# Start PostgreSQL 16 and Redis via Homebrew
brew services start postgresql@16
brew services start redis

# Launch local OSRM routing daemon in background
osrm-routed --algorithm mld ./routing/gandhinagar.osrm --port 5000 &
```

### 5.2 Step 2: Bootstrap Database Schema
```bash
cd apps/backend
npx prisma migrate dev --name init
npm run seed:demo  # Inserts PDEU pickup hub, Gandhinagar boundary, and pilot tariff
```

### 5.3 Step 3: Launch Backend Monolith Processes
```bash
# Terminal 1: Launch NestJS API Process
cd apps/backend
npm run start:api

# Terminal 2: Launch NestJS Worker Process
cd apps/backend
npm run start:worker
```

### 5.4 Step 4: Launch Admin Web Portal
```bash
# Terminal 3: Launch Next.js Admin Dashboard
cd apps/admin
npm run dev
# Opens at http://localhost:3001
```

### 5.5 Step 5: Launch Mobile Clients
```bash
# Terminal 4: Launch Flutter Passenger Mode on iOS Simulator
cd apps/mobile
flutter run -d "iPhone 15 Pro" --dart-define=APP_MODE=PASSENGER

# Terminal 5: Launch Flutter Driver Mode on Physical iPhone
flutter run -d <physical-iphone-device-id> --dart-define=APP_MODE=DRIVER
```

---

## 6. Backup, Recovery and Secrets Management

1. **Database Backups:**
   Automated logical backups are taken via standard PostgreSQL utilities:
   ```bash
   pg_dump -U lastmile_user -Fc lastmile_db > backups/lastmile_$(date +%Y%m%d_%H%M%S).dump
   ```
2. **OSRM Data Reproducibility:**
   The raw OpenStreetMap extract (`gandhinagar.osm.pbf`) and the pre-compiled OSRM graph files are checked into a shared local asset directory with reproducible compilation scripts:
   ```bash
   osrm-extract -p /opt/homebrew/share/osrm/profiles/car.lua gandhinagar.osm.pbf
   osrm-partition gandhinagar.osrm
   osrm-customize gandhinagar.osrm
   ```
3. **Optional Cloud Hosting Evaluation (Post-MVP):**
   If cloud deployment is explored post-evaluation, free-tier hosting must strictly support:
   - Native PostGIS extensions (`postgis`).
   - Dedicated Redis with persistent connection pooling.
   - Persistent WebSocket connections (no 30s HTTP gateway timeouts).
   - Independent background worker processes without idle sleep modes.

---

*End of Deployment and Environment Architecture Specification*
