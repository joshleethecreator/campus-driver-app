# Campus Driver App — Product Requirements (PRD)

This document describes the **product behavior and user-facing requirements** for the Campus Driver App.  
It is implementation-agnostic and meant for product, UX, and design discussions.

For lower-level technical details, see:
- `docs/frontend.md`
- `docs/backend.md`
- `docs/api.md`

For a UX-focused feature and screen summary, see `docs/FEATURES_UX.md`.

---

## 1. Product Summary

- A campus ride-request app with **two primary roles**: **Passenger** and **Driver**.
- Passengers can request **immediate** rides or **schedule future rides** with pickup/dropoff; drivers browse and accept ride requests.
- Each ride has **real-time status updates** and **1:1 chat** between the passenger and assigned driver.
- Users authenticate via **Firebase**. The system uses **push notifications (FCM)** and **Socket.io** so rides and chat feel live.

---

## 2. Roles / Personas

### 2.1 Passenger

- Requests a ride by selecting pickup and dropoff locations.
- Sees the current ride state (pending, accepted, ongoing, completed).
- Chats with the assigned driver for coordination.
- Receives notifications about ride events and chat messages when not in-app.

### 2.2 Driver

- Completes onboarding, including a **driver verification** step.
- Sees a list of available **pending** ride requests.
- Accepts one ride at a time (with conflict handling if someone else accepts first).
- Chats with passengers.
- Receives notifications when new rides are assigned or chat messages arrive.

### 2.3 Admin / Reviewer (Implied)

- Reviews driver verification submissions.
- Needs access to uploaded license images or documents via secure links.

---

## 3. Screen Inventory (User-facing)

### 3.1 Authentication

- **Login**: Email/password or other Firebase-backed auth.
- **Register / Onboarding**: Account creation and initial profile completion.

### 3.2 Passenger Screens

- **Request Ride**
  - Select pickup and dropoff (powered by location search).
  - Choose **Ride now** or **Schedule for later** (date/time within an allowed window).
  - Submit ride request.
- **Location Search**
  - Search UI for addresses/points of interest (with loading, empty, and error states).
- **Ride Status / Ride Details**
  - Shows ride state and key details.
  - Entry point to chat.
- **Ride Chat**
  - 1:1 messaging with the assigned driver.
  - Includes retention notice about messages being cleared daily.
- **Profile / Settings**
  - Basic account info and preferences (exact fields TBD).

### 3.3 Driver Screens

- **Driver Onboarding / Verification**
  - Upload license and any required documents.
  - See verification status (`pending`, `approved`).
- **Available Rides**
  - List of pending ride requests the driver can accept.
- **Ride Details + Accept**
  - Detailed view of a selected ride.
  - Action to accept (with conflict feedback if already taken).
- **Active Ride / Ride Status**
  - Shows active ride information and current state.
- **Ride Chat**
  - Same chat experience as passenger.
- **Profile / Settings**
  - Vehicle details and verification state display.

### 3.4 Shared / System Behaviors

- **Notification Entry Points**
  - Tapping a notification opens either Ride Details or Chat, depending on context.
- **Error / Empty States**
  - No available rides.
  - Network errors.
  - Expired rides.
  - Conflicts when a ride has already been accepted by another driver.

---

## 4. Core User Journeys

### 4.1 Passenger Journey — Request Ride to Completion

1. Passenger logs in.
2. Opens **Request Ride**.
3. Uses **Location Search** to pick pickup and dropoff.
4. Submits the ride request.
5. Ride enters **pending** state.
6. Passenger waits for a driver:
   - In-app: sees live status updates.
   - Out of app: receives **ride status notifications**.
7. Once a driver accepts, passenger can open **Ride Chat** to coordinate.
8. Ride progresses through **accepted → ongoing → completed**.
9. At the end of the day, chat history is cleared (see constraints).

### 4.2 Driver Journey — Browse, Accept, Complete

1. Driver logs in and completes onboarding, including verification.
2. Opens **Available Rides**.
3. Selects a pending ride from the list and reviews details.
4. Taps **Accept**:
   - If another driver already accepted, the driver sees a conflict message and the list is refreshed.
5. On success, the ride enters **accepted** state for that driver.
6. Driver uses **Ride Chat** to coordinate with the passenger.
7. Ride progresses through **ongoing → completed**, either manually or via auto-complete rules.

### 4.3 Passenger Journey — Scheduled Ride

1. Passenger logs in.
2. Opens **Request Ride**.
3. Uses **Location Search** to pick pickup and dropoff.
4. Chooses a **future pickup time** (within an allowed scheduling window, e.g. up to 24 hours ahead).
5. Submits the scheduled ride request.
6. Ride is stored as a scheduled ride and surfaces to drivers based on its pickup time (exact matching/allocation rules TBD).
7. Before the scheduled time:
   - Passenger can view upcoming rides in Ride Details.
   - Passenger can cancel or edit within allowed rules (exact policy TBD).
8. Around the scheduled time:
   - Drivers see the ride in **Available Rides**.
   - Passenger receives notifications when a driver accepts.
9. After acceptance, the journey continues like an immediate ride:
   - Passenger and driver can chat.
   - Ride progresses through **accepted → ongoing → completed**.

---

## 5. Feature List (User-visible Behavior)

### 5.1 Accounts & Authentication

- Firebase-backed login and registration.
- Each authenticated user has an app profile.

### 5.2 Driver Verification

- Driver profile includes a `verification_status` with at least:
  - `pending`
  - `approved`
- Drivers upload license images or documents for review.
- Admins/reviewers can view submissions via secure signed URLs.
- Product expectations:
  - Clear copy explaining why verification is needed.
  - Clear indication of current verification state.
  - Restrictions or warnings if the driver is not yet approved (exact rules TBD).

### 5.3 Location Search

- Search inputs for pickup and dropoff using a backend-proxied location API.
- UX must handle:
  - Loading state.
  - No results found.
  - Network or server errors.
  - Editing previously selected locations.

### 5.4 Ride Lifecycle & State-driven UI

- Rides have states:
- For **immediate rides**:
  - `pending`
  - `accepted`
  - `ongoing`
  - `completed`
- For **scheduled rides**, there is an additional conceptual phase:
  - “scheduled” (requested for a future time; not yet active for a driver until window opens)
- Screens and available actions must react to these states for both passenger and driver.

### 5.5 Driver “Available Rides” & Accept Flow

- Drivers only see **pending** rides in the Available Rides list.
- When accepting a ride:
  - App must handle the case where another driver has already accepted:
    - Show a clear message (e.g. “Ride already accepted by another driver.”).
    - Refresh list to remove the taken ride.

### 5.6 Real-time Chat

- Each ride has a private chat channel between its passenger and driver.
- Only the assigned passenger and driver can see and send messages.
- UI expectations:
  - Scrollable message list.
  - Input box for composing messages.
  - Timestamps or relative times.
  - Handling of connectivity issues (retry/reconnect).

### 5.7 Push Notifications

- **Chat Notifications**
  - Sent when a new chat message is created and the recipient is not currently in the chat.
  - Notification includes:
    - Ride identifier.
    - Sender name.
    - Short preview.
- **Ride Status Notifications**
  - Sent on key events:
    - Ride created (optional).
    - Ride accepted.
    - Ride cancelled.
    - Ride completed.
  - Tapping the notification deep-links to:
    - Ride Details, or
    - Ride Chat, depending on context.

### 5.8 Scheduled Rides

- Passengers can choose between:
  - **Ride now** (immediate request), or
  - **Schedule for later** (future pickup time within a configured window).
- Scheduled rides must:
  - Be clearly labeled as “Scheduled” in all relevant screens (Passenger and Driver).
  - Show the scheduled pickup time.
- Product expectations:
  - A simple view of upcoming rides for passengers.
  - Appropriate restrictions on when a scheduled ride can be changed/cancelled (policy TBD).
  - Drivers can see which rides are scheduled and when they are expected to start.

---

## 6. Constraints & Automation (Impacting UX)

### 6.1 Chat Retention

- Chat histories for non-active rides are cleared daily.
- The chat screen must always show the notice:

```text
ⓘ Messages will be cleared at end of day.
```

### 6.2 Time-based Ride Automation

- **Ride expiration**:
  - Pending rides older than **2 hours** are automatically cancelled.
- **Auto-complete**:
  - Ongoing rides older than **3 hours** are automatically marked as completed.
- UX expectations:
  - If a ride disappears or changes state due to automation, the app should clearly communicate why.

### 6.3 Offline / Poor Network Behavior

- The app caches the last-known user profile for faster startup and better offline behavior.
- UX should support:
  - Showing cached data with an “offline” indicator.
  - Graceful degradation when real-time features (chat, status) are unavailable.

---

## 7. Error Handling (From a UX Perspective)

- **409 Conflict (Driver Accept)**
  - Meaning: The driver tried to accept a ride that was just taken by someone else.
  - UX: Show a friendly message and redirect to an updated list of available rides.

- **410 Gone (Expired Ride)**
  - Meaning: A passenger tried to interact with a ride that has expired.
  - UX: Explain that the ride expired and offer an easy way to request a new ride.

---

## 8. Open Questions / TBD

- Detailed onboarding fields for passengers and drivers.
- Exact restrictions for drivers while `verification_status = pending`.
- Whether pricing / fare estimates are shown, and how.
- Exactly which transitions are user-triggered vs automated in the client UI.
- Navigation structure and explicit route names for deep-links.
- Detailed behavior for **scheduled rides**:
  - Maximum scheduling window (e.g. up to 24 or 48 hours ahead).
  - Cutoff rules for editing/cancelling scheduled rides.
  - Whether scheduled rides are auto-cancelled if no driver accepts by a certain time.

