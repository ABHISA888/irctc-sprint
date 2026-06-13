# IRCTC Usability and Technical Problems Analysis

This document outlines the detailed investigation, reproduction steps, and root causes of key technical and user experience problems identified on the IRCTC platform.

---

## Given Problem 1: Tatkal Booking Crashes at 10:00 AM

### 1. What is Broken
Under the extreme load of Tatkal bookings at 10:00 AM (for AC classes) and 11:00 AM (for non-AC classes), IRCTC's server infrastructure fails to process incoming search queries, login requests, CAPTCHAs, and payment transactions. The frontend hangs indefinitely or crashes with generic network errors (e.g., `HTTP 503 Service Unavailable`, `HTTP 504 Gateway Timeout`, or blank screens). Crucially, there is **zero feedback during failure**—no queue position, no progress indicator, and no error messages explaining the actual status.

### 2. Who is Affected and How Many
General passengers trying to book urgent Tatkal tickets. Because millions of concurrent users access the platform simultaneously, hundreds of thousands of transactions fail daily, leaving users without tickets.

### 3. Frequency
Daily, recurring exactly during the peak booking windows:
* **09:55 AM – 10:15 AM** (AC Tatkal Quota)
* **10:55 AM – 11:15 AM** (Non-AC Tatkal Quota)

### 4. Current Flow Step-by-Step
1. **User preparation:** User opens the IRCTC website or app at 09:50 AM to prepare.
2. **User login:** User logs into their IRCTC account, enters the origin (From), destination (To), and date of travel.
3. **Quota selection:** User selects the Tatkal quota from the quota dropdown.
4. **Active waiting:** User stays on the train list or search dashboard as the clock approaches 09:59:59 AM.
5. **Window opening:** At exactly 10:00:00 AM, the user attempts to search for trains, refresh the availability, or click "Book Now" on their desired train.
6. **System freeze:** The page hangs on a loading wheel or spinner. There is no indication of whether the server is processing the request, how many users are in queue, or how long they need to wait.
7. **Failure and sell-out:** The connection times out, throws a database/gateway error, or forces a logout. By the time the user logs back in or re-initiates search, the Tatkal quota is completely exhausted (typically within 1–2 minutes).

### 5. Where Exactly It Breaks
* **Specific Step:** Step 5 & Step 6 (fetching availability and clicking "Book Now").
* **Why it breaks:** The server's API gateway (`/nget/api/v1/trains`) and checkout endpoints suffer from thread pool exhaustion and database lockouts under massive concurrent write/read requests. The lack of a rate-limiting queue or client-side feedback results in users repeatedly clicking buttons, which worsens the backend server overload.

---

## Given Problem 2: Search Filters Do Not Work Reliably

### 1. What is Broken
The search filters on the train list page (e.g., Class, Departure Time, Availability) do not work reliably. When multiple filters are applied, the list frequently fails to update correctly or displays unrelated results. Furthermore, the applied filters are **completely reset** if the user clicks a train to check details/availability and then clicks the browser's/app's "Back" button.

### 2. Who is Affected and How Many
All users searching for trains between cities, affecting millions of daily visitors who rely on custom search criteria to find appropriate trains.

### 3. Frequency
Continuous. It happens on every search session where a user applies filters and navigates back and forth between search results and train/availability details.

### 4. Current Flow Step-by-Step
1. **Initial search:** User navigates to the train search page and enters details (e.g., Delhi to Mumbai) and clicks Search.
2. **Filter application:** User opens the filters panel on the sidebar or mobile modal.
3. **Class selection:** User selects "Sleeper (SL)" class to exclude AC classes.
4. **Time selection:** User selects "Morning (06:00 - 12:00)" departure time.
5. **Rapid toggling:** User toggles "Show Available Only". The UI lags, and the train list sometimes fails to synchronize, showing incorrect trains.
6. **Detailed view:** User clicks on a specific train card to expand details and check fare/seat availability.
7. **Back navigation:** The user clicks the browser's back button or the in-app back arrow to return to the search results. All filters are cleared, and the default, unfiltered list of all trains is re-rendered, requiring the user to select filters again.

### 5. Where Exactly It Breaks
* **Specific Step:** Step 5 (during multiple filter applications) and Step 7 (navigating back).
* **Why it breaks:** The application uses client-side state management (Angular components) that does not serialize filter selections to the URL query parameters or local/session storage. As a result, when navigating back, the component is re-mounted cleanly, triggering a fresh fetch of the default results and resetting the UI state.

---

## Given Problem 3: Seat Selection Resets

### 1. What is Broken
The berth preference selected by the passenger (e.g., Lower Berth for elderly travelers) is silently ignored or reset back to "No Preference" when proceeding from the passenger details entry page to the review and payment pages. The user gets no feedback that their preference was discarded before completing the booking.

### 2. Who is Affected and How Many
Families, elderly travelers, and disabled individuals who rely on lower berths. This affects tens of thousands of travelers daily. The reset rate is significantly higher on mobile devices.

### 3. Frequency
Highly frequent. This occurs in the majority of booking attempts, particularly under high-occupancy conditions.

### 4. Current Flow Step-by-Step
1. **Train selection:** User selects a train, clicks availability, and clicks "Book Now".
2. **Passenger details form:** User is redirected to the passenger information page.
3. **Preference selection:** User enters passenger name, age, and selects "Lower Berth" from the "Berth Preference" dropdown.
4. **Additional conditions:** User checks "Book only if at least one lower berth is allotted" to ensure compliance.
5. **Submission:** User clicks "Proceed" or "Review Booking" button.
6. **State reset:** On the review journey page, the summary table displays the passenger details but lists the berth preference as "No Preference" or leaves it blank.
7. **Unwanted allotment:** The user completes the payment, and the ticket is issued with a Middle or Upper berth, ignoring the initial selection.

### 5. Where Exactly It Breaks
* **Specific Step:** Step 5 & Step 6 (proceeding from passenger entry to review).
* **Why it breaks:** The frontend passenger detail form fails to properly serialize the selected preference into the API payload during the page transition, or the backend API discards/overrides the selection based on availability constraints without notifying the frontend state. On mobile, layout responsive issues cause form elements to detach from the DOM, causing submission payloads to default to null.

---

## Problem 4: Refund Tracking Visibility

### 1. What is Broken
Users cannot easily track the progress or status of refunds for cancelled tickets or failed transactions where payment was debited. The refund tracking information is deeply buried in nested menus, lacks granular real-time progress indicators, and fails to display essential reference numbers (ARN/RRN) required to coordinate with banks.

### 2. Who is Affected and How Many
Hundreds of thousands of users monthly who cancel bookings or experience booking failures after successful payment authorization.

### 3. Frequency
Ongoing; a persistent usability limitation of the account management system.

### 4. Current Flow Step-by-Step
1. **Cancellation/Failure:** User cancels a booked ticket, or a Tatkal booking fails mid-transaction after payment deduction.
2. **Search for refunds:** User returns to the home page wanting to check the refund progress.
3. **UI navigation:** The user searches for a "Refund Status" option but find no clear button on the dashboard.
4. **Nested tabs:** User goes to "My Account" -> "My Transactions" -> "Ticket Refund History".
5. **Selection:** User selects the cancelled ticket transaction.
6. **Vague status:** The system displays a generic status message like "Refund Processed" with no bank reference number or expected date of credit.
7. **Manual tracking:** The user is left to manually inspect their bank statement or call support to verify the refund.

### 5. Where Exactly It Breaks
* **Specific Step:** Step 3 (poor information design) and Step 6 (lack of detail in refund data).
* **Why it breaks:** The refund status API is disconnected from the payment gateway's settlement APIs. Consequently, the user is presented with outdated static messages rather than a dynamic progress bar showing the stages: Cancelled -> Gateway Processing -> Bank Credited.

---

## Problem 5: Session Timeout During Long Forms

### 1. What is Broken
When a user spends time entering passenger information (especially for multiple passengers, senior citizens, or child tickets), the session silently expires in the background due to aggressive server-side timeouts. When the user completes the form and clicks "Proceed", they are redirected to the login page and all entered data is lost.

### 2. Who is Affected and How Many
Family groups, tour planners, and elderly users who take longer to fill forms. This affects tens of thousands of users weekly.

### 3. Frequency
Common, particularly during peak hours when the server aggressively reduces session lifetimes.

### 4. Current Flow Step-by-Step
1. **Start form:** User clicks "Book Now" and is taken to the passenger input page.
2. **Form entry:** User starts typing passenger names, ages, card IDs, and berth preferences for 4 to 6 people.
3. **Time consumption:** The user spends 4–6 minutes ensuring all details are accurate.
4. **Silent timeout:** The backend session times out (which has a short limit like 3 minutes during peak hours).
5. **Submission:** User clicks "Proceed" to review details.
6. **Redirect:** The app redirects the user to the login screen with a message: "Session expired. Please login again."
7. **Data loss:** Upon logging back in, the user finds the passenger details form is completely empty and must start over.

### 5. Where Exactly It Breaks
* **Specific Step:** Step 5 & Step 6.
* **Why it breaks:** The client application does not run a background session keeper, does not warn the user before the session is about to expire, and does not save the form data locally (e.g., in localStorage or sessionStorage) to allow restoring the form state after logging back in.

---

## Problem 6: Mobile Navigation Complexity

### 1. What is Broken
Critical features such as PNR status checks, refund tracking, cancellation, and customer support are hidden behind multiple layers of hamburger menus and nested navigation. The home dashboard is cluttered with promotional banners, forcing elderly and less tech-savvy users to struggle to find primary booking tasks.

### 2. Who is Affected and How Many
Millions of mobile application users, specifically elderly travelers who struggle with small UI targets and nested menus.

### 3. Frequency
Permanent, inherent to the current mobile web and app layout.

### 4. Current Flow Step-by-Step
1. **Goal initiation:** An elderly user opens the IRCTC app to check the PNR confirmation status.
2. **Dashboard landing:** The user is greeted with a home dashboard filled with ads, alerts, and small icons.
3. **Search for PNR:** The user searches the screen for a direct "PNR Status" input but cannot find one.
4. **Menu search:** The user spots and taps the small hamburger icon in the top corner.
5. **Option scrolling:** The user scrolls through a long list of text options to locate "Queries".
6. **Submenu selection:** The user selects "Queries" and then taps "PNR Enquiry".
7. **External redirection:** The user is redirected to a separate page that requires them to manually re-type their PNR.

### 5. Where Exactly It Breaks
* **Specific Step:** Step 3 (lack of prominence on home page) and Step 7 (redundant input requirements).
* **Why it breaks:** The user interface layout prioritizes secondary services and ads over high-frequency actions. The lack of context preservation also prevents logged-in users from seeing their recent PNRs directly on their dashboard.
