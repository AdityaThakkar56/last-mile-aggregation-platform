# REST and Realtime API Specification

## On-Demand Aggregation Platform for Last-Mile Connectivity Using Shared Micro-Mobility

---

## 1. Global API Conventions

### 1.1 Transport and Formatting
- **Base Protocol:** HTTPS for REST, WSS for WebSockets.
- **Base Path:** `/api/v1`
- **Payload Format:** Strict JSON (`application/json; charset=utf-8`).
- **Identifiers:** Canonical UUIDv4 strings.
- **Timestamps:** ISO 8601 UTC strings with millisecond precision (`YYYY-MM-DDTHH:mm:ss.sssZ`).
- **Spatial Data:** WGS 84 (SRID 4326) coordinates specified as `{ "latitude": float, "longitude": float }`.
- **Currency Values:** Decimal strings or numbers formatted to 2 decimal places (e.g., `45.00` representing Indian Rupees ₹).

### 1.2 Authentication and Authorization Headers
All secured endpoints require an HTTP `Authorization` header containing an application-issued JWT Bearer token:
```http
Authorization: Bearer <jwt-access-token>
```
Mutating operations sensitive to network retransmissions accept an optional or mandatory `Idempotency-Key` header:
```http
Idempotency-Key: <unique-uuid-or-client-nonce>
```

### 1.3 Unified Response Envelopes
Every REST endpoint returns a consistent JSON envelope.

**Success Envelope (HTTP 200 / 201):**
```json
{
  "success": true,
  "statusCode": 200,
  "message": "Operation completed successfully",
  "data": {},
  "timestamp": "2026-11-13T12:00:00.000Z"
}
```

**Standard Error Envelope (HTTP 4xx / 5xx):**
```json
{
  "success": false,
  "statusCode": 409,
  "errorCode": "ACTIVE_BOOKING_EXISTS",
  "message": "Passenger already possesses an active booking. Please complete or cancel the current booking before requesting a new one.",
  "timestamp": "2026-11-13T12:00:00.000Z",
  "path": "/api/v1/bookings"
}
```

---

## 2. Authentication and Identity Endpoints

### 2.1 `POST /auth/google`
- **Purpose:** Exchanges a Google OAuth ID token for application-specific JWT access and refresh tokens.
- **Access:** Public.
- **Request Headers:** `Content-Type: application/json`
- **Request DTO:**
  ```json
  {
    "idToken": "eyJhbGciOiJSUzI1NiIsImtpZCI6...",
    "selectedMode": "PASSENGER"
  }
  ```
- **Response DTO (HTTP 200 OK):**
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Authentication successful",
    "data": {
      "user": {
        "id": "c1f72b6e-44d3-4a67-b8a9-923cb7c15291",
        "email": "aarav.patel@pdeu.ac.in",
        "fullName": "Aarav Patel",
        "avatarUrl": "https://lh3.googleusercontent.com/...",
        "role": "PASSENGER"
      },
      "driverProfile": null,
      "tokens": {
        "accessToken": "eyJhbGciOiJIUzI1Ni...",
        "refreshToken": "7f8b9a2c3d4e5f...",
        "expiresIn": 900
      }
    },
    "timestamp": "2026-11-13T12:00:00.000Z"
  }
  ```
- **Validation Rules:** `idToken` must be a non-empty string; `selectedMode` must be either `PASSENGER` or `DRIVER`.
- **Database Changes:** `users` upserted based on verified Google `sub`; new row inserted into `refresh_tokens`.

---

### 2.2 `POST /auth/refresh`
- **Purpose:** Rotates refresh tokens and issues a fresh 15-minute access token.
- **Access:** Public (Requires valid refresh token).
- **Request DTO:**
  ```json
  {
    "refreshToken": "7f8b9a2c3d4e5f..."
  }
  ```
- **Response DTO (HTTP 200 OK):**
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Token refreshed successfully",
    "data": {
      "accessToken": "eyJhbGciOiJIUzI1Ni...",
      "refreshToken": "a1b2c3d4e5f6...",
      "expiresIn": 900
    },
    "timestamp": "2026-11-13T12:15:00.000Z"
  }
  ```
- **Database Changes:** Prior refresh token row deleted or marked revoked; new refresh token hash inserted.

---

### 2.3 `POST /auth/logout`
- **Purpose:** Terminates the current active session.
- **Access:** Authenticated (`PASSENGER`, `DRIVER`, `ADMIN`).
- **Request DTO:**
  ```json
  {
    "refreshToken": "7f8b9a2c3d4e5f..."
  }
  ```
- **Response DTO (HTTP 200 OK):**
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Logged out successfully",
    "data": null,
    "timestamp": "2026-11-13T12:30:00.000Z"
  }
  ```
- **Database Changes:** Refresh token row deleted from `refresh_tokens`.

---

## 3. Passenger Booking Endpoints

### 3.1 `POST /bookings`
- **Purpose:** Creates a new on-demand ride request with upfront grouping consent.
- **Access:** Authenticated (`PASSENGER`).
- **Headers:** `Idempotency-Key` (Mandatory UUID).
- **Request DTO:**
  ```json
  {
    "pickupZoneId": "d8e2b1c4-55a1-4f33-8a21-9921cb44a102",
    "currentLocation": {
      "latitude": 23.15582,
      "longitude": 72.66691
    },
    "destination": {
      "latitude": 23.18940,
      "longitude": 72.62830
    },
    "requestedType": "SHARED",
    "consentToGrouping": true
  }
  ```
- **Response DTO (HTTP 201 Created):**
  ```json
  {
    "success": true,
    "statusCode": 201,
    "message": "Booking requested successfully. Enqueued for batch matching.",
    "data": {
      "id": "e4b1a8d2-33c9-4b77-9811-aa42b109cc81",
      "passengerId": "c1f72b6e-44d3-4a67-b8a9-923cb7c15291",
      "pickupZoneId": "d8e2b1c4-55a1-4f33-8a21-9921cb44a102",
      "requestedType": "SHARED",
      "status": "REQUESTED",
      "estimatedSoloFare": 80.00,
      "maxAllowedDetourPercent": 25.0,
      "maxWaitDeadline": "2026-11-13T12:02:00.000Z",
      "outstandingCancellationFee": 0.00,
      "createdAt": "2026-11-13T12:00:00.000Z"
    },
    "timestamp": "2026-11-13T12:00:00.000Z"
  }
  ```
- **Validation & Business Errors:**
  - `409 Conflict (ACTIVE_BOOKING_EXISTS)`: Passenger already has an active booking.
  - `422 Unprocessable Entity (OUTSIDE_PICKUP_GEOFENCE)`: Current GPS fix is $> 150\text{m}$ from pickup hub.
  - `422 Unprocessable Entity (DESTINATION_OUTSIDE_SERVICE_AREA)`: Destination is outside Gandhinagar service polygon.
  - `400 Bad Request (CONSENT_REQUIRED)`: `consentToGrouping` must be true for `SHARED` requests.
- **Database & Job Effects:** Inserts row into `bookings`; inserts `BOOKING_CREATED` event into `outbox_events`.

---

### 3.2 `GET /bookings/active`
- **Purpose:** Fetches current passenger booking state, matched group status, and assigned driver details for UI re-synchronization.
- **Access:** Authenticated (`PASSENGER`).
- **Response DTO (HTTP 200 OK):**
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Active booking retrieved",
    "data": {
      "booking": {
        "id": "e4b1a8d2-33c9-4b77-9811-aa42b109cc81",
        "status": "CONFIRMED",
        "requestedType": "SHARED",
        "allocatedFare": 45.00,
        "detourPercent": 14.5
      },
      "ride": {
        "id": "f5a2c1b9-88e3-4d22-9933-bb11c208dd72",
        "status": "DRIVER_ARRIVING",
        "vehiclePlate": "GJ-01-ER-4412",
        "vehicleModel": "Mahindra Treo Auto",
        "driver": {
          "id": "b2c3d4e5-66f7-4a88-8822-cc33d441ee99",
          "fullName": "Ramesh Kumar",
          "phoneNumber": "+919876543210"
        },
        "etaPickupSeconds": 180
      }
    },
    "timestamp": "2026-11-13T12:03:00.000Z"
  }
  ```

---

### 3.3 `POST /bookings/:id/cancel`
- **Purpose:** Cancels an active booking, enforcing state-dependent cancellation rules and late fee assessments.
- **Access:** Authenticated (Owning `PASSENGER`).
- **Request DTO:**
  ```json
  {
    "reason": "Change of plans; taking alternative transport"
  }
  ```
- **Response DTO (HTTP 200 OK):**
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Booking cancelled successfully",
    "data": {
      "bookingId": "e4b1a8d2-33c9-4b77-9811-aa42b109cc81",
      "status": "CANCELLED",
      "lateCancellationFeeIncurred": 30.00,
      "feeStatus": "OUTSTANDING",
      "remainingGroupDissolved": false,
      "cancelledAt": "2026-11-13T12:04:15.000Z"
    },
    "timestamp": "2026-11-13T12:04:15.000Z"
  }
  ```
- **Database & Job Effects:** Updates `bookings.status = 'CANCELLED'`; if driver was assigned, inserts ₹30 record into `cancellation_fees`; recalculates remaining group route; triggers outbox event `PASSENGER_CANCELLED_GROUP_MEMBER`.

---

### 3.4 `GET /passengers/cancellation-fees/outstanding`
- **Purpose:** Checks whether the passenger has an outstanding late cancellation fee that will be surcharged on their next ride.
- **Access:** Authenticated (`PASSENGER`).
- **Response DTO (HTTP 200 OK):**
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Outstanding fees query completed",
    "data": {
      "hasOutstandingFee": true,
      "totalOutstandingAmount": 30.00,
      "fees": [
        {
          "feeId": "a9b8c7d6-11e2-4f33-99aa-8811bb22cc33",
          "originalBookingId": "e4b1a8d2-33c9-4b77-9811-aa42b109cc81",
          "amount": 30.00,
          "createdAt": "2026-11-13T12:04:15.000Z"
        }
      ]
    },
    "timestamp": "2026-11-13T12:05:00.000Z"
  }
  ```

---

## 4. Driver Onboarding and Dispatch Endpoints

### 4.1 `POST /drivers/register`
- **Purpose:** Submits an initial driver partner application with vehicle metadata.
- **Access:** Authenticated (`PASSENGER` upgrading to `DRIVER`).
- **Request DTO:**
  ```json
  {
    "licenseNumber": "GJ0120230004921",
    "vehicle": {
      "registrationNumber": "GJ-01-ER-4412",
      "vehicleType": "E_RICKSHAW",
      "passengerCapacity": 4,
      "makeModel": "Mahindra Treo Auto"
    }
  }
  ```
- **Response DTO (HTTP 201 Created):**
  ```json
  {
    "success": true,
    "statusCode": 201,
    "message": "Driver application submitted. Please upload verification documents.",
    "data": {
      "driverProfileId": "b2c3d4e5-66f7-4a88-8822-cc33d441ee99",
      "verificationStatus": "PENDING"
    },
    "timestamp": "2026-11-13T12:00:00.000Z"
  }
  ```

---

### 4.2 `POST /drivers/me/documents`
- **Purpose:** Submits document references (Driving License, Vehicle RC) for administrative verification.
- **Access:** Authenticated (`DRIVER`).
- **Request DTO:**
  ```json
  {
    "documentType": "DRIVING_LICENSE",
    "documentUrl": "https://storage.local/documents/dl_gj012023.jpg"
  }
  ```
- **Response DTO (HTTP 201 Created):**
  ```json
  {
    "success": true,
    "statusCode": 201,
    "message": "Document uploaded successfully for review",
    "data": {
      "documentId": "11a2b3c4-d5e6-4f77-8899-001122334455",
      "documentType": "DRIVING_LICENSE",
      "status": "PENDING"
    },
    "timestamp": "2026-11-13T12:05:00.000Z"
  }
  ```

---

### 4.3 `PATCH /drivers/me/availability`
- **Purpose:** Toggles driver online/offline availability.
- **Access:** Approved `DRIVER`.
- **Request DTO:**
  ```json
  {
    "isOnline": true
  }
  ```
- **Response DTO (HTTP 200 OK):**
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Driver operational status updated",
    "data": {
      "isOnline": true,
      "operationalStatus": "AVAILABLE"
    },
    "timestamp": "2026-11-13T12:10:00.000Z"
  }
  ```
- **Errors:** `403 Forbidden (DRIVER_NOT_APPROVED)`: If `verification_status != 'APPROVED'`.

---

### 4.4 `GET /drivers/offers/pending`
- **Purpose:** Retrieves all pending dispatch offers currently available to this driver.
- **Access:** Online, Available `DRIVER`.
- **Response DTO (HTTP 200 OK):**
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Pending offers retrieved",
    "data": [
      {
        "offerId": "99e8d7c6-b5a4-4f33-8822-1100aa22bb33",
        "rideGroupId": "f5a2c1b9-88e3-4d22-9933-bb11c208dd72",
        "passengerCount": 3,
        "totalSharedFare": 140.00,
        "estimatedDistanceMeters": 4200,
        "estimatedDurationSeconds": 680,
        "expiresAt": "2026-11-13T12:12:20.000Z",
        "secondsRemaining": 17
      }
    ],
    "timestamp": "2026-11-13T12:12:03.000Z"
  }
  ```

---

### 4.5 `POST /drivers/offers/:id/accept`
- **Purpose:** Executes the critical atomic driver acceptance transaction.
- **Access:** Online, Available `DRIVER`.
- **Response DTO (HTTP 200 OK):**
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Offer accepted. Driver successfully assigned to ride.",
    "data": {
      "rideId": "f5a2c1b9-88e3-4d22-9933-bb11c208dd72",
      "pickupZone": "PDEU Main Transit Zone",
      "passengerCount": 3,
      "totalFare": 140.00,
      "routeGeometry": {
        "type": "LineString",
        "coordinates": [[72.6669, 23.1558], [72.6450, 23.1720], [72.6283, 23.1894]]
      }
    },
    "timestamp": "2026-11-13T12:12:05.000Z"
  }
  ```
- **Concurrency & Race Errors:**
  - `409 Conflict (OFFER_ALREADY_EXPIRED)`: Offer timed out past 20 seconds.
  - `409 Conflict (GROUP_ALREADY_ASSIGNED)`: Another driver won the race.
  - `409 Conflict (DRIVER_ALREADY_RESERVED)`: Driver accepted a different offer simultaneously.
- **Database Transaction Actions:** Locks `ride_groups` row; conditionally updates `driver_profiles.operational_status = 'RESERVED'`; inserts `ride_assignments`; marks offer `ACCEPTED`; withdraws all other pending offers for this driver; marks competing offers for this group `WITHDRAWN`; enqueues `RIDE_CONFIRMED` outbox event.

---

### 4.6 `POST /drivers/offers/:id/reject`
- **Purpose:** Explicitly declines a dispatch offer, keeping the group intact and excluding this driver from this group version.
- **Access:** Assigned `DRIVER`.
- **Response DTO (HTTP 200 OK):**
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Offer rejected. Group queued for next eligible driver.",
    "data": {
      "offerId": "99e8d7c6-b5a4-4f33-8822-1100aa22bb33",
      "status": "REJECTED"
    },
    "timestamp": "2026-11-13T12:12:08.000Z"
  }
  ```
- **Database & Job Effects:** Updates `driver_dispatch_offers.status = 'REJECTED'`; enqueues immediate BullMQ job to offer group to the next ranked driver.

---

## 5. Trip Execution and Cash Payment Endpoints

### 5.1 `POST /rides/:id/arrived`
- **Purpose:** Driver signals arrival at the PDEU pickup hub.
- **Access:** Assigned `DRIVER`.
- **Response DTO (HTTP 200 OK):**
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Driver arrival confirmed. Passengers notified.",
    "data": { "status": "DRIVER_ARRIVED" },
    "timestamp": "2026-11-13T12:15:00.000Z"
  }
  ```

---

### 5.2 `POST /rides/:id/start`
- **Purpose:** Driver starts the multi-stop trip after all passengers board.
- **Access:** Assigned `DRIVER`.
- **Response DTO (HTTP 200 OK):**
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Trip started. High-frequency GPS streaming active.",
    "data": { "status": "IN_PROGRESS" },
    "timestamp": "2026-11-13T12:17:00.000Z"
  }
  ```

---

### 5.3 `POST /rides/:id/complete`
- **Purpose:** Driver completes all passenger drop-offs.
- **Access:** Assigned `DRIVER`.
- **Response DTO (HTTP 200 OK):**
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Ride completed. Cash collection pending.",
    "data": {
      "status": "COMPLETED",
      "fareBreakdown": [
        {
          "bookingId": "e4b1a8d2-33c9-4b77-9811-aa42b109cc81",
          "passengerName": "Aarav Patel",
          "rideFare": 45.00,
          "outstandingFee": 30.00,
          "totalCashCollectable": 75.00
        }
      ]
    },
    "timestamp": "2026-11-13T12:35:00.000Z"
  }
  ```

---

### 5.4 `POST /payments/:id/confirm-cash`
- **Purpose:** Driver confirms physical receipt of cash from a passenger, settling ride fare and recovering any past late fee.
- **Access:** Assigned `DRIVER`.
- **Request DTO:**
  ```json
  {
    "bookingId": "e4b1a8d2-33c9-4b77-9811-aa42b109cc81",
    "amountCollected": 75.00
  }
  ```
- **Response DTO (HTTP 200 OK):**
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Cash payment recorded. Past cancellation fee settled.",
    "data": {
      "paymentId": "77a8b9c0-1122-3344-5566-778899aabbcc",
      "status": "PAID",
      "rideFareSettled": 45.00,
      "cancellationFeeSettled": 30.00,
      "totalCollected": 75.00
    },
    "timestamp": "2026-11-13T12:36:00.000Z"
  }
  ```
- **Database Changes:** Inserts `cash_payment_records`; updates `cancellation_fees.status = 'COLLECTED'` atomically; emits `payment:status:changed` Socket.IO event.

---

## 6. Administrator Dashboard Endpoints

### 6.1 `GET /admin/drivers`
- **Purpose:** Lists driver onboarding applications with verification filter.
- **Access:** Authenticated `ADMIN`.
- **Query Parameters:** `status=PENDING|APPROVED|REJECTED|SUSPENDED`, `page=1`, `limit=20`.

### 6.2 `POST /admin/drivers/:id/approve`
- **Purpose:** Approves a driver partner, allowing them to go online and receive offers.
- **Access:** Authenticated `ADMIN`.
- **Response DTO (HTTP 200 OK):**
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Driver partner approved successfully",
    "data": { "verificationStatus": "APPROVED" },
    "timestamp": "2026-11-13T12:00:00.000Z"
  }
  ```

### 6.3 `POST /admin/cancellation-fees/:id/waive`
- **Purpose:** Manually waives a disputed passenger cancellation fee.
- **Access:** Authenticated `ADMIN`.
- **Request DTO:**
  ```json
  {
    "waiverReason": "Verified campus emergency; passenger was hospitalised"
  }
  ```
- **Response DTO (HTTP 200 OK):**
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Cancellation fee waived",
    "data": {
      "feeId": "a9b8c7d6-11e2-4f33-99aa-8811bb22cc33",
      "status": "WAIVED",
      "waivedAt": "2026-11-13T12:00:00.000Z"
    },
    "timestamp": "2026-11-13T12:00:00.000Z"
  }
  ```

---

## 7. Realtime Socket.IO Event Specification

### 7.1 WebSocket Lifecycle & Rooms
- **Handshake Authentication:** The client provides the JWT in `io("http://<HOST>:3000", { auth: { token: "Bearer <token>" } })`. Unauthenticated connections are disconnected with `UnauthorizedException`.
- **Authorized Room Joins:**
  - `user:<userId>`: Automatically joined upon handshake.
  - `driver:<driverId>`: Joined by verified drivers.
  - `ride:<rideId>`: Joined dynamically when driver assignment is confirmed.
  - `admin:monitoring`: Privileged room joined by administrators.

### 7.2 Event Catalog

| Event Name | Direction | Payload Schema Summary | Delivery Expectation |
|---|---|---|---|
| `driver:location:update` | Driver $\rightarrow$ Server | `{ lat: float, lng: float, heading: float, speed: float }` | Emitted every 3–5 seconds; volatile |
| `driver:location:broadcast` | Server $\rightarrow$ Passenger | `{ driverId: UUID, lat: float, lng: float, heading: float }` | Broadcast to `ride:<rideId>`; ephemeral |
| `booking:status:changed` | Server $\rightarrow$ Passenger | `{ bookingId: UUID, status: string, timestamp: string }` | Emitted on batch claim, match, or cancel |
| `driver:offer:received` | Server $\rightarrow$ Driver | `{ offerId: UUID, groupId: UUID, paxCount: int, fare: float, timeoutSec: 20 }` | Direct 1-to-1 dispatch alert with 20s countdown |
| `driver:offer:withdrawn` | Server $\rightarrow$ Driver | `{ offerId: UUID, reason: "ACCEPTED_OTHER_OFFER" \| "GROUP_CLAIMED" }` | Instructs driver UI to dismiss offer card |
| `ride:driver:assigned` | Server $\rightarrow$ Passenger | `{ rideId: UUID, driverName: string, vehiclePlate: string, etaSeconds: int }` | Confirms ride and reveals driver info |
| `ride:status:changed` | Server $\rightarrow$ Both | `{ rideId: UUID, status: "ARRIVED" \| "IN_PROGRESS" \| "COMPLETED" }` | Advances trip UI state machine |
| `payment:status:changed` | Server $\rightarrow$ Passenger | `{ bookingId: UUID, status: "PAID", amountCollected: float }` | Displays final receipt on passenger screen |

---

*End of REST and Realtime API Specification*

