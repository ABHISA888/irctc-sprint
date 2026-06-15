# IRCTC Feature Specifications

This document details the feature specifications and wireframes for the six core usability and technical solutions addressing the problems identified on the IRCTC platform.

---

## 1. Tatkal Booking Queue Management (Virtual Waiting Room)

### Problem Statement
Under the extreme load of Tatkal bookings at 10:00 AM and 11:00 AM, the system fails to process transactions, resulting in indefinite loading wheels, database lockouts, or network failures (e.g., HTTP 503/504), without providing any queue feedback to the user.

### Current State
Users click the "Book Now" button immediately as the Tatkal window opens. This triggers a barrage of API requests that exhaust backend thread pools. Frontend components freeze, time out, or log the user out silently.

### Proposed Solution
Introduce a **Virtual Waiting Room** with a Token-Based Request Queue. On clicking "Book Now" during peak hours:
1. The user's request is queued in an in-memory Redis Sorted Set (`zset`).
2. The UI transitions to a Waiting Room displaying real-time queue position and estimated waiting time.
3. The booking action is locked (disabled) to prevent duplicate submissions.
4. When the queue token is processed, the session is unlocked, redirecting the user to the booking form with guaranteed seat hold for 5 minutes.

### Proposed User Flow
1. **Initiate Booking**: User clicks "Book Now" at exactly 10:00:01 AM.
2. **Queue Entrance**: The button disables, and a Virtual Waiting Room modal overlays the screen.
3. **Queue Tracking**: The modal establishes a WebSocket connection to fetch the live queue position.
4. **Active Countdown**: The user views their decreasing queue position and a progress bar.
5. **Checkout Transition**: Once the queue position hits 0, a transition animation fires.
6. **Form Access**: The user enters passenger details with a 5-minute checkout lock active.

### Technical Implementation Plan

**System Components Affected:**
* **API Gateway**: Rates limits incoming traffic and routes queue tokens.
* **Queue Service**: Manages in-memory token states using Redis.
* **Booking Service**: Resolves transactions once tokens are validated.
* **Notification WebSockets**: Streams real-time position updates to clients.

**Database Changes:**
* No relational SQL schema changes to prevent database locking.
* Redis Key: `tatkal:queue:train_id:class` (Sorted Set) storing `token_id` and `timestamp`.

**API Changes:**
* `POST /api/v1/booking/queue`
  * Payload: `{ train_number: "12626", class: "3A", quota: "CK", date: "2026-06-16" }`
  * Response: `{ status: "queued", token: "tok_8291f9", position: 12450, est_wait_seconds: 45 }`
* `GET /api/v1/booking/queue/stream?token=tok_8291f9` (WebSocket)
  * Returns: `{ position: 10120, est_wait_seconds: 36 }`

**Frontend Changes:**
* Create a dedicated `WaitingRoom` container component.
* Add rate-limiting debounce hooks to the "Book Now" button.

**Third-Party Services:**
* None.

### Success Metrics
* Reduction in API gateway 503/504 errors during peak Tatkal hours to < 1%.
* Conversion rate of entered bookings in Tatkal hours increases by 25%.
* User double-clicks/resubmissions decrease to 0.

### Edge Cases
* **Connection Drop**: The client falls back to HTTP polling every 5 seconds.
* **Queue Expiry**: If a token is processed but the user is idle for 5 minutes, the token expires, releasing the ticket hold to the next in queue.
* **Server Failover**: Redis replication ensures session tokens are preserved if a node fails.
* **Session Hijacking / Replay**: To prevent token replication or hijacking, the queue token is cryptographically bound to the user's source IP address and User-Agent using HMAC-SHA256, verified on every status poll request.

### Constraints
* Must run entirely in memory to bypass the slow relational DB during the queue phase.

---

### Desktop Wireframe
```
+-----------------------------------------------------------------------------+
|  IRCTC Header                                         [ My Account ] [ Log ]|
+-----------------------------------------------------------------------------+
|                                                                             |
|   +---------------------------------------------------------------------+   |
|   |                      Virtual Waiting Room                           |   |
|   |                                                                     |   |
|   |   Your request is in queue. Please do not refresh or close this.    |   |
|   |                                                                     |   |
|   |   [===========>---------------------------------------] 25%         |   |
|   |                                                                     |   |
|   |   Queue Position: #12,450                                           |   |
|   |   Estimated Wait Time: ~45 seconds                                  |   |
|   |                                                                     |   |
|   |   * Tip: Ensure passenger details are ready for quick entry.        |   |
|   +---------------------------------------------------------------------+   |
|                                                                             |
+-----------------------------------------------------------------------------+
```

### Mobile Wireframe
```
+-----------------------------------+
| [=] IRCTC Mobile          [Profile]|
+-----------------------------------+
|                                   |
|   +---------------------------+   |
|   |   Virtual Waiting Room    |   |
|   |                           |   |
|   |   In Queue...             |   |
|   |                           |   |
|   |   [=====>-----------] 30% |   |
|   |                           |   |
|   |   Position: #12,450       |   |
|   |   Wait Time: ~45s         |   |
|   |                           |   |
|   |   Do not close the app.   |   |
|   +---------------------------+   |
|                                   |
+-----------------------------------+
```

### Components
* `WaitingRoomModal`: Centered overlay modal blocking user input.
* `ProgressBar`: Smooth animated progress bar showing queue completeness.
* `QueueText`: Label indicating current position (e.g. `Position: #12,450`) and time (e.g. `Wait Time: ~45s`).
* `StatusMessage`: Rotating text showing system tips.

### Loading State
* Shows a rotating loading ring with the message: "Securing your spot in queue..."

### Error State
* Standard alert box: "Connection lost. Reconnecting to queue..." with a manual retry button.

### Empty State
* Not applicable (the waiting room is active only when a token is in the queue).

---

## 2. URL-Synced Persistent Search Filters

### Problem Statement
When search filters (Class, Departure Time, Availability) are applied, they fail to update reliably due to frontend lags. Additionally, when a user checks a train's details and navigates back, all filters are wiped out, forcing them to re-apply them.

### Current State
Filter states are saved in localized Angular component memory. Navigating back destroys the component, resetting all variables to defaults and triggering a new unfiltered search.

### Proposed Solution
Sync all search filter states directly with the URL query parameters. This keeps the URL as the single source of truth, allowing seamless back-and-forth browser navigation. Secondary caching using `sessionStorage` ensures UI elements preserve their state even if the URL is truncated.

### Proposed User Flow
1. **Enter Search**: User searches trains from Delhi to Mumbai. URL: `/search?from=NDLS&to=BCT`.
2. **Apply Filters**: User checks "Sleeper (SL)" and "Morning".
3. **URL Update**: URL dynamically updates to `/search?from=NDLS&to=BCT&class=SL&dept=morning`. The list filters reactively.
4. **View Details**: User clicks to expand availability for a specific train.
5. **Back Navigation**: User clicks the browser "Back" button.
6. **State Restored**: The application parses the URL query parameters, re-checks the "Sleeper" and "Morning" checkboxes, and displays the filtered list instantly.

### Technical Implementation Plan

**System Components Affected:**
* **Frontend Router**: Evaluates and updates URL query parameters.
* **Search Filter Component**: Intercepts check events and pushes query states.
* **Search Controller**: Listens to routing events to filter lists.

**Database Changes:**
* None.

**API Changes:**
* No backend changes needed as filtering is handled client-side or maps directly to query parameters of the existing `/trains` endpoint.

**Frontend Changes:**
* Create a `QueryParamSyncService` to serialize/deserialize the filter object.
* Bind filter checkbox models directly to URL state.

**Third-Party Services:**
* None.

### Success Metrics
* Time-to-find-train decreases by 30%.
* Average number of filter interaction clicks per user session drops by 60%.
* No filter loss reports on back navigation.

### Edge Cases
* **Invalid URL Parameters**: If `class=INVALID` is entered, the router sanitizes the parameter, falls back to "All Classes", and updates the URL.
* **Direct Share Link**: The shared URL retains all filters, letting other users see the identical filtered list.

### Constraints
* URL length constraints on older browsers (though modern query params fit well within limits).

---

### Desktop Wireframe
```
+-----------------------------------------------------------------------------+
|  IRCTC Header                                         [ My Account ] [ Log ]|
+-----------------------------------------------------------------------------+
|  URL: irctc.co.in/search?from=NDLS&to=BCT&class=SL&dept=morning             |
+-----------------------------------------------------------------------------+
|  Filters Sidebar            | Train Search Results (NDLS -> BCT)            |
|  [x] Sleeper (SL)           | +-------------------------------------------+ |
|  [ ] AC 3 Tier (3A)         | | Rajdhani Express (12952)                  | |
|                             | | DEP: 06:15 AM | ARR: 08:30 PM             | |
|  Departure Time             | | [ Check Availability ]                    | |
|  [x] Morning (06:00-12:00)  | +-------------------------------------------+ |
|  [ ] Afternoon (12:00-18:00)| | Garib Rath (12910)                        | |
|                             | | DEP: 08:10 AM | ARR: 10:20 PM             | |
|  [ Clear All ]              | | [ Check Availability ]                    | |
+-----------------------------+-----------------------------------------------+
```

### Mobile Wireframe
```
+-----------------------------------+
| [=] IRCTC Mobile  [URL: /?class=SL]|
+-----------------------------------+
| [ Filter (2 Applied) ]  [ Sort ]  |
+-----------------------------------+
|  Train Results (2)                |
|  +-----------------------------+  |
|  | Rajdhani Express (12952)    |  |
|  | DEP: 06:15 AM               |  |
|  | SL | [ Check Availability ] |  |
|  +-----------------------------+  |
|  +-----------------------------+  |
|  | Garib Rath (12910)          |  |
|  | DEP: 08:10 AM               |  |
|  | SL | [ Check Availability ] |  |
|  +-----------------------------+  |
+-----------------------------------+
```

### Components
* `FilterCheckbox`: Input elements bound to URL query params.
* `ActiveFilterTags`: Chips showing active selections (e.g. `[SL x]` `[Morning x]`).
* `ClearAllButton`: Button to purge all query parameters.

### Loading State
* Filter list displays a skeleton loader while results are being filtered.

### Error State
* Banner: "Failed to apply filters. [Retry]"

### Empty State
* Centered illustration: "No trains match the selected filters. [Clear Filters]"

---

## 3. Sticky Passenger Preferences & Selection Guard

### Problem Statement
Passenger berth preferences (e.g., Lower Berth for elderly travelers) are silently reset or ignored during form transitions to checkout, resulting in unwanted allotments without user awareness.

### Current State
Form serialization fails to carry preference selections to the payment step. Mobile layout bugs cause elements to drop from the DOM, defaulting the request payload to "No Preference".

### Proposed Solution
Add client-side React/Angular state persistence for form details, strict validation, and a **Selection Guard Modal**. If a user checks a strict constraint (e.g., "Book only if lower berth is allotted") and the system determines it cannot be fulfilled before proceeding to payment, a modal warns the user instead of silently failing.

### Proposed User Flow
1. **Form Entry**: User fills details and selects "Lower Berth" + "Book only if lower berth is allotted".
2. **Review Screen**: The summary panel highlights "Lower Berth Preference: Strict".
3. **Availability Evaluation**: The system checks availability.
4. **Allotment Warning**: If unavailable, the Selection Guard Modal triggers.
5. **Decisive Choice**: The user either confirms random assignment or cancels the flow.

### Technical Implementation Plan

**System Components Affected:**
* **Passenger Form Module**: Handles form state and validations.
* **Checkout Engine**: Verifies reservation rules prior to payment initiation.

**Database Changes:**
* None.

**API Changes:**
* `POST /api/v1/booking/validate-seats`
  * Payload: `{ passengers: [{ name: "A. Kumar", age: 68, preference: "LB" }], strict: true }`
  * Response: `{ status: "warning", message: "Lower berth not available. Proceed with middle/upper?" }`

**Frontend Changes:**
* Build a responsive `BerthPreferenceSelector` dropdown that locks state in a Redux store.
* Implement the `SelectionGuardModal` component.

**Third-Party Services:**
* None.

### Success Metrics
* Support tickets for seat allocation complaints drop by 50%.
* Zero instances of users completing booking unaware that their preferences were ignored.

### Edge Cases
* **Mixed Passengers**: One senior citizen (needs Lower Berth) and one youth (no preference). The guard validates only the strict seat and warns specifically: "1/2 preferences could not be met."

### Constraints
* Indian Railways database limits actual allocation based on real-time availability; the solution focuses on *transparency* and *user choice* rather than forcing seat allocation.

---

### Desktop Wireframe
```
+-----------------------------------------------------------------------------+
|  IRCTC Header                                         [ My Account ] [ Log ]|
+-----------------------------------------------------------------------------+
|   Passenger Review Summary                                                  |
|   +---------------------------------------------------------------------+   |
|   | Name: Abhinivesh S. | Age: 67 | Preference: Lower Berth (Strict)    |   |
|   +---------------------------------------------------------------------+   |
|                                                                         |
|   +---------------------------------------------------------------------+   |
|   | ! Preference Warning                                                |   |
|   | Lower berths are currently unavailable on this train.               |   |
|   | Proceeding will allocate the next available berth (Middle/Upper).   |   |
|   |                                                                     |   |
|   |           [ Continue with Alternative ]    [ Go Back & Change ]     |   |
|   +---------------------------------------------------------------------+   |
+-----------------------------------------------------------------------------+
```

### Mobile Wireframe
```
+-----------------------------------+
| [=] IRCTC Mobile          [Review]|
+-----------------------------------+
|  Review Details:                  |
|  A. Kumar, 67, Lower Berth        |
|  +-----------------------------+  |
|  | ! Berth Unfulfillable       |  |
|  | No lower berths left.       |  |
|  | Proceed with other seats?   |  |
|  |                             |  |
|  |   [ Book Other ] [ Cancel ] |  |
|  +-----------------------------+  |
+-----------------------------------+
```

### Components
* `PreferenceSummaryBadge`: Visual indicator showing preference status.
* `SelectionGuardModal`: Warning popup intercepting payment actions.
* `ActionButtons`: Primary button to override and secondary to cancel.

### Loading State
* Disabled checkout button showing "Checking seat availability..."

### Error State
* Standard inline message: "Cannot verify seat availability. [Retry]"

### Empty State
* Not applicable.

---

## 4. Unified Interactive Refund Timeline Tracker

### Problem Statement
Refund statuses for failed or cancelled transactions are hidden deep within multiple levels of nested menus and show only static, vague messages without ARN/RRN codes.

### Current State
Users must navigate: Account -> Transactions -> Ticket Refund History. The status shows a static "Refund Processed", leaving the user to manually verify with their bank.

### Proposed Solution
Create an **Interactive Refund Timeline Widget** prominently placed on the primary dashboard. This widget displays a step-by-step progress bar (Cancellation -> Gateway -> Bank Processing -> Credited) and lists bank reference numbers (ARN/RRN) with one-click copy buttons.

### Proposed User Flow
1. **Dashboard Entry**: User logs in and immediately sees a "Recent Refund" card on the main dashboard.
2. **Timeline Interaction**: User clicks "Track Refund".
3. **Step View**: A visual timeline shows:
   * [Tick] Cancelled (June 14)
   * [Tick] Gateway Processed (June 14)
   * [Active] Bank Processing (Expected Credit: June 17, ARN: 827182910)
4. **Copy Reference**: User clicks "Copy ARN" to resolve any delays directly with their bank.

### Technical Implementation Plan

**System Components Affected:**
* **Dashboard Interface**: Displays the quick-access widget.
* **Payment Settlement API**: Aggregates status from payment gateways (Razorpay, SBI, Paytm).

**Database Changes:**
* Booking Table updates: Add `refund_arn` (VARCHAR), `refund_step` (INT), and `expected_refund_date` (DATE).

**API Changes:**
* `GET /api/v1/refunds/track?pnr={pnr_number}`
  * Response: `{ pnr: "4281920192", refund_amount: 1420.00, steps: [{ step: 1, name: "Cancelled", status: "completed", date: "2026-06-14" }, { step: 2, name: "Gateway", status: "completed", date: "2026-06-14" }, { step: 3, name: "Bank Processing", status: "active", date: "2026-06-15", arn: "827182910" }] }`

**Frontend Changes:**
* Add `RefundTimelineWidget` to dashboard templates.
* Add a "Copy to Clipboard" utility button.

**Third-Party Services:**
* Payment Gateway refund status webhooks.

### Success Metrics
* Ensure 95% of active refunds display a bank ARN within 48 hours of cancellation, resulting in a 45% reduction in refund-related customer support queries.
* Average time spent finding refund status drops from 45 seconds to 2 seconds.

### Edge Cases
* **Missing ARN**: If the bank hasn't generated the ARN, the timeline shows: "Bank Processing (ARN will generate in 24 hours)".
* **Partial Refund**: If multi-passenger cancellation occurs, show split cards for each cancelled seat.

### Constraints
* Subject to payment gateway API response times.

---

### Desktop Wireframe
```
+-----------------------------------------------------------------------------+
|  IRCTC Header                                         [ My Account ] [ Log ]|
+-----------------------------------------------------------------------------+
|  Track Refund: PNR 4281920192                                               |
|  Refund Amount: Rs 1,420.00                                                 |
|                                                                             |
|   Cancelled        Gateway Refunded       Bank Processing       Credited    |
|     (June 14)          (June 14)             (Active)         (Est. June 17)|
|      [ O ] ------------ [ O ] -------------- [ O ] ----------- [   ]        |
|                                                |                            |
|                                                +-- ARN: 827182910  [Copy]   |
|                                                                             |
|   Need Help? [ Contact Support with ARN ]                                   |
+-----------------------------------------------------------------------------+
```

### Mobile Wireframe
```
+-----------------------------------+
| [=] IRCTC Mobile          [Refund]|
+-----------------------------------+
|  PNR: 4281920192 (Rs 1,420.00)    |
|                                   |
|  [x] Cancelled (June 14)          |
|  [x] Gateway Sent (June 14)       |
|  [-] Bank Processing (Active)     |
|      ARN: 827182910 [Copy]        |
|  [ ] Credited (Est. June 17)      |
|                                   |
|   [ Call Bank ]  [ Call Support ] |
+-----------------------------------+
```

### Components
* `RefundStepNode`: State icons indicating timeline completion.
* `ArnCopyButton`: Clipboard interaction element.
* `HelpButton`: Direct query link sending ARN info to support.

### Loading State
* Shimmering timeline placeholders loading status.

### Error State
* Banner: "Could not retrieve refund updates. [Retry]"

### Empty State
* Screen: "No recent cancellations found on this account."

---

## 5. Client-Side Form Autosave & Session Keep-Alive

### Problem Statement
Aggressive server-side timeouts during form filling cause users to lose all passenger information on submit, redirecting them to login with empty fields.

### Current State
Backend session limits are short during peak times. The client has no keep-alive ping or storage cache, resulting in complete form clears upon timeout.

### Proposed Solution
Implement:
1. **Interactive Session Keep-Alive**: Pop up a countdown warning 60 seconds before session expiry, allowing users to extend their session.
2. **Form Autosave**: Encrypt and cache form inputs in `localStorage` in real-time. If the session expires, the data is restored instantly upon logging back in.

### Proposed User Flow
1. **Start Form**: User enters passenger details.
2. **Expiry Warning**: At 60 seconds remaining, a banner warns the user.
3. **Extend Action**: User clicks "Extend", firing a light ping to refresh the backend timer.
4. **Timeout Event**: If idle, session expires. Data is cached locally before redirection.
5. **Restore Flow**: User logs back in. A top banner offers: "Restore your unsaved form? [Restore]".

### Technical Implementation Plan

**System Components Affected:**
* **Frontend Session Manager**: Handles expiry timers.
* **Storage Coordinator**: Encrypts and saves draft states.
* **Authentication Gateway**: Intercepts keep-alive calls.

**Database Changes:**
* None.

**API Changes:**
* `POST /api/v1/auth/keep-alive`
  * Action: Resets session TTL in Redis.
  * Response: `{ status: "success", session_ttl_seconds: 180 }`

**Frontend Changes:**
* Implement AES-256-GCM client-side encryption (via Web Crypto API) to encrypt drafts in localStorage, deriving the encryption key dynamically from the active session cookie so data is unreadable offline.
* Build `SessionWarningModal` and `DraftRestoreBanner`.

**Third-Party Services:**
* None.

### Success Metrics
* Form completion rates increase by 15%.
* Incomplete form submissions due to timeout decrease to 0.

### Edge Cases
* **Shared/Public Devices**: Drafts are purged automatically after 15 minutes of inactivity to protect personal identification details.

### Constraints
* Browser must have local storage enabled.

---

### Desktop Wireframe
```
+-----------------------------------------------------------------------------+
|  IRCTC Header                                         [ My Account ] [ Log ]|
+-----------------------------------------------------------------------------+
|                                                                             |
|   +---------------------------------------------------------------------+   |
|   | ! Session Expiry Warning                                            |   |
|   | Your booking session will expire in 45 seconds due to inactivity.   |   |
|   | Would you like to extend your session and keep your booking details?|   |
|   |                                                                     |   |
|   |                 [ Extend Session ]     [ Cancel Booking ]           |   |
|   +---------------------------------------------------------------------+   |
|                                                                             |
+-----------------------------------------------------------------------------+
```

### Mobile Wireframe
```
+-----------------------------------+
| [=] IRCTC Mobile          [Form]  |
+-----------------------------------+
|  +-----------------------------+  |
|  | ! Session Timeout (45s)     |  |
|  | Keep booking active?        |  |
|  |                             |  |
|  |  [ Keep Active ]  [ Exit ]  |  |
|  +-----------------------------+  |
|  Passenger details...             |
+-----------------------------------+
```

### Components
* `SessionWarningModal`: Fullscreen modal with countdown timer.
* `AutosaveToast`: Subtle banner confirming "Draft saved".
* `RestoreDraftBanner`: Dashboard banner shown post-relogin.

### Loading State
* "Extending session..." button spinner.

### Error State
* Message: "Could not extend session. Please copy details before redirecting."

### Empty State
* Not applicable.

---

## 6. Simplified Mobile Core Actions Hub

### Problem Statement
Mobile navigation is cluttered with marketing and promotional banners. Primary actions like checking PNR status, cancels, and refunds are buried deep inside small hamburger submenus.

### Current State
Mobile homepage places promotional ads at the top. Accessing PNR requires hamburger menu navigation to Queries -> PNR Enquiry, redirecting to an external form.

### Proposed Solution
Re-layout the mobile landing page to introduce a **Quick Actions Hub** (a grid of 4 core tasks: PNR, Cancel, Refund, Support) and a persistent **Bottom Navigation Bar** for primary application modules.

### Proposed User Flow
1. **Open App**: User lands on the homepage.
2. **Immediate Visibility**: High-contrast grid features "PNR Status", "Cancel Ticket", "Refund Tracker", and "Support".
3. **Bottom Navigation**: Persistent bar at the bottom provides tabs for Home, Book, Tickets, and Profile.
4. **Action Access**: Tapping PNR Status reveals an in-app overlay displaying recent PNRs for instant lookup.

### Technical Implementation Plan

**System Components Affected:**
* **Mobile Home Layout**: Grid container and navigation restructuring.
* **PNR Service**: Local device history aggregation.

**Database Changes:**
* None.

**API Changes:**
* None.

**Frontend Changes:**
* Implement persistent `BottomNavigationBar` component.
* Replace top layout sections with a mobile-optimized `QuickActionsHub` widget.

**Third-Party Services:**
* None.

### Success Metrics
* Click-through rate for PNR checks increases by 60%.
* Average task-completion time on mobile drops from 40 seconds to 5 seconds.

### Edge Cases
* **Guest Users**: Display the hub but prompt login for "Cancel Ticket" and "Refund Tracker". PNR is open to all.

### Constraints
* Core marketing banners must be relocated to a smaller carousel at the page footer to respect business ad commitments.

---

### Desktop Wireframe
*(Not applicable - Mobile Layout specific feature. Mobile layout takes priority as shown below.)*

### Mobile Wireframe
```
+-----------------------------------+
| [=] IRCTC Mobile         [Support]|
+-----------------------------------+
|  [ From: NDLS     ]  [ To: BCT ]  |
|  [ Date: 16-06-2026            ]  |
|  [ Search Trains               ]  |
|                                   |
|  Quick Actions:                   |
|  +-----------------------------+  |
|  | [ PNR ]   | [ Cancellations]|  |
|  |-----------+-----------------|  |
|  | [ Refund ]| [ Cust Care ]   |  |
|  +-----------------------------+  |
|                                   |
|  +-----------------------------+  |
|  | Ad Banner (Relocated Below) |  |
|  +-----------------------------+  |
|                                   |
|  +-----------------------------+  |
|  | HOME |  BOOK  | PNR  | PROFILE|  | (Persistent Bottom Bar)
|  +-----------------------------+  |
+-----------------------------------+
```

### Components
* `BottomNavBar`: Navigation bar stuck to the bottom viewport.
* `QuickActionGridItem`: High-contrast icon buttons.
* `AdFooter`: Relocated advertising area below core controls.

### Loading State
* Skeleton cells rendering grid items.

### Error State
* Icon placeholder with text: "Service unavailable."

### Empty State
* Not applicable.
