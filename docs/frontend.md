# Campus Driver App — Frontend (Flutter) Design

This document describes the **frontend architecture and implementation approach** for the Campus Driver App mobile client.

High-level product behavior and UX are defined in:
- `docs/prd.md`

Backend and API details are in:
- `docs/backend.md`
- `docs/api.md`

---

## Table of Contents

- [1. Overview](#1-overview)
- [2. Architecture & State Management](#2-architecture--state-management)
  - [2.1 State Management](#21-state-management)
  - [2.2 Suggested Project Structure](#22-suggested-project-structure)
- [3. Screens & Navigation (Frontend View)](#3-screens--navigation-frontend-view)
  - [3.1 Authentication](#31-authentication)
  - [3.2 Passenger Flows](#32-passenger-flows)
  - [3.3 Driver Flows](#33-driver-flows)
  - [3.4 Navigation & Deep-links](#34-navigation--deep-links)
- [4. UI Requirements & UX Constraints (Frontend Responsibilities)](#4-ui-requirements--ux-constraints-frontend-responsibilities)
  - [4.1 Chat View Retention Notice](#41-chat-view-retention-notice)
  - [4.2 Offline / Slow Network Behavior](#42-offline--slow-network-behavior)
  - [4.3 Time-based Automation Feedback](#43-time-based-automation-feedback)
- [5. Integration with Backend & API](#5-integration-with-backend--api)
  - [5.1 REST API Usage](#51-rest-api-usage)
  - [5.2 WebSockets / Socketio](#52-websockets--socketio)
  - [5.3 Push Notifications FCM](#53-push-notifications-fcm)
- [6. Testing Strategy (Frontend)](#6-testing-strategy-frontend)

---

## 1. Overview

- **Platform:** iOS and Android.
- **Framework:** Flutter.
- **Primary responsibilities:**
  - Render all user-facing flows described in `prd.md`.
  - Integrate with:
    - REST API (`https://${API_BASE_URL}/v1`)
    - WebSocket / Socket.io (`wss://${API_BASE_URL}`)
    - Firebase Auth and FCM (notifications).

---

## 2. Architecture & State Management

### 2.1 State Management

State is managed with **Riverpod**.

Key providers:

- `authProvider`
  - Holds authentication state and current user profile.
  - Handles token refresh and storage of the last-known profile.
- `rideStreamProvider`
  - Exposes real-time ride status updates through Socket.io.
  - Drives UI for active ride state and transitions.

Additional feature-specific providers (to be implemented as features grow):

- Available rides list (driver).
- Individual ride details.
- Chat message list per ride.
- Driver profile and verification state.

### 2.2 Suggested Project Structure

Under `mobile_app/lib/`:

- `lib/main.dart`
  - App entry point.
  - Initializes providers, theme, and routing.

- Suggested structure as the app grows:
  - `lib/features/`
    - Feature modules (e.g. `auth`, `rides`, `chat`, `driver`).
  - `lib/core/`
    - Shared services:
      - REST API client.
      - Socket.io client wrapper.
      - Environment/config loader.
    - Common models and utilities.
  - `lib/shared/`
    - Reusable widgets (buttons, layout components, etc.).

---

## 3. Screens & Navigation (Frontend View)

The frontend implements the screens and flows defined in the PRD.

### 3.1 Authentication

- Login screen:
  - Integrates with Firebase Auth.
  - On success, fetches/stores user profile via `authProvider`.
- Registration / onboarding:
  - Collects minimum info and possibly driver-specific fields (TBD).

### 3.2 Passenger Flows

- Request Ride:
  - Form with pickup and dropoff fields.
  - Uses location search (see below).
- Location Search:
  - Text input with suggestions from backend-proxied Photon Komoot API.
  - Handles loading / empty / error states.
- Ride Details / Status:
  - Visual representation of ride states (pending/accepted/ongoing/completed).
  - Access to Ride Chat.
- Ride Chat:
  - Per-ride chat UI (message list, input).
  - Shows persistent retention notice (see UI requirements).

### 3.3 Driver Flows

- Driver Onboarding / Verification State:
  - Shows verification status from backend (`pending`, `approved`).
  - Entry point to upload verification documents (handled via backend endpoints).
- Available Rides:
  - List view populated from `/rides/available`.
  - Pull-to-refresh or equivalent.
- Ride Details + Accept:
  - Shows ride-specific info.
  - Accept button:
    - On conflict (409), shows an error message and refreshes the list.
- Active Ride:
  - Shows current ride details and state, leveraging `rideStreamProvider`.
- Ride Chat:
  - Same as passenger view but from driver perspective.

### 3.4 Navigation & Deep-links

- The app must support deep-links from notifications:
  - To Ride Details screen.
  - To Ride Chat screen.
- Implementation can use Navigator 2.0 or a routing package (e.g. `go_router`), but must:
  - Handle routes for each key screen.
  - Accept parameters like `ride_id` when entering from a notification.

---

## 4. UI Requirements & UX Constraints (Frontend Responsibilities)

### 4.1 Chat View Retention Notice

- Every ride chat screen must display the notice:

```text
ⓘ Messages will be cleared at end of day.
```

- The notice is always visible while viewing a chat, regardless of theme or state.

### 4.2 Offline / Slow Network Behavior

- Use `flutter_secure_storage` (or similar) to cache:
  - Last-known user profile.
- App behavior:
  - On startup, show cached profile quickly while fetching fresh data in the background.
  - Indicate offline/limited connectivity where appropriate.

### 4.3 Time-based Automation Feedback

Although automation is performed on the backend, the frontend must:

- React to state changes (e.g. pending → expired, ongoing → completed).
- Clearly communicate if:
  - A ride “disappeared” from lists because it expired.
  - A ride moved to completed due to time auto-complete.

Implementation:

- Subscribe to updated ride status via `rideStreamProvider`.
- When status changes unexpectedly (from user’s perspective), show a short explanation (copy defined in PRD).

---

## 5. Integration with Backend & API

### 5.1 REST API Usage

- Base URL:

```text
https://${API_BASE_URL}/v1
```

- Frontend is responsible for:
  - Attaching auth headers (`Authorization: Bearer <firebase_id_token>`).
  - Handling common errors:
    - 409 conflict on ride accept.
    - 410 gone on operations on expired rides.

### 5.2 WebSockets / Socket.io

- Connects using Firebase ID token.
- Joins per-ride rooms with format `ride_{ride_id}`.
- Listens for:
  - New chat messages.
  - Real-time ride status updates if exposed via events.

### 5.3 Push Notifications (FCM)

- Registers for FCM and stores device tokens via `/auth/fcm-token`.
- Handles incoming notifications:
  - Chat notifications: open ride chat for the relevant `ride_id`.
  - Ride status notifications: open ride details.

---

## 6. Testing Strategy (Frontend)

- **Unit tests**
  - Providers (auth, ride stream, etc.).
  - Pure business logic.
- **Widget tests**
  - Critical screens: Request Ride, Available Rides, Ride Details, Chat.
- **Integration tests**
  - End-to-end flows for Passenger and Driver journeys.

