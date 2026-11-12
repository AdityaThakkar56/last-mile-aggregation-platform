# Glossary

| Term | Meaning |
|---|---|
| Booking | One passenger's request |
| Pickup zone | Configured common pickup point/geofence |
| Service region | Polygon within which destinations are accepted |
| Matching batch | Bounded set of eligible requests processed together |
| Booking claim | Temporary exclusive ownership during matching/offer |
| Ride group | Candidate shared trip with multiple bookings |
| Group offer | Versioned proposal of membership, route, detour and fare |
| Offer response | Passenger accept/reject/timeout |
| Dispatch | Finding and assigning eligible driver |
| Reservation | Temporary atomic hold on driver |
| Assignment | Durable driver-to-ride association |
| Ride | Actual trip after driver acceptance |
| Detour | Extra passenger travel time vs direct trip |
| OSRM Table | Pairwise travel-time/distance matrix |
| OSRM Route | Route geometry and summary |
| PostGIS | PostgreSQL spatial extension |
| BullMQ | Redis-backed Node.js queue |
| Outbox | Durable DB event record for reliable publication |
| Idempotency | Retry does not repeat business effect |
| Lease | Time-limited processing ownership |
| Stale GPS | Latest driver location is older than freshness threshold |
| Private fallback | Separately offered private trip after shared flow fails |
| Late-cancellation strike | Reliability penalty for eligible post-assignment cancellation |
