# IRCTC Part B — Feature Specifications

6 comprehensive feature specifications addressing the 6 problems documented in Part A. Each spec bridges discovery and delivery: problem statement, proposed solution, technical plan, success metrics, and edge cases.

---

## Feature Spec 1: Tatkal Virtual Queue System

### Problem Statement
IRCTC receives 20–40 lakh concurrent requests at 10:00 AM when Tatkal quota opens, causing server overload, session crashes, and zero user feedback during failure. 60–70% of users who attempt Tatkal booking fail due to server-side errors, not quota unavailability. The current booking flow gives no queue state or progress indication, causing users to refresh and amplify the load further.

### Current State (from Part A)
The existing Tatkal flow allows direct hits to the seat selection endpoint at quota opening. Users can reach the booking screen, enter passenger details, and still lose the booking at the final step because the system either freezes, throws an HTTP 502, times out, or resets the session. Steps 5–7 in the current flow show the failure: the page freezes with a spinner, the user sees a gateway error or timeout, and after refresh the session is often logged out with no payment confirmation.

### Proposed Solution
Replace the direct-hit booking flow with a virtual waiting room. Users who visit the Tatkal section between 9:55–10:05 AM are assigned a queue position. At 10:00 AM, positions are served in order. Each user sees their live queue number, estimated wait time, and a countdown. When their turn arrives, they get a 90-second booking window. If unused, the slot passes to the next user.

### Proposed User Flow — Step by Step
1. User navigates to Tatkal booking section before 10:00 AM
2. System immediately assigns the user a queue position and displays it: "Tatkal opens in 4:23 — You are in queue. Position: 4,281"
3. Live countdown shows estimated wait time: "Est. wait: ~9 minutes"
4. At 10:00 AM the queue begins processing — 500 users served per minute (backend auto-adjusts based on seat availability)
5. UI shows live position updates: "Position 4,281 → 3,100 → 1,205 → 89 → YOUR TURN — 90 seconds to complete"
6. When user's turn arrives, the seat map pre-populates with the user's saved preferences and seat selection history
7. User selects or confirms the auto-populated seat, enters or confirms passengers, and completes payment within 90 seconds
8. Booking confirmation screen appears immediately upon payment success
9. If 90s expires without action, the slot is released and the user is offered "Next available in ~3 minutes — Re-queue?" with option to join again
10. User has option to exit queue at any time without penalty

### Technical Implementation Plan

**System components affected:**
- Backend API layer: new queue management endpoints and Redis-backed queue
- WebSocket/real-time server: live position push to clients
- Session management: queue token linked to booking session
- Frontend UI: new queue screen, real-time counter component, 90-second timer
- Railway seat API integration: pre-fetch availability at queue time
- Payment gateway: single-transaction model per booking attempt

**New data requirements:**
- Redis sorted sets: queue ID, user session, position, timestamp, status (waiting / assigned / completed / expired)
- Session table addition: queue_position_id, queue_join_time, booking_window_expiry
- User preferences cache: last_selected_seat, last_passengers, preferred_class

**API changes:**
```
POST /tatkal/queue/join 
  → Returns: { queueId, position, estimatedWaitSeconds }
  
GET /tatkal/queue/status/:queueId 
  → Returns: { position, eta, bookingWindowExpiryAt, status }
  
WebSocket channel: tatkal:queue:[queueId] 
  → Emits: { position, eta, status, bookingWindowExpiryAt } every 1-2 seconds

POST /tatkal/booking/confirm
  → (Existing, but now throttled to 500 concurrent calls/minute at queue service level)
```

**Frontend changes:**
- New `TatkalQueueScreen` component: displays position, countdown, estimated time                                  
- Real-time counter component: subscribes to WebSocket, updates position live
- 90-second countdown timer: visual urgency (progress bar, color shift from green → red)
- Pre-fetch seat map data in background while user is in queue
- Auto-fill passenger details from saved profile when slot arrives
- Queue exit confirmation modal: "Are you sure? You will lose your position."

**Third-party services (if any):**
- Redis (in-memory queue store): high throughput, O(log N) insertion/polling
- Socket.io or native WebSocket: real-time position push
- Railway seat availability API: called once per queue join to validate train/class still available

### Success Metrics
- Tatkal booking completion rate during peak: 40% → 70% within 30 days
- Server error rate at 10:00 AM: 35% → less than 5%
- User satisfaction score for Tatkal flow: 2.1/5 → 4.2/5
- Queue-related complaints on social media and support: reduce by 80%
- Average booking time (from queue entry to confirmation): stays under 5 minutes for 90%+ users

### Edge Cases and Constraints
- **Railway backend constraint:** Seat reservation API must still be called at booking confirmation time (external system, cannot change). If railway API is down, queue halts gracefully with user-facing message: "Booking system temporarily unavailable. Your queue position is saved — check back in 5 minutes."
- **Queue persistence — 3-tier fallback strategy:**
  - **TIER 1 (Normal):** Redis-backed queue with WebSocket push; 500 users/min throughput; <100ms latency
  - **TIER 2 (Redis down):** Fall back to database-backed queue with 2-second HTTP polling; 100 users/min throughput; graceful degradation message: "Queue processing slower due to system load"
  - **TIER 3 (Both down):** Halt new queue entries; show users "System overloaded. Tatkal will open at [10:15 AM]. Queue will resume when capacity restored." This prevents cascading failures and crashes during peak load.
- **Rollback plan if queue fails after 10 AM:** Kill load balancer routing to queue endpoints within 5 minutes; send all traffic to direct booking with message "Queue temporarily paused. Try booking directly." Resume queue 10 minutes later if capacity is restored. This prevents a "thundering herd" of retry requests crashing the system.
- **2G/3G fallback:** WebSocket fallback to HTTP polling every 5 seconds instead of real-time push.
- **User navigation away:** If user closes tab/browser while in queue, re-entry within 10 minutes restores their position. After 10 minutes, position expires and queue is re-joined.
- **Concurrent bookings:** User cannot be in multiple queue positions simultaneously. Joining a new queue auto-exits the previous one.
- **Load shedding:** If queue length exceeds 50 lakh users, new joins are directed to a waitlist with status "High demand — try again in 30 minutes."
- **Payment timeout:** If payment does not complete within 90 seconds, a grace period of 10 additional seconds is given with a prompt. If still not complete, slot is released.

---

## Feature Spec 2: Search Filter Persistence and Reliability

### Problem Statement
The train results filter panel is inconsistent. Filters such as class, quota, and availability can reset or show results that do not match the selected state, especially after a refresh or results reload. This affects all users, particularly senior citizens and first-time users who depend on filters to narrow down options. The problem stems from filter state not being reliably preserved in session storage or tied to URL parameters.

### Current State (from Part A)
When a user enters source, destination, and date, search results show a long list. The user applies filters (Sleeper Class, Available quota only), but the page reload or availability refresh loses the filter state. The user sees waitlisted trains still present despite selecting "Available only." When the user goes back to the filter panel, it has reset to "All Classes" and "All Quotas." The fundamental issue is that filter state lives only in component memory, not in session storage or URL parameters, so navigation away and back loses it completely.

### Proposed Solution
Persist filter state to URL query parameters and session storage. When the user applies a filter, the URL is updated immediately (e.g., `?class=sleeper&quota=tatkal&availability=available`). Session storage backs this up. When the user refreshes, navigates back, or the page reloads results, the filters are automatically reapplied from the URL/session. The UI always reflects the active filter state in the filter panel and result labels.

### Proposed User Flow — Step by Step
1. User enters source, destination, and date on the home page and clicks Search
2. Search results load with all trains (no filters yet applied)
3. User opens the filter panel and selects: Class = Sleeper, Quota = Available, Departure Time = 18:00–23:00
4. URL updates immediately to `/search?src=CHE&dst=DEL&date=2026-05-15&class=sleeper&quota=available&time=18-23`
5. Results re-render instantly, showing only 12 trains matching the filters
6. Each filter chip is visible and removable ("Sleeper ✕", "Available ✕", "18:00–23:00 ✕")
7. User clicks a train to view details
8. User navigates back to results — filters are intact, results still show 12 trains
9. User refreshes the page — filters remain applied (URL preserved)
10. User shares the URL with a friend — friend opens the link and sees the same filtered results
11. User can click the ✕ on any filter chip to remove it individually, or "Clear All Filters" to reset

### Technical Implementation Plan

**System components affected:**
- Frontend search results page: filter state management, URL synchronization
- Session storage: persist filters as backup
- Search API layer: already supports filter parameters, ensure all are being honored on backend

**New data requirements:**
- Session storage object: `{ appliedFilters: { class, quota, availability, departure_time, arrival_time, price_range } }`
- URL query parameters: standardized parameter names matching backend API expectations

**API changes:**
```
GET /search/trains?src=CHE&dst=DEL&date=2026-05-15&class=sleeper&quota=available&departure_after=18:00&departure_before=23:00
  → Returns: filtered results (backend must respect all filter params and return only matching trains)
  → Ensure response includes applied_filters object for client-side validation
```

**Frontend changes:**
- Filter panel component: sync state to URL params and session storage on every change
- URL parameter watcher: re-fetch results and update UI whenever URL query params change (e.g., browser back button)
- Result list component: display active filters as removable chips above results
- Fetch trains once with all filters in URL, then render
- Add "View Filter Logic" link: shows user which filters are active and why a train matched/did not match

**Third-party services (if any):**
- None (uses existing IRCTC backend API)

### Success Metrics
- **Filter adoption rate:** % of search sessions where at least 1 filter is applied
  - Baseline: 35% → Target: 55% (tracked via URL query params + event logs on filter chip clicks)
  
- **Filter reset issue rate:** Support tickets mentioning "filters reset" or "filter state lost"
  - Baseline: 12 tickets/day → Target: <1 ticket/day within 14 days
  
- **"Clear Filters" click rate:** Signal of user confusion; should remain low
  - Baseline: 12% of filtered searches → Target: <5%
  - High rate = users getting frustrated and resetting vs. intentionally clearing
  
- **Average search completion time** (home → results → filtered view): 
  - Baseline: 90 seconds → Target: 45 seconds (measured via session recording analytics)
  
- **URL sharing with filters:** Indirect proxy for filter reliability
  - Baseline: 2% of searches → Target: 8% (tracked via URL shortener clicks)
  
- **Mobile user bounce rate on search page:** 35% → 18%
  
- **Passive user satisfaction (social + review sentiment):** Mentions of "results match filters"
  - Baseline: 30% positive mentions → Target: 85%+ positive sentiment

### Edge Cases and Constraints
- **Contradictory filters:** If user selects "Class = AC" and "Quota = Reserved" but no trains exist in that combination, show: "0 trains match your filters. Remove a filter to broaden results." Offer suggestions: "Remove quota filter to see 8 AC trains."
- **Large result sets:** If a broad search returns >1000 trains, apply pagination (50 per page). Filters remain active across all pages.
- **Availability churn:** Seats are released and booked constantly. If a train had seats when the user filtered but is now full, show: "This train sold out. Remove 'Available' filter to see waitlist options."
- **URL length limit:** If multiple filters create a very long URL, use a URL shortener fallback or session ID in place of full params.
- **Browser back button:** Ensure browser back button restores the previous filter state (use browser history API).
- **Filter timeout:** If a user's session expires, filter state is lost but the URL is still valid — next login can re-apply the same URL.

---

## Feature Spec 3: Seat Selection State Persistence

### Problem Statement
The selected berth or seat sometimes disappears between the seat map and the passenger details step. A user can choose a lower berth, continue forward, and still end up with Auto assignment or a different berth in the next screen. This affects families, elderly passengers, people with disabilities, and anyone with strong berth preference. The issue is more common on mobile than desktop and stems from the seat state not being tied to the booking session token.

### Current State (from Part A)
After the user selects a train and class, the seat map loads and shows berth availability. The user selects a specific lower berth and clicks Proceed. On the passenger details page, the selected seat disappears — the system shows "Auto" or a different berth. When the user goes back to reselect the seat, it may now appear unavailable. This creates a loop and eventually forces the user to accept a less suitable berth. The root cause is that seat selection lives only in client-side component state, not in the backend booking session.

### Proposed Solution
Store the selected seat in the backend booking session immediately after user selection, not just in frontend component state. When the user proceeds to passenger details, the backend pre-populates the selected seat and carries it forward through the booking flow. If the seat becomes unavailable due to a competing booking, show the user a choice: "Your lower berth was just booked. Here are 2 alternatives with similar properties. Choose one or re-select manually."

### Proposed User Flow — Step by Step
1. User selects a train and class
2. Seat map loads; user examines berths
3. User clicks on a specific lower berth — seat is highlighted immediately
4. An API call is made in the background: `POST /booking/reserve-seat` with the selected berth ID
5. Backend returns confirmation: `{ reservedSeatId, expiryAt: now + 10 minutes }`
6. UI shows "Lower Berth #35 Reserved for 10 minutes ⏱️"
7. User clicks Proceed to Passenger Details
8. Passenger details screen pre-populates with the reserved berth ID
9. User confirms or modifies passenger name, age, etc.
10. User clicks Proceed to Payment
11. Payment screen confirms: "Seat: Lower Berth #35"
12. At booking confirmation, the reserved seat is locked to the booking
13. If the seat expires (10 min timeout) or is taken by another user, system shows: "Your reserved seat is no longer available. 2 alternatives found. [View Alternatives] [Re-select Manually]"

### Technical Implementation Plan

**System components affected:**
- Backend booking session table: add `reserved_seat_id`, `reserved_seat_expiry` columns
- Seat reservation API: new endpoint to reserve a seat (10-minute hold)
- Frontend seat selection component: immediately call reserve-seat API on click
- Frontend passenger details page: fetch and display reserved seat
- Payment processing: validate seat is still reserved at payment confirmation time

**New data requirements:**
- `BookingSession` table additions: `reserved_seat_id (UUID)`, `reserved_seat_expiry (timestamp)`, `reserved_seat_alternative_suggestions (JSON)`
- `SeatReservation` temporary table: tracks all 10-minute seat holds across the platform (for deduplication)

**API changes:**
```
POST /booking/reserve-seat
  Body: { trainId, journeyDate, class, coachNumber, seatNumber }
  → Returns: { reservedSeatId, seatNumber, expirySeconds: 600, lockToken }

GET /booking/session/:sessionId/reserved-seat
  → Returns: { reservedSeatId, seatNumber, coachNumber, expirySeconds }

GET /booking/seat-alternatives/:reservedSeatId
  → Returns: [ { seatId, seatNumber, distance_from_original, berth_type }, ... ]
```

**Frontend changes:**
- Seat map component: on seat click, immediately call `POST /booking/reserve-seat` instead of just updating local state
- Show inline "Reserving..." spinner on clicked seat
- Passenger details page: fetch reserved seat info on mount, display seat lock status and expiry countdown
- Countdown timer: if expiry < 2 minutes, show warning: "Seat reservation expires in 1:45. Complete your booking to lock it in."
- Seat alternatives modal: if reservation expires, offer quick-pick alternatives or manual re-selection

**Conflict Resolution Strategy (Post-Peer-Review Addition):**

When a reserved seat is no longer available at payment confirmation time, follow this flow to avoid charging users for failed bookings:

1. **Detection:** At payment confirmation, validate reserved seat is still available (call Railway API to confirm)
2. **If seat still available:** Proceed with normal payment + booking
3. **If seat NO LONGER available:**
   - DO NOT charge user any payment; stop at pre-payment validation
   - Show modal: "Your selected seat is no longer available. Choose an alternative or cancel this booking (no charges will be applied)."
   - Offer 3 alternatives from same class/coach if possible
   - If user chooses alternative: reserve new seat immediately (new POST /booking/reserve-seat call)
   - If user cancels: session ends, no charges incurred
4. **Grace period for stale data:** If Railway API latency is high (5+ seconds), cache seat availability for 3 seconds. If a seat shows "taken" in cache but was just reserved by user, give 30-second grace period: "This seat is being finalized. Completing payment now will secure it." Display 30-second countdown.

This prevents the #1 user complaint: being charged for a booking that failed due to seat unavailability.

**Third-party services (if any):**
- None (uses IRCTC backend only)

### Success Metrics
- Seat reset complaints: reduce by 85%
- "Wrong berth received" refund requests: reduce by 75%
- Booking completion rate (after seat selection): 78% → 89%
- Mobile user booking completion rate: 65% → 78%
- Average time from seat selection to payment: 3 minutes → 5.5 minutes (slightly longer but more reliable)

### Edge Cases and Constraints
- **Concurrent bookings:** If another user books the same seat while first user is in passenger details, the system must detect this before payment. Show: "Your reserved seat was just booked by another user. [View 3 Alternatives] [Cancel and Re-Select]"
- **10-minute timeout:** Seat reservation expires after 10 minutes. If user is slow on passenger details, show countdown and final reminder: "Reservation expires in 30 seconds. Tap here to extend 5 more minutes or save passenger profile to book faster next time."
- **Mobile network loss:** If user loses connection while seat is reserved, the reservation persists on backend. When user reconnects, the reservation is still valid — auto-recover the flow.
- **Seat availability API lag:** Railway backend availability data may be stale (5–10 second lag). Account for this when checking if reserved seat is still available at payment time. If seat shows unavailable but was just reserved, give user 30-second grace period to complete payment: "This seat is being booked right now — completing payment will secure it for you. Rush!"
- **Multiple seat reservations:** Prevent user from holding multiple seats simultaneously. If user tries to reserve a second seat, the first reservation is automatically released.

---

## Feature Spec 4: PNR Status Integration on Main Site

### Problem Statement
PNR status is not part of the main booking surface. From the IRCTC home page, the user is sent to a separate Indian Railways enquiry site (`indianrail.gov.in/enquiry/PNR`) with a different layout, navigation, and interaction model. Booked passengers checking status on the day of travel, especially users who need a quick status lookup before leaving for the station, are forced to leave the booking context. This breaks continuity and creates friction in the post-booking experience.

### Current State (from Part A)
The home page exposes a PNR Status link that opens a legacy Passenger Reservation Enquiry site instead of staying inside IRCTC. The destination page uses a dated interface with only a PNR field and a Submit button. Users have to re-authenticate, navigate a different site, and then return to IRCTC for any other task. The flow is: Home → Click PNR Status → Navigate to external site → Read PNR status → Manually return to IRCTC for rebooking or other actions.

### Proposed Solution
Integrate PNR status lookup into the main IRCTC website. After login, users see a "My Bookings" dashboard showing all booked PNRs with their current status (CNF, WL, RAC, TDR). Users can click a PNR to see full details: train name, route, passenger names, journey date, current status, and a timeline of status changes. A search box allows quick PNR lookup by 10-digit number. The entire experience stays within IRCTC.

### Proposed User Flow — Step by Step
1. User logs into IRCTC
2. Home page now shows a "My Active Bookings" section: list of recent PNRs with status at-a-glance (CNF, WL, RAC, TDR badge)
3. User can click on any PNR to see full details: train name, seats/berths, route, passengers, journey date, current status, checkin time
4. Status section shows timeline: "Booking Confirmed (15 May 2026, 10:15 AM) → WL Position Updated to 8 (14 May, 4 PM) → Confirmed (14 May, 11:30 PM)"
5. If status is WL, show the WL prediction from the AI model (see AI Feature Spec): "78% chance of confirmation based on historical patterns for this route and class"
6. User can also search for an older booking: Home → "Search PNR" box → enter 10-digit PNR → results show matching booking with status and full details
7. From PNR detail page, user can: view cancellation policy, apply for refund if TDR, check alternate train options if cancelled, or book a related train
8. Push notifications keep user updated: "Your WL 15 has been confirmed on 12622 Tamil Nadu Express. Boarding at Chennai Central at 22:30."

### Technical Implementation Plan

**System components affected:**
- Frontend: new "My Bookings" dashboard page, PNR detail page, PNR search component
- Backend: PNR lookup service, status history tracking, push notification service
- Database: PNR status history table, notification preferences
- External integration: Railway enquiry system API (if available for data sync)

**New data requirements:**
- `PNRStatusHistory` table: pnr_number, status, status_change_timestamp, status_notes, wl_position (if WL), coach_number (if CNF)
- `UserBookings` denormalized view: user_id, pnr, train_name, route, journey_date, current_status, last_updated_timestamp
- `PNRNotificationPreferences` table: user_id, notify_on_status_change (Y/N), notify_method (SMS/Email/Push)

**API changes:**
```
GET /user/bookings
  → Returns: [ { pnr, trainName, route, journeyDate, currentStatus, lastUpdated }, ... ]

GET /pnr/:pnrNumber
  → Returns: { pnr, trainName, route, journeyDate, passengers: [{name, age, berth}], 
              currentStatus, wlPosition (if WL), statusHistory: [{status, timestamp, notes}], 
              refundEligibility, alternateTrainOptions }

GET /pnr/search?q=:pnrNumber
  → Returns: [ { pnr, trainName, journeyDate, status } ] (for typeahead)

POST /pnr/:pnrNumber/apply-refund (if TDR)
  → Initiates refund process via Railway backend
```

**Frontend changes:**
- "My Bookings" dashboard page: grid or list of recent PNRs with status badges
- PNR detail page: full booking info, status timeline, action buttons (refund, rebooking, etc.)
- PNR search box with typeahead on home page
- Status badge component: color-coded (green=CNF, orange=WL, red=TDR)
- Push notification component: for status change alerts

**Third-party services (if any):**
- Indian Railways enquiry API (if available for real-time sync) or database replication from Railway system
- Push notification service: Firebase Cloud Messaging or AWS SNS
- SMS gateway: for SMS status updates (existing contract likely already with IRCTC)

### Success Metrics
- PNR status lookup inquiries on external site: reduce by 70% within 30 days (as users prefer on-site lookup)
- User satisfaction score for post-booking experience: 2.5/5 → 4.0/5
- Repeat visitors checking status within 48 hours of booking: increase by 50% (better engagement)
- Support queries about booking status: reduce by 40%
- Mobile user session length on IRCTC (time spent checking PNR + rebooking): increase by 25%

### Edge Cases and Constraints
- **Railway API latency:** The Indian Railways enquiry system has varying latency (5–30 seconds per query). Cache PNR status locally with 5-minute TTL; user can force refresh with "Update Now" button.
- **Status synchronization lag:** If a user's WL is confirmed on Railways but IRCTC status cache hasn't updated yet, show: "Status may not be live. [Refresh Now]"
- **Old bookings:** Bookings older than 1 year may not be accessible (Railway data retention). Show: "Booking record archived. Call 139 for historical booking inquiries."
- **TDR (Train Delayed/Refund):** Handle TDR differently — show refund eligibility timeline and auto-process refunds where possible, manual support for edge cases.
- **Multiple PNRs on same booking:** Some users may have split bookings (multiple PNRs for the same journey). Show grouped view: "2 PNRs for your journey to Delhi (15 May)"
- **Offline mode:** If Railway API is down, show cached status with "Last updated X minutes ago. [Retry]"

---

## Feature Spec 5: Reservation Chart Search UX Improvement

### Problem Statement
The reservation chart page opens with a blank-looking train selector that announces an unhelpful accessibility message before the user types anything. The page expects users to already know the exact train or number and the boarding station, but it does not make that dependency obvious. The field behavior is confusing for older users and first-time users who don't already know the train number. The empty state reads like a failed search instead of a usable starting point.

### Current State (from Part A)
The page shows three required inputs: Train Name/Number, Journey Date, and Boarding Station. The first field behaves like a searchable combobox, but its empty state announces "0 results available. Select is focused, type to refine list, press Down to open the menu" — which is confusing before the user has entered anything. The page does not guide users on whether to type a train name (e.g., "Rajdhani") or number (e.g., "12625"), and it requires knowing the exact boarding station in advance. Most users abandon the page or enter partial data, leading to errors or no results.

### Proposed Solution
Redesign the Reservation Chart entry screen with clear affordances: (1) Show example input in the train field tooltip ("e.g., 12625 or Tamil Nadu Express"); (2) Replace the cryptic empty state with "Start typing a train name or number" prompt; (3) Add a secondary "Popular Trains" section below the search for users who don't know the exact train; (4) Allow boarding station to be optional with a "Show all stations on this train" fallback; (5) Show a loading skeleton and "Fetching chart..." during data fetch; (6) Error state is clear: "No chart data found for this train. It may not run today."

### Proposed User Flow — Step by Step
1. User opens the Reservation Charts page
2. The train selector shows placeholder text: "Type train name or number… e.g., 12625, Rajdhani Express"
3. Below the form, a "Popular Trains Today" section shows 5 frequently searched trains with their numbers and next departure
4. User clicks on "12622" in the popular trains, which auto-fills the train field
5. User selects today's date from the date picker (or leaves default)
6. Boarding Station field appears with placeholder: "Optional — leave blank to see all stations"
7. User leaves Boarding Station blank and clicks "View Chart"
8. A loading state shows: "Fetching reservation chart…" with a skeleton loader
9. The chart page loads showing all stations on the route with availability
10. User can now click on any station to drill into availability for that station

### Technical Implementation Plan

**System components affected:**
- Frontend: Reservation Chart entry page redesign, search autocomplete component, popular trains widget
- Backend: Popular trains service (train frequency service), existing chart API (no changes needed)
- Cache: Popular trains list (updated daily or hourly based on actual searches)

**New data requirements:**
- `PopularTrains` cache table: train_id, train_name, train_number, search_frequency (today/this_week), next_departure_time
- (Derived from analytics on actual chart searches)

**API changes:**
```
GET /charts/popular-trains?date=2026-05-15&limit=10
  → Returns: [ { trainNumber, trainName, nextDepartureTime, dayOfWeek }, ... ]

(Existing GET /charts/train/:trainId endpoint remains unchanged)
```

**Frontend changes:**
- New entry page component: clean form with helpful labels and placeholders
- Train autocomplete: shows train number + name as user types, popular trains section below
- Date picker: pre-selected to today, allow past/future
- Boarding Station field: labeled "Optional" with clear instruction
- Loading state: skeleton chart view while fetching
- Error state: "No chart data found. Try a different train or date."
- Empty state for popular trains: always visible, auto-filled on click

**Third-party services (if any):**
- None (uses existing IRCTC backend)

### Success Metrics
- User abandon rate on Reservation Charts page: 30% → 8% (measured by sessions that don't reach chart view)
- Average time to chart view: 2 minutes → 30 seconds
- First-time user success rate (reach chart without error): 45% → 82%
- Accessibility complaint rate on this page: reduce by 70%
- Mobile user engagement on charts: increase by 40%

### Edge Cases and Constraints
- **Train number ambiguity:** Some trains have multiple compositions or run on different dates. If the user searches "12625" and there are 3 variants, show: "Found 3 trains with this number. Which one? [12625A - 08:30 today] [12625 - 18:00 today] [12625B - runs only weekends]"
- **No chart data:** If a train runs but chart data hasn't been published yet (rare), show: "Chart is being prepared. Check back in 10 minutes."
- **Historical charts:** Allow users to search for charts from past journeys (read-only, for reference). Show archive indicator: "📋 This is an archived chart from 5 April 2026."
- **Slow autocomplete:** If train search takes >3 seconds, show: "Searching…" + suggest trying fewer characters or the train number instead of name.

---

## Feature Spec 6: AskDisha Support Layer Integration Redesign

### Problem Statement
The main booking page loads an AskDisha support layer and related embedded content alongside the search form. The assistant is presented as a floating element competing with the search form for attention, and the page produces cross-origin frame access errors in the browser console. This makes the support experience feel bolted on rather than integrated, clutters the primary booking flow, and introduces technical debt. On mobile, the overlap becomes more pronounced and hurts the primary user intent.

### Current State (from Part A)
The booking home page shows the search form with a floating AskDisha support prompt and embedded assistant/complementary area simultaneously. The browser console logs frame-access and permissions-policy errors during page load. Users attempting a booking have to process both the booking form and the support layer at once, which creates cognitive friction. On smaller viewports, the overlap intensifies and diverts attention from the primary booking task.

### Proposed Solution
Redesign the support layer to be contextual and off to the side on desktop, and hidden-by-default on mobile. Replace the floating prompt with a "Help?" button that opens a side panel when clicked. Move embedded assistant content into a collapsible footer or a dedicated help tab. On mobile, hide all support elements by default; provide a "Help & FAQ" tap target in the navigation bar. Fix all cross-origin frame errors. Ensure the primary booking form takes up 100% of the mobile viewport.

### Proposed User Flow — Step by Step

**Desktop (1024px+):**
1. User opens IRCTC home page
2. The main booking search form is centered and prominent
3. To the right of the form, a subtle "Help?" button is visible (not a floating prompt)
4. User can proceed directly to booking without any visual distraction from support content
5. If user clicks "Help?", a side panel slides in from the right showing FAQ, recent issues, and AskDisha assistant
6. User can minimize or close the help panel and continue booking
7. Help panel persists as user navigates within the search results

**Mobile (< 768px):**
1. User opens IRCTC home page
2. The booking search form fills the entire viewport — no support elements visible
3. Navigation bar has a "?" icon in the top-right, which opens a help modal on tap
4. User can close the help modal and proceed with booking immediately
5. Help modal contains FAQ, status links, and AskDisha assistant in a single scrollable view
6. All frame errors are resolved — no console warnings

### Technical Implementation Plan

**System components affected:**
- Frontend: home page layout, help panel component, mobile navigation
- AskDisha integration: refactor embedded frame to load asynchronously (not on critical path)
- CSS/Layout: ensure no layout shift, responsive design for both desktop and mobile

**New data requirements:**
- None (reuses existing FAQ and assistant content)

**API changes:**
- None (AskDisha API endpoint remains the same, just deferred loading)

**Frontend changes:**
- Redesign home page grid: 60% booking form, 40% reserved for help panel (desktop)
- Help panel component: collapsible, can be hidden and re-opened via "Help?" button
- Mobile layout: 100% booking form by default, help in a dismissible modal
- Navigation bar: add "?" icon visible only on mobile
- Defer AskDisha frame loading: use IntersectionObserver so frame only loads if user clicks Help
- Remove floating prompt element
- Resolve frame access errors: adjust Content-Security-Policy headers, ensure CORS configuration

**Third-party services (if any):**
- AskDisha API (no changes, just refactored integration approach)

### Success Metrics
- Mobile booking form conversion rate: 28% → 42% (removing visual distraction)
- Desktop booking form conversion rate: 31% → 33% (help available but not intrusive)
- AskDisha assistant usage: maintain or increase (users who actually want help can find it easily)
- Page load time: reduce by 800ms (deferred frame loading)
- Browser console errors on home page: 0
- Support ticket volume: maintain or reduce (help panel should prevent some support needs)

### Edge Cases and Constraints
- **Help panel persistent state:** Remember if user opened help panel last session. If they did, show it again (but allow hiding). If they never opened it, don't show by default.
- **FAQ freshness:** Ensure FAQ content is kept up-to-date with current known issues. Stale FAQ can erode trust.
- **AskDisha availability:** If AskDisha service is down, show a graceful fallback: "Support assistant temporarily unavailable. [View FAQ] [Contact Support]"
- **Network latency:** If help panel takes >3 seconds to load due to frame latency, show skeleton and "Loading…"
- **Mobile tap targets:** Ensure "?" icon is at least 48px × 48px for comfortable mobile tapping.
- **A/B test consideration:** Consider A/B testing the Help panel position (right side vs. left side on desktop) to find optimal placement.

---

## Summary Table: 6 Features at a Glance

| Feature | Problem | Impact | Complexity |
|---------|---------|--------|-----------|
| 1. Tatkal Virtual Queue | Crashes at 10:00 AM, 60% failure rate | Critical: 40L+ users daily | Very High: Redis, WebSocket, queue logic |
| 2. Search Filter Persistence | Filters reset after refresh, state lost | High: All search users | Low: URL params + session storage |
| 3. Seat Selection State | Seat resets between steps, wrong berth booked | High: 30% of bookings have berth preference | Medium: Backend session state |
| 4. PNR Status on Main Site | Redirects to external legacy site, breaks flow | High: All post-booking users | High: New dashboard, API integration |
| 5. Reservation Chart UX | Empty state confusing, high abandon rate | Medium: Niche use case (30% of users) | Low: UI/UX redesign + hints |
| 6. AskDisha Integration | Clutters home page, frame errors, mobile friction | Medium: Mobile experience bloat | Low: Layout redesign + deferred loading |
