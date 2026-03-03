# Campus Driver App — Backend Design

This document describes the **backend architecture and behavior** for the Campus Driver App.

Product-level behavior and UX are in:
- `docs/prd.md`

Frontend details are in:
- `docs/frontend.md`

Api details are in:
- `docs/api.md`


---

## Table of Contents

- [1. Overview](#1-overview)
- [2. Data Model & Schema](#2-data-model--schema)
  - [2.1 High-level ER Diagram](#21-high-level-er-diagram)
  - [2.2 Core Tables](#22-core-tables)
- [3. Security & Authentication](#3-security--authentication)
  - [3.1 Authentication Flow](#31-authentication-flow)
  - [3.2 Driver Verification](#32-driver-verification)
- [4. Real-time Communication Socketio](#4-real-time-communication-socketio)
  - [4.1 Connection](#41-connection)
  - [4.2 Rooms & Access Control](#42-rooms--access-control)
  - [4.3 Chat Flow](#43-chat-flow)
  - [4.4 Notifications Integration](#44-notifications-integration)
  - [4.5 Retention Behavior](#45-retention-behavior)
- [5. Background Jobs & Automation](#5-background-jobs--automation)
  - [5.1 Scheduled Jobs](#51-scheduled-jobs)
- [6. Error Handling & Business Rules](#6-error-handling--business-rules)
  - [6.1 Domain-specific HTTP Codes](#61-domain-specific-http-codes)
  - [6.2 General Errors](#62-general-errors)
- [7. Environment & Configuration](#7-environment--configuration)
  - [7.1 Domain Configuration](#71-domain-configuration)
  - [7.2 Other Configuration](#72-other-configuration)
- [8. Responsibilities Summary](#8-responsibilities-summary)

---

## 1. Overview

- **Runtime:** Node.js
- **Framework:** Express
- **Database:** PostgreSQL (e.g. AWS RDS)
- **Auth:** Firebase Auth (ID tokens verified via Firebase Admin SDK)
- **Realtime:** Socket.io for chat and ride updates
- **Notifications:** Firebase Cloud Messaging (FCM)
- **Storage:** AWS S3 (user uploads, driver license images)

The backend exposes REST and realtime interfaces that implement the product behavior described in `prd.md`.

---

## 2. Data Model & Schema

### 2.1 High-level ER Diagram

Conceptual relationships:

```text
users ─────────────────── rides
│                     │
├── driver_profiles   ├── messages
└── fcm_tokens        └── notifications
```

### 2.2 Core Tables

#### `users`

- Stores personal information and university verification status.
- Linked to Firebase user via UID.

#### `driver_profiles`

- Stores driver-specific information:
  - Vehicle details.
  - `verification_status` (`pending`, `approved`, etc.).
- Linked 1:1 with `users` for users who are drivers.

#### `rides`

- Core entity for ride requests and their lifecycle:
  - Pickup and dropoff coordinates.
  - Timestamps (created, accepted, started, completed).
  - Optional **scheduled pickup time** when created as a scheduled ride.
  - Status:
    - `pending`
    - `accepted`
    - `ongoing`
    - `completed`

#### `messages`

- Stores chat messages for each ride.
- Contains `deleted_at` column to support soft deletion for retention rules.

#### `fcm_tokens`

- Stores FCM device tokens for each user.
- Used to send ride and chat notifications.

---

## 3. Security & Authentication

### 3.1 Authentication Flow

1. Mobile app authenticates against Firebase.
2. Firebase returns a JWT (ID Token).
3. Mobile app calls backend APIs with:

```http
Authorization: Bearer <firebase_id_token>
```

4. Backend uses Firebase Admin SDK to verify token.
5. On success, backend resolves or creates the local `user` record and attaches `user_id` to the request context.

### 3.2 Driver Verification

- Drivers upload license images to a **private S3 bucket**.
- Backend generates **signed URLs** for admin/reviewer access.
- Verification result is stored in `driver_profiles.verification_status`.
- APIs that allow performing driver actions (e.g. ride accept) may enforce that `verification_status = 'approved'` (exact rules configurable).

---

## 4. Real-time Communication (Socket.io)

### 4.1 Connection

- Clients connect to a Socket.io endpoint, authenticating with Firebase ID token.
- During the handshake:
  - Backend verifies the token.
  - Associates the connection with `user_id`.

### 4.2 Rooms & Access Control

- Each ride has a dedicated room:

```text
ride_{ride_id}
```

- Access rule:
  - Only the ride’s `passenger_id` and `driver_id` may join its room.

### 4.3 Chat Flow

1. Client opens ride chat screen and connects to Socket.io.
2. Client joins `ride_{ride_id}` room.
3. When user sends a message:
   - Client emits `message:send` with:
     - `ride_id`
     - sender `user_id`
     - message content & metadata.
4. Backend:
   - Validates membership (user belongs to the ride).
   - Persists message to `messages`.
   - Emits `message:new` event to `ride_{ride_id}` room.

### 4.4 Notifications Integration

- If recipient is not connected to the chat:
  - Backend looks up recipient’s FCM tokens.
  - Sends an FCM notification with:
    - `ride_id`
    - Sender name
    - Short preview
- For ride status changes (created, accepted, cancelled, completed):
  - Backend sends FCM notifications to relevant users.
  - Payloads are designed so the client can deep-link to the correct screen.

### 4.5 Retention Behavior

- A daily job marks messages as deleted (`deleted_at`) for non-active rides.
- Frontend displays a permanent notice that messages are cleared daily.

---

## 5. Background Jobs & Automation

### 5.1 Scheduled Jobs

| Job              | Schedule          | Action                                              |
|------------------|-------------------|-----------------------------------------------------|
| Ride Expiration  | Every 5 minutes   | Cancels `pending` rides older than 2 hours         |
| Auto-Complete    | Every 5 minutes   | Marks `ongoing` rides as `completed` after 3 hours |
| Message Cleanup  | Daily at 00:00    | Soft-deletes chat history for closed rides         |
| Scheduled Rides  | Every 5 minutes   | Activates rides whose scheduled pickup time is due |

Implementation details:

- Timer/cron mechanism (e.g. cronjob, worker, or managed scheduler).
- Operations should be idempotent and safe to retry.

---

## 6. Error Handling & Business Rules

### 6.1 Domain-specific HTTP Codes

- **409 Conflict**
  - When:
    - A driver attempts to accept a ride that another driver has already accepted.
  - Behavior:
    - Return 409 with a code/message the frontend can map to “ride already accepted”.

- **410 Gone**
  - When:
    - A passenger interacts with a ride that has expired.
  - Behavior:
    - Return 410 with a code/message the frontend can map to “ride expired”.

### 6.2 General Errors

- Standard REST semantics:
  - 400 for validation errors.
  - 401 for auth issues (invalid/expired token).
  - 403 for forbidden actions.
  - 404 for not found.
  - 500 for server errors.

---

## 7. Environment & Configuration

### 7.1 Domain Configuration

- Backend uses environment variable:

```text
APP_DOMAIN=campus-driver-app.xmum-codingclub.com
```

- Usage:
  - CORS configuration.
  - Constructing absolute URLs for S3-hosted assets.
  - Token issuer / audience checks if needed.
  - Allowed origins for Socket.io.

### 7.2 Other Configuration

- Database connection string for PostgreSQL.
- S3 bucket names and credentials (through environment / secret manager).
- Firebase Admin credentials.
- FCM configuration for sending notifications.

---

## 8. Responsibilities Summary

The backend is responsible for:

- Enforcing authentication and authorization for all actions.
- Maintaining consistent ride state transitions.
- Preventing double-accept of rides.
- Persisting and serving chat messages within retention rules.
- Emitting realtime events and notifications.
- Running scheduled jobs that enforce time-based business rules.

