# REST API Contract

Base path: `/api/v1`. This is the endpoint catalogue; exact DTO schemas, response types, pagination and status codes should be finalized in Swagger during implementation. Example payloads are illustrative.

## Conventions
- JSON; UUID IDs; UTC ISO 8601.
- JWT `Authorization: Bearer`.
- DTO validation, server-side role/ownership guards.
- Pagination for lists/history.
- Idempotency key for booking creation and retry-sensitive commands.
- Standard response envelope:
```json
{"success":true,"data":{},"message":"Operation completed"}
```
Error:
```json
{"success":false,"statusCode":409,"message":"You already have an active booking","code":"ACTIVE_BOOKING_EXISTS"}
```

## Auth/users
| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/auth/google` | Verify Google ID token, create/retrieve user, issue app tokens |
| POST | `/auth/refresh` | Rotate/refresh app tokens |
| POST | `/auth/logout` | Revoke refresh session |
| GET | `/auth/me` | Current user/profile/effective roles |
| PATCH | `/users/me` | Update permitted profile |

Illustrative Google request: `{"idToken":"<google-id-token>","selectedMode":"PASSENGER"}`. Verify signature, issuer, audience, expiry; use Google `sub` as stable identity key. Use app-issued JWT for normal API calls.

## Driver/admin
| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/drivers/register` | Submit driver application |
| GET | `/drivers/application-status` | Check approval |
| GET | `/drivers/me` | Driver profile |
| PATCH | `/drivers/me` | Update allowed fields |
| POST | `/drivers/me/documents` | Submit document references |
| PATCH | `/drivers/me/availability` | Online/offline |
| GET | `/admin/drivers` | List applications |
| GET | `/admin/drivers/:id` | Application detail |
| POST | `/admin/drivers/:id/approve` | Approve |
| POST | `/admin/drivers/:id/reject` | Reject |
| POST | `/admin/drivers/:id/suspend` | Suspend |

## Zones
| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/zones` | List pickup zones |
| GET | `/zones/:id` | Zone details |
| GET | `/service-regions` | Destination regions |
| PATCH | `/admin/zones/:id` | Configure geofence |
| PATCH | `/admin/service-regions/:id` | Configure polygon |

## Passenger bookings
| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/bookings` | Create SHARED/PRIVATE booking |
| GET | `/bookings/active` | Active booking |
| GET | `/bookings/:id` | Detail |
| GET | `/bookings/history` | History |
| POST | `/bookings/:id/cancel` | Cancel |
| POST | `/bookings/:id/continue-sharing` | Reconfirm after timeout |

Illustrative booking:
```json
{
  "pickupZoneId":"<uuid>",
  "destination":{"latitude":23.0225,"longitude":72.5714},
  "requestedType":"SHARED",
  "idempotencyKey":"<unique-key>"
}
```
Backend validates passenger, active booking, pickup GPS geofence, destination polygon and request idempotency. Never trust client fare or canonical pickup.

## Offers
| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/offers/active` | Current offer |
| GET | `/offers/:id` | Offer details |
| POST | `/offers/:id/accept` | Accept exact offer |
| POST | `/offers/:id/reject` | Reject group, remain eligible |

Offer includes version, expiry, stop order, route summary, estimated time, passenger detour and fare. Any changed proposal creates a new offer version and fresh responses.

## Driver dispatch and ride
| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/drivers/offers/active` | Pending driver offer |
| POST | `/drivers/offers/:id/accept` | Accept |
| POST | `/drivers/offers/:id/reject` | Reject |
| GET | `/drivers/trips/active` | Active trip |
| GET | `/drivers/trips/history` | History |
| POST | `/rides/:id/arrived` | Driver arrived |
| POST | `/rides/:id/start` | Start |
| POST | `/rides/:id/complete` | Complete |
| GET | `/rides/:id` | Ride detail |
| GET | `/rides/:id/route` | Route/stops |

## Location
| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/rides/:id/location` | Latest known location |
| GET | `/rides/:id/locations/history` | Authorized history |
Frequent GPS uses Socket.IO.

## Fare/payment
| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/bookings/:id/fare` | Fare breakdown |
| GET | `/payments/:id` | Payment detail |
| POST | `/payments/:id/confirm-cash` | Driver confirms cash |
| GET | `/payments/history` | Passenger history |

## Admin
| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/admin/dashboard` | Metrics |
| GET | `/admin/bookings` | Booking monitor |
| GET | `/admin/rides` | Ride monitor |
| GET | `/admin/payments` | Payment monitor |
| GET | `/admin/cancellations` | Cancellation review |
| POST | `/admin/cancellations/:id/reverse-strike` | Reverse erroneous strike |
| GET | `/admin/matching-batches` | Batch diagnostics |

All admin routes require backend ADMIN authorization. Frontend hiding is not security.
