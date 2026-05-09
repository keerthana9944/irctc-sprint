# IRCTC Part B — Impact vs Effort Matrix

## The Matrix

|                        | **Low Effort** | **High Effort** |
|------------------------|----------------|-----------------|
| **High Impact**        | Search Filters | Tatkal Queue    |
|                        | Reservation Chart UX | PNR Integration |
|                        |                | Seat Persistence |
| **Low Impact**         | AskDisha Integration | *(None)* |

---

## How I Scored Each Dimension

### Impact Scoring (1–5 scale, where 3 = medium)

**Drivers of Impact Score:**
1. **Users affected** (from Part A frequency analysis): 5pts if 40L+, 4pts if 5L–40L, 3pts if 1L–5L, 2pts if <1L
2. **Core booking flow** (is this on the critical path to booking?): +2pts if YES, 0pts if NO
3. **Severity of consequence** (what happens if user hits this problem?): 2pts if trip missed, 1pt if frustration only
4. **Frequency** (how often does user encounter this?): 2pts if every session, 1pt if intermittent

**Impact Scoring Table:**

| Problem | Users Affected | Core Flow | Severity | Frequency | **Total** | **Tier** |
|---------|---|---|---|---|---|---|
| 1. Tatkal Queue | 5 | +2 | 2 | 2 | **11/11** | CRITICAL |
| 2. Search Filters | 4 | +2 | 1 | 2 | **9/11** | HIGH |
| 3. Seat Persistence | 3 | +2 | 2 | 1 | **8/11** | HIGH |
| 4. PNR Integration | 4 | +0 | 1 | 2 | **7/11** | HIGH |
| 5. Reservation Chart UX | 2 | +0 | 1 | 2 | **5/11** | MEDIUM |
| 6. AskDisha Integration | 4 | +0 | 0 | 2 | **6/11** | MEDIUM |

**Reasoning:**
- Tatkal (11/11): 40L+ users, core booking, critical consequence (miss booking window), happens daily
- Search (9/11): All users, core flow, frustration + friction, frequent
- Seat (8/11): Affects 30% of bookings + families/elderly (high stakes), core flow, medium frequency
- PNR (7/11): All post-booking users, not core booking, medium urgency, frequently checked
- Chart (5/11): 30% of users, not core flow (niche), medium frustration, frequent for those who use it
- AskDisha (6/11): Desktop + mobile users, not core flow, minor impact, persistent on every session

---

### Effort Scoring (1–5 scale, where 3 = medium)

**Drivers of Effort Score:**
1. **System components touched** (frontend, backend, DB, payment, etc.): 1pt per component
2. **New infrastructure** (Redis, queues, WebSocket, ML pipeline, etc.): +2pts if YES
3. **Risk of breaking existing flows**: +1pt if HIGH risk, 0pts if LOW risk
4. **Railway API dependencies** (IRCTC cannot fully control): +1pt per dependency

**Effort Scoring Table:**

| Problem | Components | Infrastructure | Risk | Railway Deps | **Total** | **Tier** |
|---------|---|---|---|---|---|---|
| 1. Tatkal Queue | 3 (frontend + backend + session) | +2 (Redis + WebSocket) | +1 (high risk, peak hours) | +1 (seat API) | **7/8** | HIGH |
| 2. Search Filters | 1 (frontend only) | 0 | 0 | 0 | **1/8** | LOW |
| 3. Seat Persistence | 3 (frontend + backend + session) | 0 | +1 (seat race condition) | 0 | **4/8** | MEDIUM |
| 4. PNR Integration | 4 (frontend + backend + DB + notify) | 0 | 0 | +1 (sync with Railway) | **5/8** | MEDIUM-HIGH |
| 5. Reservation Chart UX | 1 (frontend only) | 0 | 0 | 0 | **1/8** | LOW |
| 6. AskDisha Integration | 1 (frontend only) | 0 | 0 | 0 | **1/8** | LOW |

**Reasoning:**
- Tatkal (7/8): Frontend + backend + session state + Redis + WebSocket = high complexity; peak-hour risk; railway seat API dependency
- Search (1/8): URL params + session storage, frontend only, no new infra, no railway dependency
- Seat (4/8): Frontend + backend session + payment integration, but no new infrastructure; race conditions require careful handling
- PNR (5/8): New dashboard page + backend PNR service + notifications + data sync with Railway API
- Chart (1/8): UX redesign only, existing backend endpoints, no infrastructure
- AskDisha (1/8): Layout refactoring + deferred loading, pure frontend

---

## Placement Justifications

### Problem 1: Tatkal Virtual Queue System — **Major Project** (High Impact, High Effort)

**Why Impact is High (11/11):**
This problem affects 40–50 lakh users daily and directly blocks 60–70% of Tatkal bookings during peak hours. The core booking flow breaks at the exact moment quota opens, preventing users from even reaching the seat selection step. The consequence is catastrophic: users miss their only viable booking window and incur real financial loss. The frequency is 100% during 10:00 AM peak windows on every single day. No other problem in the set affects the platform at this scale.

**Why Effort is High (7/8):**
The solution requires Redis infrastructure, a WebSocket/real-time push service, queue polling and dequeue logic, session token management, mobile fallback strategies (polling vs. WebSocket), and frontend state management. The implementation touches 5+ system components and introduces elevated risk during peak hours — a failure during Tatkal opening hour could cause another platform crash. The Railway backend seat API dependency adds complexity: we must handle the case where the queue system succeeds but the seat API fails partway through.

**Sprint Implication:**
This is the highest-priority feature. Do this first. It is a 6–8 week dedicated sprint for a specialized backend + frontend team. Do not underestimate the complexity: testing a queue system at 20–40 lakh concurrent users requires load testing infrastructure that may not yet exist.

**How to verify it's done:**
- Load test: 20L concurrent users entering queue, <5% error rate
- Peak hour (10 AM) end-to-end test: queue position shown live, 90-second booking window works, fallback to polling on 2G works
- No 502 errors on peak day; all errors are graceful degradation (queue halted, not crashed)

---

### Problem 2: Search Filter Persistence & Reliability — **Quick Win** (High Impact, Low Effort)

**Why Impact is High (9/11):**
Every single user searches for trains at least once per booking. The filters panel is broken or unreliable for all of them. Filter state resets after refresh, availability updates, or back button navigation, forcing users to re-apply filters or manually scan results. This creates friction on 100% of search sessions. The consequence is longer search times and reduced trust in the availability labels shown by IRCTC. It's not as critical as Tatkal (no trip missed), but it's the highest-friction issue for the most users.

**Why Effort is Low (1/8):**
Persisting filter state requires only frontend work: (a) update URL query parameters when filters change, (b) read URL params and session storage on page load, (c) re-apply filters to results. This is a 2-3 day sprint for a frontend engineer. No backend changes needed — the search API already supports filter parameters. No new infrastructure. Extremely low risk of breaking existing flows because it's purely additive (existing filter logic + persistence).

**Sprint Implication:**
Do this second, immediately after Tatkal queue. It's a morale win for the team: ship it in a single sprint, measure user satisfaction spike, learn momentum for the next items.

**How to verify it's done:**
- Refresh page while filters are applied → filters re-apply automatically
- Navigate back to results after clicking a train → filters still applied
- Share filtered URL with a friend → they see the same filtered results
- Mobile + desktop both work
- No regression in search result loading time

---

### Problem 3: Seat Selection State Persistence — **Major Project** (High Impact, Medium-High Effort)

**Effort Score Note (Post-Peer-Review):** Originally scored as 4/8 (Medium), but race condition complexity and Railway API conflict resolution elevate this to **High Effort** for this sprint. Recommend pairing a senior backend engineer (2+ years distributed systems experience) with a mid-level frontend engineer to handle seat conflict resolution, race conditions, and payment validation edge cases.

**Why Impact is High (8/11):**
Users with strong berth preferences (families, elderly, people with disabilities) book tickets specifically to choose a lower berth or adjacent seats. When the seat resets between steps, users are forced to accept worse berths and lose utility of their booking. 30% of bookings have an explicit berth preference, making this high-volume. The consequence is real: elderly passengers lose control of their journey comfort, families are separated across berths. Frequency is intermittent but high-consequence (not every session, but when it hits, user is highly frustrated).

**Why Effort is Medium-High (4/8):**
The solution requires a backend change: store reserved seat ID in the booking session and carry it forward through to payment. This requires (a) new POST /booking/reserve-seat endpoint, (b) session table additions, (c) SeatReservation temporary table for deduplication, (d) frontend changes to call reserve-seat on click, (e) passenger details page to fetch and display reserved seat. Moderate risk: if a seat is released while user is on passenger details page, we must handle the race gracefully. This is a 3-4 week sprint for backend + frontend.

**Sprint Implication:**
Schedule this after Search Filters. Seat persistence is high-impact for users who care (families, elderly) and medium complexity. It's a solid follow-up to the quick wins.

**How to verify it's done:**
- Select a seat, wait 2 minutes, confirm seat is still reserved (show countdown)
- Go back to seat map after passenger details → selected seat is still highlighted
- If seat is taken while in passenger details, show alternatives and let user choose
- Seat reservation expires gracefully after 10 minutes (show warning and offer extend)

---

### Problem 4: PNR Status Integration on Main Site — **Major Project** (High Impact, Medium-High Effort)

**Why Impact is High (7/11):**
Every user who has booked a ticket needs to check status at least once (often multiple times close to departure). Currently they are routed to a legacy external site, breaking continuity of the IRCTC experience and creating friction. The consequence is a fragmented, untrusted user experience. Frequency is high: any user checking status post-booking. The impact is lower than Tatkal (booking still succeeds, just with poor follow-up) but still high because it affects all 10+ crore IRCTC users who book.

**Why Effort is Medium-High (5/8):**
The solution requires a new "My Bookings" dashboard page, a PNR detail page, integration with Railway status sync (or caching), push notifications, and possibly refund processing APIs. This touches 4 system components. Risk is moderate: Railway API latency and stale data must be handled gracefully. This is a 4-5 week sprint for backend + frontend + notifications.

**Sprint Implication:**
Schedule this after Seat Persistence. High-impact feature that improves post-booking experience, not the core booking flow. Good use of parallel engineering teams if you have them.

**How to verify it's done:**
- After booking, PNR is visible in "My Bookings" dashboard within 5 seconds
- PNR detail page shows all status history with timestamps
- Pushing a page refresh shows updated status from Railway system within 5 seconds (or cached if Railway API is slow)
- AI prediction for WL tickets is shown on detail page
- Push notification arrives 48h before departure with status update

---

### Problem 5: Reservation Chart UX Improvement — **Quick Win** (Medium Impact, Low Effort)

**Why Impact is Medium (5/11):**
The Reservation Chart page has a confusing empty state that makes it hard for first-time users and older users to start searching. It's not a core booking problem — only 30% of users use this page (most just book directly without checking the chart). But for the 30% who do use it, the friction is real: high abandon rate, user confusion about whether to enter train name or number. The consequence is moderate: users abandon the page or call support, but they can still book without the chart.

**Why Effort is Low (1/8):**
The fix is pure UX/UI: replace the cryptic empty state with helpful prompts, show popular trains, make boarding station optional, add loading states. No backend changes. No new infrastructure. This is a 1-2 week design + frontend sprint.

**Sprint Implication:**
Do this as a parallel track or fill-in work if you have extra front-end capacity. It's a morale booster and accessibility improvement.

**How to verify it's done:**
- Empty state no longer says "0 results available"; instead says "Start typing a train name or number"
- Popular trains section shows 5 frequently searched trains with next departure time
- Boarding station is labeled "Optional — leave blank to see all stations"
- First-time user success rate increases from 45% to 82%
- Accessibility tool (WAVE, Axe) shows 0 errors on the page

---

### Problem 6: AskDisha Support Layer Integration Redesign — **Fill-In** (Medium Impact, Low Effort)

**Why Impact is Medium (6/11):**
The AskDisha support layer clutters the home page, produces frame errors, and distracts users on mobile. The consequence is minor: it adds visual noise but doesn't block booking (users can still search and book). The problem is primarily a mobile UX friction point and technical debt (console errors). Impact is real but not trip-critical like Tatkal or friction-critical like Search Filters.

**Why Effort is Low (1/8):**
The fix is layout refactoring + CSS + deferred frame loading. Move the floating prompt to a collapsible help panel on desktop; hide support completely on mobile with a [?] icon in the nav. Load the AskDisha iframe only when user clicks Help. No backend changes. No new infrastructure. This is a 1 week frontend sprint.

**Sprint Implication:**
Schedule this as work-in-progress or fill-in work for mobile specialists. Fix console errors early (they hurt platform trust), but don't block other features waiting for this.

**How to verify it's done:**
- Mobile: Search form fills entire viewport; [?] icon in nav opens help modal on tap
- Desktop: Help panel can be hidden and re-opened; does not compete with search form
- No frame errors in browser console on home page
- Page load time improves by 800ms (from deferred frame loading)
- Mobile booking form conversion rate increases (users less distracted)

---

## Recommended Sprint Order

### Sprint 1 (Weeks 1–8): Tatkal Virtual Queue System
- **Why first:** Highest impact, only 60–70% of Tatkal bookings succeed; blocking the platform at peak hours
- **Team:** Backend (Redis, queue service, WebSocket) + Frontend (queue screen, real-time counter) + QA (load testing)
- **Success criteria:** <5% error rate at 20L concurrent users; live position updates working; 90-second booking window functional
- **Parallel work:** Start Search Filter persistence design while Tatkal backend is being built

---

### Sprint 2 (Weeks 2–3): Search Filter Persistence & Reliability
- **Why second:** Quick win; high impact for all users; removes friction from every search session
- **Team:** Frontend only (can run in parallel with Tatkal backend sprint)
- **Success criteria:** Filters persist across refresh, back button, URL sharing; search time reduced from 90s to 45s
- **Parallel work:** Start Seat Persistence design

---

### Sprint 3 (Weeks 4–7): Seat Selection State Persistence
- **Why third:** High impact for preference-driven users (families, elderly); medium effort
- **Team:** Backend (session state) + Frontend (seat reservation UI) + QA
- **Success criteria:** Seat persists from seat map to passenger details; grace period for race conditions; countdown timer shows reservation expiry
- **Parallel work:** Finish Search Filters; start PNR Integration design

---

### Sprint 4 (Weeks 5–9): PNR Status Integration on Main Site
- **Why fourth:** High impact; improves post-booking experience; not blocking core flow
- **Team:** Backend (dashboard query, status sync) + Frontend (dashboard + detail page) + Notifications + QA
- **Success criteria:** PNR appears in dashboard within 5s of booking; detail page shows status timeline; AI prediction displayed for WL; push notification sent 48h before
- **Parallel work:** Reserve Reservation Chart UX for Mobile specialists

---

### Sprint 5 (Weeks 8–9): Reservation Chart UX Improvement & AskDisha Redesign
- **Why last:** Medium impact; low effort; can run as parallel mobile specialist track
- **Team:** Frontend (chart page + help panel redesign)
- **Success criteria:** Chart abandon rate drops from 30% to 8%; mobile users' help panel opens without blocking search; console errors = 0
- **Parallel work:** None; finalize docs and start peer review feedback incorporation

---

## Effort Estimate Summary

| Feature | Weeks | Team Size | Dependency |
|---------|-------|-----------|-----------|
| 1. Tatkal Queue | 6–8 | 5 (Backend + Frontend + QA) | Infrastructure setup |
| 2. Search Filters | 2–3 | 2 (Frontend) | *None* |
| 3. Seat Persistence | 3–4 | 3 (Backend + Frontend) | Tatkal not required |
| 4. PNR Integration | 4–5 | 4 (Backend + Frontend + Notify) | Railway API access |
| 5. Chart UX | 1–2 | 1 (Frontend) | *None* |
| 6. AskDisha | 1–2 | 1 (Frontend) | *None* |
| **Total** | **18–24 weeks** | **Parallel teams** | **16 weeks critical path** |

---

## Why This Order Maximizes Impact

1. **Tatkal first:** Only feature that can prevent platform crash and blocks 60%+ of bookings; blocks everything else until stable
2. **Search Filters second:** Quick win; boosts team morale; customers see progress; highest user-facing impact per effort
3. **Seat Persistence third:** High-impact for engaged user segment; medium effort; enables families to book confidently
4. **PNR fourth:** Post-booking feature; improves engagement + AI prediction showcase; Railway integration adds complexity but lower urgency
5. **Chart UX + AskDisha last:** Lower impact; pure UX; no blocking dependencies; good parallel work while others finish

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Tatkal queue system fails on launch day | Platform crash, 40L+ users affected | Run 2-month load testing; canary deployment at 5% users first |
| Search filters URL gets too long | Browser compatibility issue | Implement URL shortener; use session ID as fallback |
| Seat race condition not handled | User loses reservation at payment | Implement 30-second grace period; show "completing payment..." |
| Railway PNR API latency high (30s) | Dashboard slow to load | Cache PNR data; show "updating..." spinner; allow force refresh |
| AskDisha frame never loads | Support unavailable | Fallback to direct support link; no frame errors expected |

---

## Success Criteria for Full Part B

- All 6 feature specs written with problem + solution + technical plan + metrics + edge cases
- Wireframes created for all 6 UI-related problems (mobile + desktop, before/after)
- AI feature spec complete with model choice (XGBoost), data sourcing, output design, fallback
- 2×2 matrix populated with all 6 problems placed in correct quadrants
- 3-sentence justification written for each quadrant placement
- Peer review session completed; specs updated based on feedback
- All deliverables traced back to Part A problems with specific references
- Pull request created with complete Part B documentation + inline spec + wireframe + matrix + peer review notes
