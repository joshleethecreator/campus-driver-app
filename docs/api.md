# Campus Driver App — API Design

This document describes the primary **HTTP and realtime APIs** used by the mobile frontend.

Product behavior is defined in:
- `docs/prd.md`

Frontend details are in:
- `docs/frontend.md`

Backend architecture details are in:
- `docs/backend.md`

---

## 1. Conventions

- Base URL (configured via environment):

```text
REST: https://${API_BASE_URL}/v1
WS:   wss://${API_BASE_URL}
```

- Authentication:
  - All protected endpoints require:

```http
Authorization: Bearer <firebase_id_token>
```

- Responses:
  - Use JSON for request/response bodies.
  - Use standard HTTP status codes with domain-specific codes where needed (see Errors section).

---

## 2. Authentication & Profile APIs

### 2.1 Register / Link Firebase User

**POST** `/auth/register`

Purpose:
- Link a Firebase user (by UID in the token) to a local `users` record and create initial profile data.

Auth:
- Requires valid Firebase ID token.

Request body (example):

```json
{
  "displayName": "Alice Passenger",
  "role": "passenger"
}
```

Response (example):

```json
{
  "userId": "uuid",
  "role": "passenger",
  "created": true
}
```

### 2.2 Store FCM Token

**POST** `/auth/fcm-token`

Purpose:
- Register or update a device’s FCM token for a logged-in user.

Auth:
- Requires valid Firebase ID token.

Request body (example):

```json
{
  "token": "fcm-token-string",
  "platform": "android"
}
```

Response:

```json
{ "ok": true }
```

---

## 3. Ride & Driver APIs

### 3.1 Create Ride (Immediate or Scheduled)

**POST** `/rides`

Purpose:
- Create a new ride request, either **immediate** or **scheduled**.

Auth:
- Passenger only.

Request body (example):

```json
{
  "pickup":  { "lat": 0, "lng": 0, "label": "Campus Gate A" },
  "dropoff": { "lat": 0, "lng": 0, "label": "Library" },
  "scheduledPickupAt": "2026-02-21T15:30:00Z"
}
```

Notes:
- `scheduledPickupAt` is optional:
  - If omitted or very close to “now”, the ride behaves as an immediate ride.
  - If provided and in the future, the ride is treated as a **scheduled ride** and surfaced to drivers appropriately.

Response (example):

```json
{
  "rideId": "uuid",
  "status": "pending",
  "scheduledPickupAt": "2026-02-21T15:30:00Z"
}
```

### 3.2 Get Available Rides (Driver)

**GET** `/rides/available`

Purpose:
- Return a filtered list of `pending` ride requests visible to the current driver.

Auth:
- Driver only (server side enforces role/verification).

Response (example):

```json
[
  {
    "rideId": "uuid",
    "pickup": { "lat": 0, "lng": 0, "label": "Campus Gate A" },
    "dropoff": { "lat": 0, "lng": 0, "label": "Library" },
    "createdAt": "2026-02-21T12:00:00Z",
    "status": "pending"
  }
]
```

### 3.3 Accept Ride (Driver)

**PATCH** `/rides/{id}/accept`

Purpose:
- Allow a driver to accept a pending ride.
- Enforces that only one driver can accept a ride (row-level locking on the backend).

Auth:
- Driver only, typically verified & approved.

Request:
- No body required beyond path parameter, or may include driver-specific metadata.

Possible responses:

- **200 OK** – ride successfully accepted.
- **409 Conflict** – ride has already been accepted by another driver.

Response body (success example):

```json
{
  "rideId": "uuid",
  "status": "accepted",
  "driverId": "driver-uuid"
}
```

### 3.3 Location Search

**GET** `/location/search`

Query parameters (example):

```text
GET /location/search?q=hostel+block+A
```

Purpose:
- Provide search suggestions for pickup and dropoff locations.
- Proxies Photon Komoot API from the backend to avoid exposing external API details and to handle CORS.

Response (example):

```json
[
  {
    "label": "Hostel Block A, Campus",
    "lat": 0,
    "lng": 0
  }
]
```

---

## 4. Realtime / Socket.io API

### 4.1 Connection

- WS endpoint: `wss://${API_BASE_URL}`
- Client includes Firebase ID token in connection query or headers.
- Backend validates token and associates socket with `userId`.

### 4.2 Rooms

- Per-ride room format:

```text
ride_{ride_id}
```

### 4.3 Events

#### `message:send` (client → server)

Payload (example):

```json
{
  "rideId": "uuid",
  "body": "On my way!",
  "timestamp": "2026-02-21T12:03:00Z"
}
```

Server behavior:
- Validates that the user belongs to the ride.
- Persists message to the `messages` table.
- Emits `message:new` to the `ride_{rideId}` room.

#### `message:new` (server → clients)

Payload (example):

```json
{
  "rideId": "uuid",
  "messageId": "msg-uuid",
  "senderId": "user-uuid",
  "body": "On my way!",
  "timestamp": "2026-02-21T12:03:00Z"
}
```

Usage:
- Frontend subscribes to this event in the ride chat screen to update the message list in real time.

---

## 5. Error Semantics

Some HTTP errors have explicit product meaning:

- **409 Conflict**
  - Returned when a driver tries to accept a ride that has already been accepted.
  - Frontend message: “Ride already accepted” with list refresh.

- **410 Gone**
  - Returned when a user tries to interact with an expired ride.
  - Frontend message: “Ride expired” with option to request a new ride.

Other errors:

- **400 Bad Request** – validation problems or malformed input.
- **401 Unauthorized** – missing or invalid Firebase token.
- **403 Forbidden** – user authenticated but not allowed (e.g. unverified driver).
- **404 Not Found** – resource does not exist (e.g. ride ID invalid).
- **500 Internal Server Error** – unexpected backend issues.

---

## 6. Environment Variables (API-facing)

- `API_BASE_URL`
  - Used by the mobile app to construct REST and WebSocket URLs.
  - Mapped as:

```text
REST: https://${API_BASE_URL}/v1
WS:   wss://${API_BASE_URL}
```

The backend also uses `APP_DOMAIN` and other variables as described in `docs/backend.md` for server-side configuration.

