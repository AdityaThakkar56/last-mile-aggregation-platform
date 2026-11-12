# Product Requirements Document (PRD)

## Product summary
The platform groups commuters leaving a common transit pickup point and connects them with shared micro-mobility vehicles (e-rickshaws, bike taxis or shuttle vehicles). It groups compatible destinations, optimizes a multi-stop route, allocates fares and assigns a driver.

Initial pilot: configurable PDEU pickup zone, Gandhinagar. Destinations are coordinates validated against a bounded service region, not a hardcoded list of place names.

## Problem
Separate rides from the same pickup area may travel along overlapping corridors. Pooling compatible requests can improve vehicle utilization and reduce per-passenger cost while respecting waiting time and detour limits.

## Actors
- **Passenger:** Google sign-in, choose passenger mode, select destination, choose shared/private, create/cancel booking, accept/reject group offer, track ride, pay cash.
- **Driver:** Google sign-in, choose driver mode, submit profile/vehicle/documents, await admin approval, go online, accept/reject dispatch, share GPS, operate trip, confirm cash.
- **Admin:** approve/reject/suspend drivers, configure pickup/service zones, monitor bookings/rides/payments, review cancellation strikes.

## MVP scope
Included: Google OAuth, role selection, driver approval, one pickup zone, service polygon, shared/private bookings, matching, route optimization, detour/fare allocation, unanimous offers, driver dispatch, explicit private fallback, GPS, cash payment, admin dashboard, local demo.
Excluded: digital payment gateway, automated refunds/settlement, multiple stations, dynamic pricing, ML pricing, microservices, Kubernetes, CI/CD, production 24/7 hosting.

## Product rules
### Pickup and service region
- Passenger selects named pickup zone.
- Backend validates current GPS against configured geofence.
- Initial 150 m radius is a configurable pilot default and must be site-tested.
- Canonical pickup is configured zone point, not arbitrary client coordinates.
- Destination must be inside configured service polygon.

### Shared/private
- Passenger chooses SHARED or PRIVATE at booking.
- Only SHARED requests enter shared matching.
- Shared failure may produce a separately priced private offer.
- Never silently convert a shared request to private.

### Matching limits (initial configurable defaults)
- 4 paying passengers + 1 driver.
- Group size 2–4.
- Matching window 60 s.
- Maximum shared wait 120 s from original booking creation; rematching does not reset it.
- Offer response timeout 20 s.
- Maximum detour 25% per passenger.
- Max total trip duration threshold is pending.

### Group acceptance and rematching
- Every passenger accepts the same current offer.
- Reject group != cancel request.
- Rejection keeps request eligible subject to retry policy.
- Timeout differs from explicit rejection.
- First timeout: no response; temporarily eligible. Second consecutive timeout: pause and request reconfirmation; expire if not reconfirmed by deadline.
- Rejection policy discussed: first rejection tries different candidates; second asks whether to continue; third stops automatic shared matching and offers private/cancel. Thresholds are configurable and require final confirmation.
- Any changed group, route or fare requires a new offer and fresh acceptance from every remaining passenger.
- One booking can have only one active offer/claim.
- Avoid immediately repeating exact failed group configuration.

### Driver assignment
- Dispatch begins only after unanimous passenger acceptance.
- Ride confirmed only after driver accepts.
- Driver reservation must be atomic.
- Reject/timeout tries next eligible driver until dispatch deadline.
- If no driver, notify and offer private fallback where applicable.

### Cancellation
- Preserve remaining group only if min size, capacity, route, detour and duration constraints remain valid; otherwise rematch.
- After driver assignment but before trip start, apply late-cancellation reliability policy unless driver/platform caused it.
- Proposed strike policy: first warning; second within 30 days = 15-minute shared cooldown; third = 60-minute cooldown; strikes expire after 30 days; admin may reverse. Confirm thresholds before enforcement.

### Payment
Cash only. Passenger pays after ride completion; driver confirms received. Payment confirmation is separate from ride completion.

## Functional acceptance criteria
- Google login creates/retrieves identity.
- Passenger and driver modes work with server-side authorization.
- Unapproved driver cannot go online.
- Pickup/service geospatial checks reject invalid points.
- Concurrent requests cannot create multiple active bookings for one passenger.
- Shared requests group only when all constraints pass.
- Driver dispatch cannot begin before all passengers accept.
- No duplicate active driver assignment.
- Private fallback requires consent.
- Authorized passenger sees driver location and stale indicator.
- Admin can approve drivers and inspect operations.
- Cash payment is auditable.

## Demo success
Two passenger requests form a feasible group; route matrix/order computed; both accept; approved driver accepts; ride is tracked live; driver completes; cash marked received; admin sees completed ride. Demonstrate at least one failure/cancellation scenario.
