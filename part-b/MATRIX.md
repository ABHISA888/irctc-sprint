# Impact vs Effort Matrix

## The Matrix

|                   | Low Effort         | High Effort        |
|-------------------|--------------------|--------------------|
| **High Impact**   | - Mobile Core Actions Hub | - Tatkal Booking Queue Management<br>- Interactive Refund Timeline Tracker |
| **Low Impact**    | - Persistent Search Filters<br>- Form Autosave & Keep-Alive<br>- Sticky Passenger Preferences | *(None)* |

## How I Scored Each Dimension

### Impact Scoring (1–5)
I scored Impact based on:
- Number of users affected (from Part A frequency analysis)
- Whether the problem is in the core booking flow
- Severity of consequence for the user

### Effort Scoring (1–5)
I scored Effort based on:
- Number of system components touched
- Whether new infrastructure is required
- Risk of breaking existing flows
- Railway API dependencies

---

## Placement Justifications

### Tatkal Booking Queue Management — High Impact / High Effort
* **Impact (5/5)**: This feature directly addresses peak Tatkal crashes affecting hundreds of thousands of booking attempts daily, making it a critical improvement to the core booking flow.
* **Effort (5/5)**: Implementing this requires setting up new in-memory infrastructure (Redis cluster) and WebSocket gateways to handle real-time queues, which poses high risks of breaking current login flows.
* **Placement**: Being High Impact / High Effort, this is a major strategic project that must be carefully planned and executed as a primary engineering goal.

### URL-Synced Persistent Search Filters — Low Impact / Low Effort
* **Impact (2/5)**: While persistent filters improve daily search convenience for millions of visitors, losing filters is a minor annoyance rather than a booking-blocking failure.
* **Effort (2/5)**: The solution is client-side, requiring only Angular router state mapping without any database modifications or external API integrations.
* **Placement**: Being Low Impact / Low Effort, this is a quick win that can be easily shipped to polish the user experience.

### Sticky Passenger Preferences — Low Impact / Low Effort
* **Impact (3/5)**: This issue primarily affects a specific segment of travelers (families and senior citizens) under high-occupancy conditions, meaning it has a lower frequency than page crashes.
* **Effort (2/5)**: The technical plan requires only form state validation and minor backend API checks before checkout, with no database updates.
* **Placement**: As a Low Impact / Low Effort feature, it should be scheduled during low-traffic periods after major booking flow updates are completed.

### Unified Interactive Refund Timeline Tracker — High Impact / High Effort
* **Impact (4/5)**: This feature benefits hundreds of thousands of users dealing with transaction failures monthly, directly reducing the high volume of refund-related customer support tickets.
* **Effort (4/5)**: The effort is high because it depends on integrating with third-party payment gateway APIs (Razorpay, SBI ePay) and making schema changes to track status.
* **Placement**: This High Impact / High Effort placement means it is a key customer-satisfaction driver that should follow core booking stabilization.

### Client-Side Form Autosave & Session Keep-Alive — Low Impact / Low Effort
* **Impact (3/5)**: Session timeout data loss affects tens of thousands of users weekly, causing frustration but ultimately allowing them to re-attempt booking.
* **Effort (2/5)**: The solution relies on client-side browser storage and a single lightweight backend keep-alive endpoint, making it highly isolated and low-risk.
* **Placement**: This Low Impact / Low Effort designation makes it a simple candidate to run in parallel with larger backend tasks.

### Simplified Mobile Core Actions Hub — High Impact / Low Effort
* **Impact (5/5)**: Restructuring navigation on mobile affects millions of users daily and dramatically reduces task times for PNR and cancellation searches.
* **Effort (2/5)**: The effort is low because it is a layout and routing reorganization of the mobile homepage that does not modify the underlying booking logic.
* **Placement**: Being High Impact / Low Effort, this is the highest priority item that should be tackled first.

---

## Recommended Sprint Order

1. **Simplified Mobile Core Actions Hub**: High user impact on navigation with very low engineering effort; delivers immediate visual value.
2. **Client-Side Form Autosave & Session Keep-Alive**: Prevents form data loss with minimal client-side changes, improving form completion rates before tackling core infrastructure.
3. **URL-Synced Persistent Search Filters**: A quick, client-side routing enhancement to stabilize search list usability.
4. **Sticky Passenger Preferences**: Resolves seat allocation transparency with low-effort validation checks before payment integration begins.
5. **Tatkal Booking Queue Management (Virtual Waiting Room)**: Highly critical for system survival during peak hours, but requires dedicated sprint time for setup and load testing.
6. **Unified Interactive Refund Timeline Tracker**: High impact for customer support, but placed last due to heavy dependencies on external payment gateway webhooks.
