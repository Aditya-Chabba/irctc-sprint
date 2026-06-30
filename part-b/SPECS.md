# IRCTC Feature Specifications — Part B

---

## Feature Spec 1: Tatkal Booking Virtual Queue

### Problem Statement
Every morning at 10:00 AM, an estimated 20-40 lakh users hit the Tatkal booking endpoint simultaneously, causing server freezes and HTTP 502 errors with zero feedback (Part A, Problem 1). Users who reach the payment page lose their seats with no way to know if their request was queued, failed, or succeeded. This is a daily, severe failure that determines whether Tier 2/3 city travelers can secure essential train travel.

### Current State (from Part A)
As documented in PROBLEMS.md, the current flow breaks at Steps 4-6: the user clicks "Book Now" at 9:59:45, the page freezes at 10:00:00 with no progress feedback, and after 15-45 seconds returns either an HTTP 502, a session timeout, or a CAPTCHA reset. The system gives zero indication of what happened, causing repeat clicks that compound server load and worsen the crash for everyone.

### Proposed Solution
When a user clicks "Book Now" at or near 10:00 AM, instead of hitting the overloaded endpoint directly, they are placed into a visible virtual queue. They see a live queue position counter and estimated wait time. When their turn arrives, they get a 90-second window to complete passenger details and payment before their slot expires and passes to the next person.

### Proposed User Flow — Step by Step
1. User opens IRCTC at 9:50 AM, logs in, searches for the train, selects Tatkal quota
2. User fills passenger details and clicks "Book Now" at 9:59:45 AM
3. Instead of freezing, the page transitions to a Queue Screen showing "You are #4,281 in line"
4. A live countdown shows estimated wait time, updating every few seconds
5. User can leave the tab open and check back — position updates in real time
6. When the user's turn arrives, they get a banner: "Your turn! Complete booking in 90 seconds"
7. Passenger details (already filled) and payment screen load instantly, seat is held for 90 seconds
8. User completes payment within the window — booking confirms
9. If the user misses the 90-second window, they see a clear message: "Your slot expired — rejoin queue" with a one-click rejoin option, instead of silently losing the seat with no explanation

### Technical Implementation Plan
**System components affected:**
- Frontend (new queue UI component, countdown screen)
- Backend API (new queue service)
- Database (queue state table)
- No changes needed to the core payment gateway integration

**New data requirements:**
- A `tatkal_queue` table/collection with fields: `user_id`, `train_id`, `class`, `quota`, `queue_position`, `entered_at`, `expires_at`, `status` (waiting/active/expired/completed)
- Redis (in-memory store) preferred over a relational table for ordering and TTL-based expiry at this concurrency scale

**API changes:**
- `POST /api/tatkal/queue/join` — Request: `{train_id, class, quota, user_id}` — Response: `{queue_position, estimated_wait_seconds}`
- `GET /api/tatkal/queue/status` — Response: `{queue_position, status, time_remaining}` (polled or pushed via WebSocket)
- `POST /api/tatkal/queue/confirm` — called when it's the user's turn, locks the seat for 90 seconds — Response: `{seat_locked: true, expires_at}`

**Frontend changes:**
- New queue screen/route shown immediately after "Book Now" during Tatkal hours
- Local state holds queue position and a live countdown timer
- WebSocket or short-poll (every 2-3 sec) subscription updates position in real time

**Third-party services (if any):**
- Redis (or equivalent in-memory queue) for managing queue order and TTL-based slot expiry at high concurrency

### Success Metrics
- Tatkal booking completion rate increases from an estimated ~40% to 70%+ during the 10:00-10:05 AM peak window
- Server error rate (502s, timeouts) during this window drops to near zero
- User-reported "I don't know what happened" support tickets/social mentions decrease measurably

### Edge Cases and Constraints
- What happens if a user's internet drops while in queue — their slot should hold for a grace period (e.g. 15 seconds) before expiring, to allow reconnection
- What happens if the queue itself becomes overloaded — a documented cap on queue depth with clear messaging ("Tatkal quota is full for this train") rather than infinite queuing
- This depends on Railway backend APIs for actual seat allocation, which IRCTC does not fully control — the queue can guarantee fair ordering for attempts but cannot guarantee seat availability if the backend's own quota check fails independently
- Graceful degradation: if the queue service itself goes down, the system should fall back to the current direct-booking flow rather than blocking all bookings entirely

---

## Feature Spec 2: Persistent, Reliable Search Filters

### Problem Statement
Train search filters (class, quota, availability, departure time) apply to a stale cached result set roughly 30-40% of the time, showing waitlisted trains under an "Available" filter and resetting to "All Classes" when the user navigates back (Part A, Problem 2). All 8 crore registered users searching for trains are affected, with senior citizens and first-time users most harmed since they trust filter output without double-checking.

### Current State (from Part A)
As documented in PROBLEMS.md, the flow breaks at Steps 3-6: the user applies a filter (e.g. Sleeper Class, Available), the page reloads showing waitlisted trains anyway, and clicking back resets the filter to "All Classes" entirely. This forces users to abandon filtering and manually scan 20-40 trains, adding 8-15 minutes to a simple search.

### Proposed Solution
Filters are applied against live, freshly-fetched data rather than a cached snapshot, and filter selections persist in the URL/session state so they survive back-navigation and page reloads. The user selects a filter, sees results update with a "Live" indicator confirming the data is current, and returning via the back button preserves their exact filter selection.

### Proposed User Flow — Step by Step
1. User enters source, destination, date, clicks Search Trains
2. Results load showing 20-40 trains with a "Live - Updated just now" indicator
3. User selects "Sleeper Class" and "Available" from the filter panel
4. Results re-fetch live (brief loading state) and update to show only matching, currently-available trains
5. User clicks into a train — class shown matches exactly what the filter promised
6. User clicks back — filter selections are still applied exactly as left them, results still match
7. User changes filters again if needed — same live, accurate behavior repeats
8. User finds and books a matching train in under 2 minutes instead of 8-15

### Technical Implementation Plan
**System components affected:**
- Frontend (filter state management, URL query params)
- Backend API (live availability fetch instead of cached)
- No database schema changes needed

**New data requirements:**
- None persistent — filter state is ephemeral, stored in URL query parameters and synced to router/session state so back-navigation restores it

**API changes:**
- `GET /api/trains/search?from=X&to=Y&date=Z&class=SL&availability=true` — filters passed as query parameters directly to the search endpoint, so the backend returns pre-filtered, live data

**Frontend changes:**
- Filter selections live in URL query state (router state) instead of local-only component state
- A "Live - Updated Xs ago" timestamp shown next to results

**Third-party services (if any):**
- None required

### Success Metrics
- Filter-result mismatch rate drops from ~30-40% to under 5%
- Average time-to-find-preferred-train decreases from 8-15 minutes to under 2 minutes
- Filter-reset-on-back-navigation complaints drop to near zero

### Edge Cases and Constraints
- What happens if live availability changes between load and click — a brief "Refreshing..." state on click rather than booking against stale data
- What happens on slow/2G connections — a loading skeleton replaces the abrupt "page reload" currently seen
- Depends on Railway backend's availability API response time, which IRCTC does not control — a 5-second timeout with graceful fallback to last-known-good data is needed
- Graceful degradation: if live fetch fails entirely, show last-known results with a clear "could not refresh" warning rather than a blank page

---

## Feature Spec 3: Reliable Seat Selection State Persistence

### Problem Statement
When a user selects a specific seat (e.g. a lower berth for an elderly passenger) and proceeds to the next step, the selection is lost in 15-25% of sessions, rising to 35% on mobile (Part A, Problem 3). This disproportionately harms families with elderly/disabled members, who end up with an "Auto" assignment that could place them anywhere on the train.

### Current State (from Part A)
As documented in PROBLEMS.md, the flow breaks at Steps 3-5: the user selects a lower berth, clicks "Proceed," and the passenger details page shows "Auto" or a different berth — not what was selected. Going back to reselect shows the seat as already taken. The seat selection state is not passed correctly between the seat map component and the passenger form, and mobile re-renders clear local state entirely.

### Proposed Solution
The selected seat is locked and visibly confirmed the instant the user clicks it, with a persistent confirmation banner ("Lower Berth #34 selected ✓") that follows the user through every subsequent screen until booking completes. If the selection cannot be carried forward, the user sees an explicit warning before proceeding, instead of silently defaulting to "Auto."

### Proposed User Flow — Step by Step
1. User selects train, class, and quota, proceeds to seat selection
2. Seat map loads showing available, booked, and selected berths
3. User clicks a lower berth — it turns blue and a confirmation banner appears: "Lower Berth #34 selected ✓ (held for 10 min)"
4. User clicks "Proceed" — passenger details page loads
5. The same confirmation banner persists at the top: "Lower Berth #34 selected ✓"
6. Passenger details form shows the correct, locked seat — not "Auto"
7. User completes passenger details and proceeds to payment — banner still visible
8. If the hold expires before payment, user sees: "Your seat hold expired — reselect or extend" with one-click options, instead of silently losing the seat

### Technical Implementation Plan
**System components affected:**
- Frontend (seat map component, passenger details form, shared state layer)
- Backend API (seat-hold/lock endpoint)
- Database (short-lived hold record)

**New data requirements:**
- A `seat_hold` table/collection: `session_id`, `train_id`, `seat_number`, `class`, `held_at`, `expires_at` (e.g. 10-minute hold)

**API changes:**
- `POST /api/seats/hold` — Request: `{train_id, class, seat_number, session_id}` — Response: `{hold_confirmed: true, expires_at}` — called the instant a seat is clicked
- `GET /api/seats/hold/status` — Response: `{seat_number, status, time_remaining}` — used by the passenger details page to confirm and display the held seat

**Frontend changes:**
- Seat selection moves from local component-only state to shared session-level state (e.g. React Context) that the passenger details page reads directly
- Persistent visual confirmation banner across all subsequent screens
- Mobile re-render reads from shared state instead of resetting on navigation

**Third-party services (if any):**
- None required

### Success Metrics
- Seat selection persistence failure rate drops from 15-25% (35% on mobile) to under 3% across all devices
- Support complaints about "wrong seat" or "got Auto instead" decrease measurably
- Mobile-specific failure rate gap closes to within 2% of desktop

### Edge Cases and Constraints
- What happens if two users try to hold the same seat simultaneously — the hold endpoint must be atomic (first hold wins, second gets immediate "seat just taken" message)
- What happens if a user's hold expires while filling passenger details — a clear warning with one-click "extend hold" or "reselect seat" option
- Depends on the Railway backend's seat inventory system honoring holds in real time, which may not be natively supported — a fallback design (client-side optimistic locking with backend reconciliation) should be planned
- Graceful degradation: if the hold service is unavailable, fall back to current "Auto" behavior but with an explicit warning shown to the user rather than silent substitution

---

## Feature Spec 4: Clear Feedback for Logged-Out Search Attempts

### Problem Statement
When a logged-out user fills in valid search criteria and clicks "Search Trains," nothing happens — no redirect, no error, no login prompt (Part A, Problem 4). This affects every first-time or logged-out visitor trying to evaluate train options before registering, observed in 100% of attempts, leaving users unable to tell if it's their input, connection, or the platform that's broken.

### Current State (from Part A)
As documented in PROBLEMS.md, the flow breaks at Steps 5-6: the user enters valid source, destination, date, and class, clicks "Search Trains," and the page produces no visible response whatsoever — no spinner, no error, no navigation.

### Proposed Solution
Logged-out users can search and view train results exactly as logged-in users do — search should never require login, since login is only meant to gate booking, not searching. If account-gating is genuinely required by design, clicking "Search Trains" while logged out immediately shows a clear inline message ("Please login to search trains") with a one-click login button.

### Proposed User Flow — Step by Step
1. User opens IRCTC without logging in
2. User enters source, destination, date, class
3. User clicks "Search Trains"
4. Results load immediately and normally, exactly as they would for a logged-in user
5. User can browse all train options, fares, and availability without an account
6. Only when the user clicks "Book Now" on a specific train are they prompted to login/register
7. (Alternative, if login truly is required for search): user sees an inline banner immediately on click: "Please login to search trains" with a "Login" button, instead of silence

### Technical Implementation Plan
**System components affected:**
- Frontend (search submission handler, auth-state check)
- Backend API (search endpoint auth requirements)
- No database changes needed

**New data requirements:**
- None

**API changes:**
- `GET /api/trains/search` — should not require an auth token at all. If login truly is required, the endpoint should return a clear `401 Unauthorized` with `{error: "login_required", message: "Please login to search trains"}` instead of failing silently

**Frontend changes:**
- The search submission handler checks auth state before calling the API; if login is required and the user is unauthenticated, an inline modal/banner with a "Login" CTA appears instead of a silent failure

**Third-party services (if any):**
- None required

### Success Metrics
- Search submission "silent failure" rate drops from 100% (logged-out users) to 0%
- Conversion from logged-out search attempt to registration/login increases

### Edge Cases and Constraints
- What happens if the user's session expires mid-search — the same clear messaging should apply
- Low-effort fix with no Railway backend dependency, since this is purely an auth-gating and feedback issue
- Graceful degradation: even if the login-check logic itself fails, the system should default to allowing search rather than silently blocking it

---

## Feature Spec 5: Live Refresh on Class/Quota Change

### Problem Statement
On the search results page, switching the class dropdown updates only the label while fare, waitlist numbers, and "last updated" timestamp stay frozen on the previous class's data (Part A, Problem 5), observed consistently across multiple class switches. Every logged-in user comparing classes to find better availability is affected, since stale data actively misleads users into comparing identical-looking numbers across genuinely different products.

### Current State (from Part A)
As documented in PROBLEMS.md, the flow breaks at Step 4: selecting a new class from the dropdown updates the visual label only — fare (₹1895), waitlist status (WL14/WL11/WL4), and "Updated 8 Minutes and 41 Seconds ago" remain identical no matter which class is selected.

### Proposed Solution
Selecting a new class or quota immediately triggers a visible loading state on the fare/availability card, followed by a live re-fetch showing the correct fare, waitlist status, and a fresh "Updated just now" timestamp for that specific class. The user always sees data matching the currently selected class.

### Proposed User Flow — Step by Step
1. User logs in and navigates to a train's seat selection/availability screen
2. Page loads showing fare and availability for the default class
3. User selects a different class from the dropdown
4. A brief loading skeleton appears on the fare/availability card
5. New data loads: updated fare, updated waitlist/RAC numbers, "Updated just now" timestamp
6. User selects another class — same live refresh repeats accurately
7. User can confidently compare real data across classes to make a booking decision

### Technical Implementation Plan
**System components affected:**
- Frontend (class/quota dropdown change handler, results card re-render logic)
- Backend API (per-class availability fetch)
- No database schema changes needed

**New data requirements:**
- None — this is a data-binding fix, not a new data model

**API changes:**
- `GET /api/trains/{train_id}/availability?class=3A&quota=GN` — existing endpoint, but frontend must call it on every class/quota dropdown change, not just initial page load

**Frontend changes:**
- The class dropdown's `onChange` handler triggers a re-fetch of availability data scoped to the newly selected class/quota
- A loading skeleton/spinner shows briefly during re-fetch

**Third-party services (if any):**
- None required

### Success Metrics
- Stale-data display rate after a class/quota change drops from ~100% (observed) to 0%
- Support tickets referencing "fare didn't update" or "wrong availability shown" decrease measurably

### Edge Cases and Constraints
- What happens if the re-fetch is slow on weak connections — previous class's data stays visible but dimmed/marked stale rather than blanking out
- What happens if the user rapidly switches classes before any fetch completes — only the most recent request's response should update the UI (debouncing/request cancellation needed) to avoid race conditions
- Graceful degradation: if the re-fetch fails, show an explicit "couldn't load [class] data — tap to retry" message rather than silently leaving stale data displayed

---

## Feature Spec 6: Session-Aware Search Results Page

### Problem Statement
When a logged-in user logs out while viewing a search results page, the page does not detect the logout — it keeps showing stale, authenticated-looking content, and clicking "Modify Search" afterward triggers a "Please Wait..." overlay that never resolves (Part A, Problem 6). This affects users managing sessions across tabs, observed to fail consistently on every interaction attempt post-logout.

### Current State (from Part A)
As documented in PROBLEMS.md, the flow breaks at Steps 4 and 6: after logout, the results page remains visually unchanged except for the top nav reverting to "LOGIN / REGISTER," and clicking "Modify Search" produces an indefinite "Please Wait..." overlay with no timeout or error.

### Proposed Solution
The results page actively listens for session/auth-state changes. The instant a logout is detected, even from another tab, the page shows a clear banner ("Your session has ended — please login to continue") and disables stale actions like "Modify Search" until the user re-authenticates, rather than letting those actions hang indefinitely.

### Proposed User Flow — Step by Step
1. User is logged in, viewing search results
2. User logs out (in the same tab or another tab)
3. Results page immediately detects the logout via an auth-state listener
4. A clear banner appears: "Your session has ended — please login to continue browsing"
5. Action buttons like "Modify Search" are disabled or redirect to login instead of hanging
6. User clicks "Login" in the banner, re-authenticates
7. User is returned to the same results page with their search parameters intact (preserved via URL), able to continue immediately

### Technical Implementation Plan
**System components affected:**
- Frontend (global auth-state listener, results page action handlers)
- Backend API (session validation on action endpoints)
- No database schema changes needed

**New data requirements:**
- None — relies on existing session/token state, just needs to be actively checked

**API changes:**
- All action endpoints triggered from the results page (e.g. `POST /api/trains/search` via Modify Search) should return `401 Unauthorized` with `{error: "session_expired"}` immediately if the token is invalid, instead of hanging

**Frontend changes:**
- A global auth-state listener (interval-based token check, or cross-tab storage event for logout) updates the UI the moment a logout is detected
- Action buttons check auth state before sending requests; if expired, show the re-login banner instead of firing a request that will hang

**Third-party services (if any):**
- None required

### Success Metrics
- "Stuck loading after logout" failure rate drops from 100% (observed) to 0%
- Reduction in users needing a manual hard-refresh to recover from a frozen page

### Edge Cases and Constraints
- What happens if the logout was unintentional — the re-login banner should make it one click to log back in and resume, with search params persisted in the URL
- Depends on the backend correctly invalidating sessions immediately on logout rather than allowing a grace period where old tokens still validate
- Graceful degradation: if the auth-state listener itself fails to detect a logout, action endpoints still independently return 401 on stale tokens, so the user is never left in a permanently stuck state even if the frontend listener misses it

---