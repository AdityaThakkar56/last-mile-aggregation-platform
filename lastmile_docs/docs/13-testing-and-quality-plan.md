# Testing and Quality Plan

## Unit tests
Fare allocation/rounding; detour; route permutation; group feasibility; booking/offer transitions; expiry; driver eligibility/ranking; cancellation strikes; DTO validation.

## Integration tests
Google token verification boundary; Prisma constraints; PostGIS geofence/service polygon; booking+outbox transaction; BullMQ idempotency; driver reservation; payment authorization; Socket.IO room authorization.

## Concurrency tests
- Two simultaneous bookings by same passenger.
- Two workers claim same booking.
- Two drivers accept same group.
- One driver accepts two groups concurrently.
- Passenger accepts as offer expires.
- Passenger cancels while batch is processing.
- Duplicate outbox publication.
- Out-of-order GPS events.

## End-to-end tests
- Shared booking → offer → unanimous accept → driver assignment → trip → cash payment.
- Private booking.
- Shared failure → explicit private fallback.
- Driver application → admin approval → online.
- Passenger cancellation before/after driver assignment.
- Driver reject/timeout → next driver.
- Reconnect and REST state recovery.

## Invariants
1. At most one active booking per passenger.
2. At most one active claim per booking.
3. Group has 2–4 paying passengers.
4. Every passenger accepts current offer version.
5. Changed offer requires fresh responses.
6. Ride not confirmed before driver acceptance.
7. At most one active assignment per driver and ride.
8. Only approved driver can accept.
9. Private fallback requires consent.
10. Fare allocations sum to group total after rounding policy is defined.
11. Only assigned driver can confirm cash.
12. Payment pending does not block ride completion.
13. Old GPS cannot overwrite new GPS.
14. Redis outage does not lose committed outbox event.
15. OSRM outage does not fabricate route/fare.

## Manual QA
Location permission denied; outside pickup radius; destination outside polygon; route not found; weak Wi-Fi/reconnect; app background/foreground; expired offer; driver suspension; duplicate taps; non-admin admin API; iPhone using Mac LAN IP.
