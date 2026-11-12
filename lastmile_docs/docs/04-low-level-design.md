# Low-Level Design (LLD)

## Backend module responsibilities
| Module | Responsibility |
|---|---|
| Auth | Google token verification, app JWT, refresh/logout |
| Users | Profiles, roles, account state |
| Drivers | Application, verification state, availability |
| Vehicles | Vehicle/capacity |
| Bookings | Create/read/cancel and lifecycle |
| Matching | Batches, claims, candidate groups, offers |
| Routing | OSRM table/route, timeouts, circuit breaker |
| Fares | Solo estimates, shared allocation, snapshots |
| Dispatch | Driver eligibility, offers, reservation |
| Locations | GPS validation, Redis latest, DB snapshots |
| Payments | Cash payment record/audit |
| Notifications | In-app/push orchestration |
| Admin | Approvals, monitoring, zone configuration, strike review |
| Infrastructure | Prisma, Redis, BullMQ, outbox, config |

## Suggested NestJS layout
```text
apps/backend/src/
  main.ts
  api.module.ts
  worker.main.ts
  worker.module.ts
  common/{guards,decorators,filters,interceptors,pipes,dto}/
  modules/{auth,users,drivers,vehicles,bookings,matching,routing,fares,dispatch,locations,payments,notifications,admin}/
  infrastructure/{database,redis,queues,outbox,telemetry}/
apps/backend/prisma/{schema.prisma,migrations}/
```

## Flutter layout
```text
apps/mobile/lib/
  app/
  core/{api,auth,routing,storage,realtime,location,theme}/
  features/
    auth/
    role_selection/
    passenger/{booking,offers,tracking,history}/
    driver/{onboarding,availability,trip_offers,active_trip,payments}/
```

## Admin layout
```text
apps/admin/
  app/{login,dashboard,drivers,bookings,rides,payments,cancellations,zones}/
  src/{lib,components,features}/
```

## Domain object separation
- Booking = one passenger request.
- Matching batch = bounded processing window.
- Ride group = candidate set and optimized route.
- Group offer = versioned proposal of membership/route/fare.
- Offer response = one passenger decision.
- Driver assignment = driver reservation/assignment.
- Ride = actual operational trip after driver acceptance.
- Payment = independent collection record.

## State separation
Do not mix every lifecycle into one enum. Keep booking, batch, group offer, passenger response, driver, assignment, ride and payment states independent. Candidate statuses:
- Booking: REQUESTED, MATCHING, OFFERED, ACCEPTED, CONFIRMED, IN_PROGRESS, COMPLETED, CANCELLED, EXPIRED, NO_DRIVER_AVAILABLE.
- Batch: PENDING, PROCESSING, COMPLETED, FAILED.
- Offer: PENDING, PARTIALLY_ACCEPTED, ACCEPTED, REJECTED, EXPIRED, SUPERSEDED.
- Response: PENDING, ACCEPTED, REJECTED, TIMED_OUT.
- Driver: OFFLINE, AVAILABLE, RESERVED, ASSIGNED, ON_TRIP, SUSPENDED.
- Ride: CONFIRMED, DRIVER_ARRIVING, IN_PROGRESS, COMPLETED, CANCELLED.
- Payment: PENDING, PAID, FAILED, DISPUTED.
These are candidates; exact persisted values/transition matrix are schema-time work.

## Transaction boundaries
Short transactions for booking+outbox, booking claims, offer+responses, offer acceptance+dispatch outbox, driver reservation+assignment+ride confirmation, cancellation/group changes, payment confirmation+audit. Never hold transaction across OSRM, notifications, GPS or user wait.

## Boundary rules
Modules call services/interfaces rather than mutating each other's tables casually. Durable cross-module events use outbox. Keep business logic out of controllers and UI.
