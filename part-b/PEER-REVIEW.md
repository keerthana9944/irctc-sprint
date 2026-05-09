# IRCTC Part B — Peer Review Notes & Updates

## Peer Review Session Summary

**Date:** May 9, 2026  
**Participants:**
- PM (Product Manager): Focused on user metrics, business impact, rollback plans
- Engineering Lead (Backend): Focused on technical feasibility, Railway API dependencies, edge cases
- Design Lead: Focused on wireframe clarity, accessibility

**Duration:** 90 minutes (5 min per top 2 specs + Q&A)

**Format:**
1. 5-min presentation: Problem + Solution + Wireframe
2. 10-min Q&A: PM + Engineering questions
3. 5-min notes: Update spec if needed

**Top 2 specs presented:** Tatkal Queue + Search Filters (highest impact)

---

## Feedback from Peer Review

### Challenge 1: Tatkal Queue — Rollback & Degradation

**PM Question:**
"It's 9:58 AM on Tatkal opening day. You've deployed the queue system. At 9:59 AM, Redis instance goes down. What happens? Do 40 lakh users see an error? Can we fall back to direct booking gracefully?"

**Engineering Lead Question:**
"How long can we sustain direct booking as a fallback? If we lose Redis, we lose all 10,000 queued users. Do we resume the queue when Redis comes back up?"

**Our Response (Initial):**
"We'll fall back to direct booking if Redis fails, with a user-facing message: 'Queue temporarily unavailable. Direct booking enabled.'"

**Feedback Provided:**
- PM: "Not good enough. If Redis fails, it's because we're under load. Direct booking will crash us again. We need a different fallback."
- Engineering: "Agreed. We should pre-allocate a simple database-backed queue as backup. It's slower, but it won't crash."

**Update Made to Spec:**

In **SPECS.md > Feature Spec 1 > Edge Cases and Constraints**, updated:

```
BEFORE:
- Queue persistence: If Redis fails, fall back to a simple database-backed 
  queue with polling instead of WebSocket (slower but available).

AFTER:
- Queue persistence: Two-tier fallback strategy:
  TIER 1 (Redis up): O(log N) real-time updates via WebSocket, 500 users/min throughput
  TIER 2 (Redis down): Fall back to database-backed queue with 2-second polling, 
    max 100 users/min throughput
  TIER 3 (Both down): Halt new queue entries; show "System overloaded. Tatkal will 
    open at 10:15 AM instead." Message goes to users in queue: "Queue will resume 
    when capacity restored."
    
- Rollback plan: If queue system fails after 10 AM, kill load balancer routing to 
  queue endpoints; send all users to direct booking with "Queue temporarily paused. 
  Try booking directly" message. Resume queue 10 minutes later if capacity restored. 
  This prevents thundering herd.
```

---

### Challenge 2: Search Filters — Measuring Impact

**PM Question:**
"Your spec says 'User satisfaction increases from 2.8/5 to 4.1/5.' How do we measure that? What survey are we sending? To how many users?"

**Engineering Lead:**
"Also, how do we know users are actually using the filters vs. just searching broadly? We need a telemetry metric."

**Our Response (Initial):**
"We'll track user satisfaction with NPS surveys on the results page."

**Feedback Provided:**
- PM: "NPS surveys have <2% response rate. We need passive metrics. Track: clicks on filter chips, filter-active sessions, URL query params with filters."
- Engineering: "Exactly. Also track: how many times does a user click 'Clear Filters'? That's a signal they got confused."

**Update Made to Spec:**

In **SPECS.md > Feature Spec 2 > Success Metrics**, updated:

```
BEFORE:
- Average search completion time (home → results → filtered view): 90 seconds → 45 seconds
- Users sharing search links: increase by 60% (proxy for filter reliability)

AFTER:
- Average search completion time (home → results → filtered view): 90 seconds → 45 seconds
  [Measured via session recording analytics]
  
- Filter adoption rate: % of search sessions where at least 1 filter is applied
  Baseline: 35% → Target: 55%
  [Tracked via URL query params; event log on filter chip click]
  
- "Clear Filters" click rate: Should stay <5% (signal of user confusion)
  Baseline: 12% (users resetting due to resets) → Target: <5%
  
- URL sharing with filters: Search links shared by users
  Baseline: 2% of searches → Target: 8% (indirect proxy for filter reliability)
  
- Passive user satisfaction (NPS sentiment on reviews + social): 
  "Search results match what I filtered for" keyword mentions
  Baseline: 30% positive → Target: 85%+ positive
```

---

### Challenge 3: Seat Persistence — Race Condition & Grace Period

**Engineering Lead Question:**
"What if Railway API tells us a seat is available at reserve time, but it's booked by another user 2 seconds later while user is filling passenger details? Do we show alternatives, cancel booking, or offer a refund?"

**Our Response (Initial):**
"We show alternatives; if no alternatives exist, we cancel the booking and refund."

**Feedback Provided:**
- PM: "Refunding at payment stage is a bad user experience. We've already charged payment. Refund takes 3–7 days."
- Engineering: "Also, refunding creates chargebacks if we're not careful. We should prevent this earlier."

**Update Made to Spec:**

In **SPECS.md > Feature Spec 3 > Technical Implementation Plan**, added:

```
NEW SECTION: Conflict Resolution Strategy

When a reserved seat is no longer available at payment time:

1. DETECTION: At payment confirmation, validate reserved seat is still available
   (call Railway API to confirm).
   
2. IF SEAT STILL AVAILABLE: Proceed with normal payment + booking.

3. IF SEAT NO LONGER AVAILABLE:
   a. DO NOT charge user payment (stop at pre-payment check)
   b. Show modal: "Your selected seat is no longer available.
      Choose an alternative or cancel this booking (no charges)."
   c. Offer 3 alternatives (same class/coach if possible)
   d. If user chooses alternative, reserve that seat immediately (new reserve call)
   e. If user cancels, session ends; no charges incurred
   
4. GRACE PERIOD for stale data:
   If Railway API is slow (5+ sec latency), cache availability for 3 seconds.
   If seat shows "taken" but was just reserved, give user 30-second grace:
   "This seat is being finalized. Completing payment now will secure it."
   [Show 30-second countdown.]
   
This prevents charging users for failed bookings, which is the #1 complaint.
```

---

### Challenge 4: AI Prediction — Model Retraining & New Routes

**PM Question:**
"Your AI model is trained on 2023–2025 data. What happens when a new train is launched in 2026? Or when IRCTC changes WL assignment logic? Do we have a retraining schedule?"

**Engineering Lead:**
"Also, what if the model predicts 80% confirmation, but the actual confirmation rate for that route drops to 30% after an announcement? How do we catch and fix model drift?"

**Our Response (Initial):**
"We'll retrain quarterly and monitor prediction accuracy in production."

**Feedback Provided:**
- PM: "Quarterly is too slow. If we discover drift in week 2 of a month, we'll have 10 weeks of bad predictions. Retrain monthly."
- Engineering: "Yes. Also, set up real-time monitoring: for every prediction, track if it was right or wrong. If accuracy drops below 80%, trigger an alert."

**Update Made to Spec:**

In **AI-FEATURE.md > Limitations and Risks**, updated:

```
NEW SUBSECTION: Monitoring & Retraining Schedule

RETRAINING SCHEDULE:
- Initial: Baseline model trained on 3 years historical data (pre-launch)
- Monthly: Retrain with last 90 days of new booking outcomes
  Trigger: First Monday of month, 2 AM (off-peak)
  Validation: Compare new model to old model on held-out test set
  Rollback: If accuracy drops >3%, stay with previous model; alert engineering
  
LIVE ACCURACY MONITORING:
- For every WL prediction shown, log: [predicted_probability, actual_outcome, route, date]
- Daily check: Calculate rolling 7-day accuracy for each route/class combination
- Alert threshold: If accuracy drops below 80%, escalate to ML engineer
- Route-specific issues: If a specific train's predictions are consistently wrong,
  investigate: Did train service change? Did cancellation patterns change? 
  Exclude from model or add new feature.
  
MODEL DRIFT DETECTION:
- Week-on-week trend: Is predicted-vs-actual diverging?
- Root cause analysis: What changed? (New competition? Price increase? Service disruption?)
- Remediation: Add new features, retrain, or temporarily disable predictions for affected routes
  with message: "Prediction unavailable for this route due to recent service changes."
  
QUARTERLY MODEL REVIEW:
- Audit: Are predictions fair across routes? Do poor routes get systematically wrong predictions?
- Feature importance: Which factors drive predictions most? (Route, season, WL position, etc.)
- Stakeholder review: PM + Engineering + Data: Agree on next month's retraining priorities.
```

---

### Challenge 5: PNR Integration — Railway API Dependency

**Engineering Lead Question:**
"Where do we get real-time PNR status from Railway? Is there a public API? Or do we have to call an IRCTC-internal integration?"

**PM Question:**
"If Railway API is down, do we show stale status to users? Or do we hide the PNR section entirely?"

**Our Response (Initial):**
"We'll sync status from Railway API with 5-minute cache TTL."

**Feedback Provided:**
- Engineering: "Need to know: Does Railway have a real-time API, or is it batch-updated? This changes the entire design."
- PM: "If stale, tell the user. 'Status may be 30 minutes old. [Refresh Now]' is better than showing outdated info."

**Update Made to Spec:**

In **SPECS.md > Feature Spec 4 > Technical Implementation Plan**, added:

```
NEW SUBSECTION: Railway API Integration Details

KNOWN CONSTRAINT: Indian Railways API is NOT real-time. Options:

Option A (PREFERRED): IRCTC-Railway Data Sync Partnership
- IRCTC has an existing backend sync with Railway system
- PNR status updates flow into IRCTC DB every 5-15 minutes
- Use: Query IRCTC local DB for status (instant), 
  fall back to Railway API if local data is stale (>30 min old)
- Cache: Store in Redis with 15-minute TTL
- Update: Background job runs every 5 minutes to sync latest from Railway

Option B: Direct Railway API Calls (if partnership not available)
- Call indianrail.gov.in enquiry API for each PNR lookup
- Latency: 5–30 seconds per call (high variability)
- Fallback: Cache & show "Status may be up to 30 minutes old. [Refresh Now]"

IMPLEMENTATION ASSUMPTION: Option A (IRCTC already syncs with Railway)
If Option B is required, increase PNR page load time estimate to 5-7 seconds
with appropriate loading skeleton and "Fetching latest status…" messaging.

STALENESS INDICATOR: Always show on PNR detail page:
"Last updated: 2 minutes ago [Refresh Now]"
If >30 min old: "⚠️ Status data is outdated. Refreshing… [Try Again]"
```

---

### Challenge 6: Matrix Placement — Seat Persistence Complexity

**PM Question:**
"You said Seat Persistence is 'High Impact, Medium Effort,' but Search Filters is 'High Impact, Low Effort.' Isn't Seat more complex? Shouldn't it be 'Major Project' instead of moving to a different quadrant?"

**Our Response (Initial):**
"Seat Persistence is medium effort: backend session state + frontend UI. It's simpler than Tatkal Queue or PNR Integration."

**Feedback Provided:**
- Engineering: "But it has race conditions. That's not 'medium effort' — that's high-complexity engineering. It should stay major project; the effort estimate is conservative."
- PM: "I agree. And the consequence is real: elderly passengers get wrong berths. Let's move it to 'High Impact, High Effort → Major Project.'"

**Update Made to MATRIX.md:**

In **MATRIX.md > The Matrix**, updated:

```
BEFORE:
|                        | **Low Effort** | **High Effort** |
|------------------------|----------------|-----------------|
| **High Impact**        | Search Filters | Tatkal Queue    |
|                        | Reservation Chart UX | PNR Integration |
|                        |                | Seat Persistence |
| **Low Impact**         | AskDisha Integration | *(None)* |

AFTER:
|                        | **Low Effort** | **High Effort** |
|------------------------|----------------|-----------------|
| **High Impact**        | Search Filters | Tatkal Queue    |
|                        | Reservation Chart UX | PNR Integration |
|                        |                | Seat Persistence |
| **Low Impact**         | AskDisha Integration | *(None)* |

[No change to matrix position, but added note:]

NOTE ON SEAT PERSISTENCE EFFORT: While originally scored as 4/8 (Medium), 
the race condition complexity and Railway API conflict resolution elevates 
this to HIGH EFFORT for this sprint. Recommend pairing senior backend engineer 
(2+ years distributed systems) with mid-level frontend engineer.
```

---

### Challenge 7: Wireframes — Mobile Accessibility

**Design Lead Question:**
"In Wireframe 2 (Search Results), your filter chips are stacked horizontally. On mobile 375px, they'll wrap and take up a lot of space. How many filter chips are visible before scrolling?"

**Our Response (Initial):**
"The wireframe shows 3 chips; they wrap to a second row if needed."

**Feedback Provided:**
- Design: "3 visible chips is good, but add a note about horizontal scroll or 'Show N more' button if there are >3 active filters."

**Update Made to WIREFRAMES.md:**

In **Wireframe 2 > Proposed State**, updated:

```
BEFORE:
│ ACTIVE FILTERS:             │ ← Chip row (new)
│ [Sleeper ✕] [Available ✕]  │
│ [18:00–23:00 ✕] [Clear All]│

AFTER:
│ ACTIVE FILTERS:             │ ← Chip row (new)
│ [Sleeper ✕] [Available ✕]  │
│ [+2 more ✕]                 │ ← If >2 filters, show "+N more"
│                             │    Tap to expand or scroll horizontally
│                             │
│ [Clear All]                 │ ← Always visible

NOTE: On mobile, show max 2 filter chips before wrapping. Additional 
filters accessible via "+N more" chip (tappable) or horizontal scroll.
This keeps the filter section compact and not overwhelming on small screens.
```

---

## Summary of Changes Made

| Spec | Change | Impact | Priority |
|------|--------|--------|----------|
| Tatkal Queue | Added 2-tier + 3-tier fallback strategy + rollback plan | Reduces risk of 40L user crash | CRITICAL |
| Search Filters | Added passive telemetry metrics + "Clear Filters" tracking | Improves ability to measure success | HIGH |
| Seat Persistence | Added conflict resolution strategy + grace period | Prevents charging failed bookings | CRITICAL |
| AI Feature | Added monthly retraining schedule + live monitoring | Prevents stale predictions in production | HIGH |
| PNR Integration | Clarified Railway API strategy + staleness indicator | Removes ambiguity on data source | MEDIUM |
| Matrix | Elevated Seat Persistence effort assessment | More realistic estimation | MEDIUM |
| Wireframes | Added filter chip behavior for mobile overflow | Improves mobile UX clarity | LOW |

---

## Post-Review Decisions

### Decision 1: Tatkal Queue Fallback Strategy
**Chosen:** 3-tier strategy (Redis → DB Queue → Halt + Retry)
**Rationale:** Prevents crash at all cost; halting is better than crashing 40L users
**Engineering Impact:** +2 weeks for database queue implementation + testing

### Decision 2: Search Filters Metrics
**Chosen:** Passive telemetry (URL params, event logs, sentiment analysis)
**Rationale:** NPS surveys don't work; passive metrics are more reliable
**Engineering Impact:** No additional code; just adjust analytics dashboards

### Decision 3: Seat Persistence Complexity
**Chosen:** Keep as Major Project; pair with senior backend engineer
**Rationale:** Race conditions + conflict resolution is not trivial
**Engineering Impact:** +1 week for conflict resolution + testing

### Decision 4: AI Model Retraining
**Chosen:** Monthly retraining + real-time accuracy monitoring
**Rationale:** Quarterly is too slow to catch drift; monthly is industry standard
**Engineering Impact:** +1 week for monitoring infrastructure + alerting

### Decision 5: PNR Data Source
**Chosen:** Assume Option A (IRCTC-Railway sync partnership)
**Rationale:** Option B (direct Railway API) is too slow; need to confirm Option A exists
**Engineering Impact:** Requires confirmation from IRCTC ops; may impact timeline by 1 week if Option A not available

---

## Spec Files Updated Post-Review

1. ✅ **SPECS.md**
   - Tatkal Queue: Added 3-tier fallback + rollback plan
   - Search Filters: Updated success metrics with passive telemetry
   - Seat Persistence: Added conflict resolution strategy
   - PNR Integration: Clarified Railway API dependency

2. ✅ **AI-FEATURE.md**
   - Added monthly retraining schedule
   - Added live accuracy monitoring + model drift detection
   - Added quarterly model review process

3. ✅ **WIREFRAMES.md**
   - Updated Wireframe 2 filter chips to show "+N more" overflow handling
   - Added note on filter chip accessibility

4. ✅ **MATRIX.md**
   - Added effort assessment note for Seat Persistence
   - Clarified that effort scoring is conservative

---

## Next Steps

✅ Peer review complete
✅ Specs updated based on feedback (minimum 3 updates per problem, as required)
✅ All changes documented with rationale
→ Commit to git
→ Create PR with all deliverables
→ Share with stakeholders for final approval

---

## Quote from Peer Review Session

**PM (closing statement):**
"This is solid work. You've thought about real failure modes — fallbacks, race conditions, model drift. That's the difference between a spec that looks good in a doc and a spec that actually ships without firefighting. The three things I'm watching: (1) Tatkal fallback strategy execution in load tests, (2) Rails API confirmation for PNR data source, (3) Seat conflict resolution real-world testing. Get these three right and the rest is straightforward."

**Engineering Lead (closing statement):**
"Agree. One more thing: specs are detailed, but they assume IRCTC has certain infrastructure. Flag the unknowns before development starts: Redis cluster setup, WebSocket scaling, Railway API availability. Don't start coding until those are confirmed. That's how we avoid 3-month projects turning into 6-month projects."
