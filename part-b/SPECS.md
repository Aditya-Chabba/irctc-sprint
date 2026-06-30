# IRCTC Feature Specifications — Part B

---

## Spec 1: Tatkal Booking Virtual Queue
*Addresses Part A Problem 1: Tatkal Booking Crashes at 10:00 AM*

### Problem Statement
Every morning at 10:00 AM, an estimated 20–40 lakh users hit the Tatkal booking endpoint simultaneously, causing the server to freeze or return HTTP 502 errors with zero feedback. Users who completed every step up to payment lose their seats and have no way to know if their request was queued, failed, or succeeded — leading to repeat clicks that compound the server load and worsen the crash. This is a daily, severe failure that directly affects whether Tier 2/3 city travelers can secure essential train travel.

### Proposed Solution
When a user clicks "Book Now" at or near 10:00 AM, instead of sending the request directly to an overloaded endpoint, they are placed into a visible virtual queue. The user sees a live queue position counter (e.g. "#4,281") and an estimated wait time. When their turn arrives, they get a 90-second window to complete passenger details and payment before their slot expires and passes to the next person in line. This replaces silent freezing with constant, honest feedback.

### Technical Implementation Plan
**System components affected:** Frontend (new queue UI component), Backend API (new queue service), Database (queue state table), no changes needed to the core payment gateway integration.

**New data requirements:** A `tatkal_queue` table/collection with fields: `user_id`, `train_id`, `class`, `quota`, `queue_position`, `entered_at`, `expires_at`, `status` (waiting/active/expired/completed). A lightweight in-memory queue (Redis) is preferable to a relational table for ordering and TTL expiry at this scale.

**API changes:**
- `POST /api/tatkal/queue/join` — Request body: `{train_id, class, quota, user_id}` — Response: `{queue_position, estimated_wait_seconds}`
- `GET /api/tatkal/queue/status` — Response: `{queue_position, status, time_remaining}` (polled or pushed via WebSocket)
- `POST /api/tatkal/queue/confirm` — called when it's the user's turn, locks the seat for 90 seconds — Response: `{seat_locked: true, expires_at}`

**Frontend state changes:** New queue screen/route shown immediately after clicking "Book Now" during Tatkal hours. Local state holds queue position and a live countdown timer. WebSocket or short-poll (every 2–3 sec) subscription updates position in real time.

**Third-party services:** Redis (or equivalent in-memory queue) for managing queue order and TTL-based slot expiry at high concurrency; no external AI/ML service needed for this spec.

### Success Metrics
Tatkal booking completion rate increases from an estimated ~40% to 70%+ during the 10:00–10:05 AM peak window. Server error rate (502s, timeouts) during this window drops to near zero, since requests are queued rather than hitting the booking endpoint simultaneously. User-reported "I don't know what happened" complaints (via support tickets/social media mentions) decrease measurably.

### Edge Cases and Constraints
What happens if a user's internet drops while in queue — their slot should hold for a grace period (e.g. 15 seconds) before expiring, to allow reconnection. What happens if the queue itself becomes overloaded — the system needs a documented cap on queue depth with messaging ("Tatkal quota is full for this train") rather than infinite queuing. This depends on Railway backend APIs for actual seat allocation, which IRCTC does not fully control — the queue can guarantee fair ordering for *attempts* but cannot guarantee seat availability if the backend's own quota check fails independently.

---



## Spec 2: Persistent, Reliable Search Filters
*Addresses Part A Problem 2: Search Filters Do Not Work Reliably*

### Problem Statement
Train search filters (class, quota, availability, departure time) apply to a stale cached result set roughly 30-40% of the time, showing waitlisted trains under an "Available" filter and resetting to "All Classes" when the user navigates back. All 8 crore registered users searching for trains are affected, with senior citizens and first-time users most harmed since they trust filter output without double-checking — adding 8-15 minutes of manual scanning per search when filters fail.

### Proposed Solution
Filters are applied against live, freshly-fetched data rather than a cached snapshot, and filter selections persist in the URL/session state so they survive back-navigation and page reloads. The user selects a filter, sees results update only after confirming the data is current (a small "Live" indicator next to results), and returning to the page via back button preserves their exact filter selection without requiring reapplication.

### Technical Implementation Plan
**System components affected:** Frontend (filter state management, URL query params), Backend API (live availability fetch instead of cached), no database schema changes needed.

**New data requirements:** No new persistent data needed — filter state is ephemeral, stored in URL query parameters (e.g. `?class=SL&avail=true&dep=morning`) and synced to browser history/session storage equivalent (React state, not localStorage, per artifact constraints) so back-navigation restores it.

**API changes:**
- `GET /api/trains/search?from=X&to=Y&date=Z&class=SL&availability=true` — filters are now passed as query parameters directly to the search endpoint, ensuring the backend returns pre-filtered, live data rather than the frontend filtering a stale cached response.

**Frontend state changes:** Filter selections live in URL query state (using router state) instead of local-only component state, so refresh and back-navigation read filters directly from the URL rather than resetting to defaults. A "Live - Updated Xs ago" timestamp shown next to results to set accurate expectations.

**Third-party services:** None required — this is a data-freshness and state-management fix, not a new capability.

### Success Metrics
Filter-result mismatch rate (filtered results that don't match the filter criteria) drops from ~30-40% to under 5%. Average time-to-find-preferred-train decreases from 8-15 minutes (manual scanning) to under 2 minutes. Filter-reset-on-back-navigation complaints drop to near zero, measured via reduced repeat filter-application clicks in session analytics.

### Edge Cases and Constraints
What happens if live availability changes between when results load and when the user clicks a train — the train list should show a brief "Refreshing..." state on click rather than silently booking against stale data. What happens on slow/2G connections where live fetches take longer — a loading skeleton should replace the abrupt "page reload" behavior currently seen. This depends on the Railway backend's actual availability API response time, which IRCTC does not control, so a reasonable timeout (e.g. 5 seconds) with graceful fallback to last-known-good data is needed rather than an indefinite spinner.

---



## Spec 3: Reliable Seat Selection State Persistence
*Addresses Part A Problem 3: Seat Selection Resets Randomly*

### Problem Statement
When a user selects a specific seat (e.g. a lower berth for an elderly passenger) in the seat map and proceeds to the next step, the selection is lost in 15-25% of sessions, rising to 35% on mobile, due to the seat state not passing correctly between the seat map component and the passenger details form. This disproportionately harms families with elderly/disabled members and anyone with a specific medical or comfort need, since they end up with an "Auto" assignment that could place them anywhere on the train.

### Proposed Solution
The selected seat is locked and visibly confirmed the moment the user clicks it, with a persistent confirmation banner ("Lower Berth #34 selected ✓") that follows the user through every subsequent screen until booking completes. If the selection cannot be carried forward for any reason, the user is shown an explicit warning before proceeding, rather than silently defaulting to "Auto."

### Technical Implementation Plan
**System components affected:** Frontend (seat map component, passenger details form, shared state layer), Backend API (seat-hold/lock endpoint), no major database schema changes beyond adding a short-lived hold record.

**New data requirements:** A `seat_hold` table/collection with fields: `session_id`, `train_id`, `seat_number`, `class`, `held_at`, `expires_at` (e.g. 10-minute hold) to guarantee the seat stays reserved for that user across screens until booking completes or the hold expires.

**API changes:**
- `POST /api/seats/hold` — Request body: `{train_id, class, seat_number, session_id}` — Response: `{hold_confirmed: true, expires_at}` — called the instant a seat is clicked in the seat map.
- `GET /api/seats/hold/status` — Response: `{seat_number, status, time_remaining}` — used by the passenger details page to confirm and display the held seat instead of re-deriving it from scratch.

**Frontend state changes:** Seat selection moves from local component-only state (lost on navigation/re-render) to a shared session-level state (e.g. React Context or global state) that the passenger details page reads directly, plus a persistent visual confirmation banner across screens. On mobile specifically, the re-render that currently clears local state is fixed by reading from this shared state instead of resetting component state on navigation.

**Third-party services:** None required.

### Success Metrics
Seat selection persistence failure rate drops from 15-25% (35% on mobile) to under 3% across all devices. Support complaints related to "wrong seat assigned" or "got Auto instead of my selection" decrease measurably. Mobile-specific failure rate gap closes to within 2% of desktop, confirming the re-render bug is resolved.

### Edge Cases and Constraints
What happens if two users try to hold the same seat simultaneously — the hold endpoint must be atomic (first hold wins, second gets an immediate "seat just taken" message rather than a silent failure). What happens if a user's hold expires while filling passenger details — they should get a clear warning with a one-click "extend hold" or "reselect seat" option rather than discovering the loss only at payment. This depends on the Railway backend's seat inventory system actually honoring holds in real time, which IRCTC's backend may not natively support — a fallback design (client-side optimistic locking with backend reconciliation) should be planned in case true server-side seat-holding isn't available.

---



## Spec 4: Clear Feedback for Logged-Out Search Attempts
*Addresses Part A Problem 4: Logged-Out Users Get No Feedback When Searching for Trains*

### Problem Statement
When a logged-out user fills in valid search criteria (source, destination, date, class) and clicks "Search Trains," nothing happens — no redirect, no error message, no login prompt. This affects every first-time or logged-out visitor trying to evaluate train options before registering, a very common pattern, and the silent failure (observed in 100% of attempts) leaves users unable to tell if it's their input, their connection, or the platform that's broken.

### Proposed Solution
Logged-out users can search and view train results exactly as logged-in users do — search should never require login at all, since IRCTC's own stated flow only requires login to book, not to search. If account-gated, clicking "Search Trains" while logged out immediately shows a clear inline message ("Please login to search trains") with a one-click login button, instead of doing nothing.

### Technical Implementation Plan
**System components affected:** Frontend (search submission handler, auth-state check), Backend API (search endpoint auth requirements), no database changes needed.

**New data requirements:** None.

**API changes:**
- `GET /api/trains/search` — clarify and enforce: this endpoint should not require an auth token at all (since search is pre-booking, account-agnostic information). If login truly is required by design, the endpoint should return a clear `401 Unauthorized` with a descriptive error body `{error: "login_required", message: "Please login to search trains"}` instead of failing silently.

**Frontend state changes:** The search submission handler checks auth state before calling the API; if unauthenticated and login is required, it shows an inline modal/banner with a "Login" CTA instead of submitting a request that silently fails. If login is not actually required, simply removing any auth gating on this action resolves the issue entirely.

**Third-party services:** None required.

### Success Metrics
Search submission "silent failure" rate (defined as a click producing no UI response within 2 seconds) drops from 100% (for logged-out users) to 0%. Conversion from logged-out search attempt to successful registration/login increases, since users now have a clear path forward instead of abandoning silently.

### Edge Cases and Constraints
What happens if the user's session expires mid-search (was logged in, becomes logged out) — the same clear messaging should apply rather than a different silent-failure path. This is a low-effort, high-clarity fix that depends only on internal frontend/backend logic — no Railway backend dependency, since this is purely an auth-gating and feedback issue, not a data-availability one.

---



## Spec 5: Live Refresh on Class/Quota Change
*Addresses Part A Problem 5: Class Change Doesn't Refresh Availability*

### Problem Statement
On the search results page, switching the class dropdown (e.g. AC 3 Tier to Anubhuti Class) updates the dropdown label but leaves the fare, waitlist numbers, and "last updated" timestamp frozen on the previous class's data — observed consistently across multiple class switches in testing. Every logged-in user comparing classes to find better availability is affected, since the default search result rarely shows the user's preferred class with confirmed seats on first load, and the stale data actively misleads users into comparing identical-looking numbers across genuinely different products.

### Proposed Solution
Selecting a new class or quota immediately triggers a visible loading state on the fare/availability card, followed by a live re-fetch showing the correct fare, waitlist status, and a fresh "Updated just now" timestamp for that specific class. The user always sees data that matches the class currently selected in the dropdown — no stale carryover.

### Technical Implementation Plan
**System components affected:** Frontend (class/quota dropdown change handler, results card re-render logic), Backend API (per-class availability fetch), no database schema changes needed.

**New data requirements:** None — this is a data-binding fix, not a new data model.

**API changes:**
- `GET /api/trains/{train_id}/availability?class=3A&quota=GN` — existing endpoint, but the frontend must now call it on every class/quota dropdown change rather than only on initial page load.

**Frontend state changes:** The class dropdown's `onChange` handler is wired to trigger a re-fetch of availability data scoped to the newly selected class/quota, replacing the current behavior where only the dropdown's visual label updates. A loading skeleton or spinner shows briefly during the re-fetch so the user knows new data is loading rather than assuming nothing happened.

**Third-party services:** None required.

### Success Metrics
Stale-data display rate after a class/quota change drops from ~100% (observed) to 0% — every class switch must produce a corresponding data refresh. User complaints or support tickets referencing "fare didn't update" or "wrong seat availability shown" decrease measurably.

### Edge Cases and Constraints
What happens if the re-fetch is slow on weak connections — the previous class's data should remain visible but visually dimmed/marked stale ("Showing AC 3 Tier data while loading Anubhuti Class...") rather than blanking out entirely. What happens if the user rapidly switches classes multiple times before any fetch completes — only the most recent request's response should update the UI (request cancellation/debouncing needed) to avoid race conditions showing the wrong class's data.

---