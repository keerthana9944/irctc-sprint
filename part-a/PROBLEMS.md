# IRCTC Problem Discovery — Part A

## Summary
- Total problems documented: 6 (3 given + 3 self-discovered)
- Platform explored: irctc.co.in and linked IRCTC / Indian Railways enquiry pages, live as of 07 May 2026
- Devices used: Desktop Chrome on Windows
 - Repository: https://github.com/keerthana9944/irctc-sprint

---

## Problem 1: Tatkal Booking Crashes at 10:00 AM [Given]

**Category:** Performance / Reliability

**What is broken:**
The Tatkal booking flow becomes unresponsive or fails right when quota opens at 10:00 AM. Users can reach the booking screen, enter passenger details, and still lose the booking at the final step because the system either freezes, throws a gateway error, or times out without explaining what happened.

**Affected users:**
Anyone trying to book Tatkal tickets, especially daily commuters, emergency travelers, and users in Tier 2 and Tier 3 cities who rely on IRCTC for time-sensitive travel.

**Frequency:**
Daily, concentrated in the 9:58 AM to 10:05 AM window. The failure is a recurring peak-hour condition rather than an isolated incident.

**Current flow - step by step:**
1. User opens IRCTC around 9:50 AM and logs in.
2. User searches for a train and selects the Tatkal quota.
3. Availability still shows seats just before opening time.
4. User fills passenger details and clicks Book Now at 9:59:45.
5. At 10:00:00 the page freezes and shows a spinner with no queue state.
6. After a delay, the user sees HTTP 502, a timeout, or a CAPTCHA/session reset.
7. User refreshes and often finds the quota gone or the session logged out.
8. User has no reliable feedback about whether payment was attempted.

**Where exactly it breaks:**
Step 5: the system gives no progress or queue feedback while the request is waiting, so users repeat actions and amplify the load.

**Impact:**
Users miss the only viable booking window, waste time re-logging in, and often panic-check their bank statement or UPI app for an unknown payment state.

---

## Problem 2: Search Filters Do Not Work Reliably [Given]

**Category:** Information Architecture / UX

**What is broken:**
The train results filter panel is inconsistent. Filters such as class, quota, and availability can reset or show results that do not match the selected state, especially after a refresh or results reload.

**Affected users:**
All users searching for trains, with extra pain for senior citizens, first-time users, and travelers who depend on filters to narrow down accessible or time-specific options.

**Frequency:**
Intermittent, with failures becoming more common during high-traffic periods.

**Current flow - step by step:**
1. User enters source, destination, and date.
2. Search results show a long list of trains.
3. User applies filters such as Sleeper Class and Available.
4. The page reloads and some waitlisted trains still remain visible.
5. User opens a train and sees a class state that contradicts the filter.
6. User goes back and the filter state has reset.
7. User abandons the filter panel and scans trains manually.

**Where exactly it breaks:**
Step 4 to step 6: the selected filter state is not preserved reliably when availability updates or when the user returns to the result list.

**Impact:**
The search takes much longer than it should, and users lose trust in the availability labels shown by the site.

---

## Problem 3: Seat Selection Resets Randomly [Given]

**Category:** UX / State Management

**What is broken:**
The selected berth or seat sometimes disappears between the seat map and the passenger details step. A user can choose a lower berth, continue forward, and still end up with Auto assignment or a different berth in the next screen.

**Affected users:**
Families, elderly passengers, travelers with disabilities, and anyone with a strong berth preference. This is especially costly for people who need a lower berth or adjacent seats.

**Frequency:**
Intermittent, with a higher failure rate on mobile flows than on desktop.

**Current flow - step by step:**
1. User selects a train, class, and quota.
2. Seat map loads and shows berth availability.
3. User selects a specific seat or lower berth.
4. User clicks Proceed.
5. Passenger details page shows Auto or a different berth.
6. User goes back to reselect a seat.
7. The previous seat may now appear unavailable.
8. User continues with a less suitable berth or abandons the flow.

**Where exactly it breaks:**
Step 4 to step 5: the seat state is not carried consistently from the seat map to the next booking step.

**Impact:**
Users lose control over berth preference and may discover the mismatch only after they have already progressed deep into the booking flow.

---

## Problem 4: PNR Status Lives on a Separate Legacy Site

**Category:** Information Architecture / UX

**What is broken:**
PNR status is not part of the main booking surface. From the IRCTC home page, the user is sent to a separate Indian Railways enquiry site with a different layout, different navigation, and a different interaction model. The user has to leave the booking context just to check basic trip status.

**Affected users:**
Booked passengers checking status on the day of travel, especially users who need a quick status lookup before leaving for the station.

**Frequency:**
Always, whenever a user checks PNR status after booking.

**How I found it:**
I opened the IRCTC home page, clicked the PNR Status entry, and landed on a legacy Passenger Reservation Enquiry site instead of staying inside the main booking experience.

**Screenshot or description:**
The home page exposes a PNR Status link that opens `indianrail.gov.in/enquiry/PNR/PnrEnquiry.html`, while the destination page uses a dated, separate enquiry layout with only a PNR field and a Submit button.

**Current flow - step by step:**
1. User opens the IRCTC home page.
2. User clicks PNR Status.
3. A new site opens on a different domain and interface.
4. User reads a separate help string about where to find the PNR.
5. User enters the 10-digit PNR number.
6. User submits the form to get current status.
7. User still has to return to the main site for booking actions or other trip tasks.

**Where exactly it breaks:**
Step 2 to step 3: the user is routed into a different product surface for a basic post-booking task, which breaks continuity and makes the journey feel fragmented.

**Impact:**
The status-check flow feels detached from the booking flow, and users have to bounce between sites to answer simple trip questions.

---

## Problem 5: Reservation Chart Search Is Cryptic and Hard to Start

**Category:** Information Architecture / Accessibility

**What is broken:**
The reservation chart page opens with a blank-looking train selector that announces an unhelpful accessibility message before the user types anything. The page expects users to already know the exact train or number and the boarding station, but it does not make that dependency obvious in the flow.

**Affected users:**
Passengers checking charts close to departure, especially older users and first-time users who do not already know the train number.

**Frequency:**
Always, for anyone who uses the Reservation Chart page.

**How I found it:**
I opened the live Reservation Charts page from IRCTC and focused the train selector. The field exposed the message “0 results available. Select is focused, type to refine list, press Down to open the menu,” which is confusing before the user has entered anything.

**Screenshot or description:**
The page shows only three required inputs: Train Name/Number, Journey Date, and Boarding Station. The first field behaves like a searchable combobox, but its empty state reads like an error instead of a prompt.

**Current flow - step by step:**
1. User opens the Reservation Charts page.
2. User sees three required fields with no example train entry.
3. User clicks the train selector.
4. The empty state says there are zero results before the user has searched.
5. User must guess whether to type a train name or number.
6. User also has to know the boarding station in advance.
7. User submits the form to get the chart.

**Where exactly it breaks:**
Step 3 to step 4: the field’s empty state is confusing and looks like a failed search instead of a usable starting point.

**Impact:**
Users do not get a clear entry point for chart lookup and may abandon the page or enter partial, incorrect information.

---

## Problem 6: AskDisha Support Layer Competes With the Booking Flow

**Category:** UX / Mobile

**What is broken:**
The main booking page loads an AskDisha support layer and related embedded content alongside the search form. On the live page, the assistant is presented as a floating element, and the page also produces cross-origin frame errors in the browser console. That makes the support experience feel bolted on rather than integrated.

**Affected users:**
First-time users, mobile users, and anyone who relies on the support assistant while trying to start a booking.

**Frequency:**
Always on the home / booking page, because the assistant layer is part of the default page experience.

**How I found it:**
I opened the live train-search page and observed a floating AskDisha assistant plus embedded content in the booking surface while the browser logged frame access errors.

**Screenshot or description:**
The booking home page shows the search form, a floating AskDisha prompt, and an embedded assistant/complementary area at the same time. The browser console also logged a cross-origin frame access error while the page loaded.

**Current flow - step by step:**
1. User opens the IRCTC home page.
2. The booking form loads with a floating AskDisha support prompt.
3. Additional embedded assistant content appears near the booking area.
4. The browser console logs frame-access and permissions-policy errors during load.
5. User begins entering the from and to stations.
6. The page asks the user to process both the booking form and the support layer at once.
7. On a smaller viewport, the overlap becomes harder to ignore.

**Where exactly it breaks:**
Step 2 to step 4: support content is introduced before the booking task is complete, adding visual noise and technical errors to the primary flow.

**Impact:**
The page feels cluttered and less trustworthy, and mobile users are more likely to lose focus while starting a booking.
