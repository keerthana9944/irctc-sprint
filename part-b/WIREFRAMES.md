# Wireframes

This file contains mid-fidelity wireframe templates and concrete ASCII-style examples you can use immediately. Follow the instructions below for each of the six UI-related problems documented in `part-a/PROBLEMS.md`.

Wireframe rules (required)
- Mobile width: 375px (primary target).
- Label every component (e.g., "Queue counter", "Filter chip").
- Annotate interactions (e.g., "Tap → open train details").
- Include before state, after state, loading/empty/error states.
- Export final wireframes as PNGs and save in `assets/wireframes/`.

How to use
1. For each problem in `part-a/PROBLEMS.md`, duplicate one section below and replace placeholders with the problem title, context, and image file.
2. Create the visual wireframe in Figma/Excalidraw at 375px width, label components and interactions, export PNG, then save to `assets/wireframes/`.
3. Update the `Wireframe` image link in the corresponding section below.

---

## Example ASCII Wireframe — Search Results with Filter Chips

Context: Improves search filter persistence and result confidence indicator. Use this ASCII example as a template for other wireframes.

```
┌─────────────────────────────────────────┐
│ IRCTC          [🔍]  [👤]              │
├─────────────────────────────────────────┤
│ [New Search]                             │
├─────────────────────────────────────────┤
│ 🧭  RESULTS (12 trains)                  │  ← Count updated
│                                         │
│ ACTIVE FILTERS:                         │  ← Chip row (new)
│ [Sleeper ✕] [Available ✕]               │
│ [18:00-23:00 ✕] [Clear All]             │
│                                         │
│ 📊 RESULTS MATCH FILTERS: 82%           │  ← Confidence label (new)
│                                         │
│ • Train 12622 Tamil Nadu                │  ← Consistent results
│   ↓ Sleeper - Available - CNF           │
│   ↓ Chennai→Delhi - ₹2400               │
│                                         │
│ • Train 12625 Kerala Exp                │
│   ↓ Sleeper - CNF - ₹2380               │
│                                         │
│ • Train 12627 Bangalore Exp             │
│   ↓ Sleeper - Available                 │
│                                         │
│ [Show 3 more] / [Load +5]               │
├─────────────────────────────────────────┤
│ ⚙ [Filters]  [Sort: Time ▾]             │  ← Bottom filter bar
└─────────────────────────────────────────┘
```

Annotations (for above)
- `RESULTS (12 trains)`: show live count updated after filters applied.
- `ACTIVE FILTERS`: persistent chip row that survives navigation and reload.
- `RESULTS MATCH FILTERS`: confidence label computed from how many results match strict filter criteria.
- Each train row: consistent layout, status line (CNF/Available/WL), route and price.
- Bottom bar: primary access to filters and sort controls.

Wireframe section template

## Wireframe — [Problem Title]

### Context
Reference `part-a/PROBLEMS.md` — replace with the exact problem reference.

### Before (current broken UI)
- Short description of the current screen and failure point.

### After (proposed UI)
![Wireframe: Problem Title](../assets/wireframes/wireframe-[n].png)
*Caption: Mobile view — proposed layout.*

### Components (labelled)
- Header: IRCTC logo + profile
- Primary action: [e.g., "Search trains"]
# Wireframes

This file contains mid-fidelity wireframe templates and concrete ASCII-style wireframes for each of the six problems documented in `part-a/PROBLEMS.md`.

Wireframe rules (required)
- Mobile width: 375px (primary target).
- Label every component (e.g., "Queue counter", "Filter chip").
- Annotate interactions (e.g., "Tap → open train details").
- Include before state, after state, loading/empty/error states.
- Export final wireframes as PNGs and save in `assets/wireframes/`.

How to use
1. For each problem in `part-a/PROBLEMS.md`, confirm the problem title and replace the placeholder headings below.
2. Use the ASCII wireframes as an exact layout guide for the Figma/Excalidraw design (375px width). Label components and add interaction annotations in the design file.
3. Export PNG named `wireframe-[short-name].png` → save to `assets/wireframes/` → update the image link in this file and in `part-b/SPECS.md`.

---

## Wireframe 1 — Tatkal Booking Crashes at 10:00 AM

### Context
Problem 1 from `part-a/PROBLEMS.md`: Tatkal booking fails at 10:00 AM due to lack of a queue/waiting-room and no clear feedback.

### After (proposed UI) — Tatkal Virtual Queue (mobile)

```
┌─────────────────────────────────────────┐
│ IRCTC          [🔍]  [👤]              │
├─────────────────────────────────────────┤
│  TATKAL OPENS IN  00:04:23              │  ← Live countdown
├─────────────────────────────────────────┤
│  YOU ARE IN QUEUE                         │
│  Position: #4,281    ETA: ~9m            │
│  ████████░░░░░░░░░░  Progress bar       │
│  Tip: Pre-loading seat map & details     │
├─────────────────────────────────────────┤
│  WHEN YOUR TURN → 90s BOOKING WINDOW     │
│  [Seat map preview]  [Passenger preview] │
│  [Change Train]  [Update Passengers]     │
├─────────────────────────────────────────┤
│  Status: Connected (WS)  | Fallback: Poll every 5s
└─────────────────────────────────────────┘
```

Annotations
- Live countdown visible before 10:00; joins queue when user enters Tatkal page between 9:55–10:05.
- Client connected to `tatkal:queue:[queueId]` WebSocket channel; fallback to polling when network weak.
- When turn arrives, UI opens pre-loaded seat map and starts a 90s countdown (visual + haptic cue).

Edge states
- Offline: show banner "Connection lost — retry" and fallback polling.
- Queue fail (Redis down): show a clear message and allow direct booking attempt with rate limit.

---

## Wireframe 2 — Search Filters Do Not Work Reliably

### Context
Problem 2 from `part-a/PROBLEMS.md`: filters reset or show inconsistent results after refresh or reload.

### After (proposed UI) — Search results with persistent filter chips

```
┌─────────────────────────────────────────┐
│ IRCTC          [🔍]  [👤]              │
├─────────────────────────────────────────┤
│ [New Search]                             │
├─────────────────────────────────────────┤
│ 🧭 RESULTS (12 trains)                   │ ← Count updates when filters apply
│                                         │
│ ACTIVE FILTERS:                         │ ← Persistent chip row (sessionStorage)
│ [Sleeper ✕] [Available ✕] [18:00-23:00 ✕]
│ [Clear All]                             │
│                                         │
│ 📊 RESULTS MATCH FILTERS: 82%           │ ← Confidence label
│                                         │
│ • Train 12622 Tamil Nadu                │
│   ↓ Sleeper - Available - CNF           │
│   ↓ Chennai→Delhi - ₹2400               │
│                                         │
│ [Show 3 more] / [Load +5]               │
├─────────────────────────────────────────┤
│ ⚙ [Filters]  [Sort: Time ▾]             │
└─────────────────────────────────────────┘
```

Annotations
- Filters stored in `sessionStorage`; re-applied on reload and back navigation.
- Confidence label computed client-side from result count / filter match percentage.
- Filter chip tap toggles filter state and triggers incremental fetch (debounced 300ms).

Edge states
- Slow network: disable chips briefly and show spinner inside chip while searching.

---

## Wireframe 3 — Seat Selection Resets Randomly

### Context
Problem 3 from `part-a/PROBLEMS.md`: selected berth disappears between seat map and passenger details.

### After (proposed UI) — Seat map + selection persistence

```
┌─────────────────────────────────────────┐
│ Train 12622  | Sleeper | 2 Pax           │
├─────────────────────────────────────────┤
│ [Seat map grid]                          │
│  ┌─────────────┬─────────────┐           │
│  │ [A1] [A2]    │ [B1*] [B2] │  *selected│
│  │ [LWR] [MDR]  │ [UPR] [RAC] │           │
│  └─────────────┴─────────────┘           │
├─────────────────────────────────────────┤
│ Selected: B1 (Lower)  • Price: ₹150      │
│ [Proceed]  [Auto-assign fallback]        │
├─────────────────────────────────────────┤
│ Passenger details (preview)              │
│ Name: Keerthana  • Age: 29               │
└─────────────────────────────────────────┘
```

Annotations
- Selected seat added to a client-side booking state object and written to a short-lived backend reservation token (10s) to lock the seat before the next step.
- On Proceed: UI verifies the reservation token; if token invalid, show clear message and let user choose again.

Edge states
- Token expired: show modal "Seat locked by other user — reselect or Auto-assign" with retry action.

---

## Wireframe 4 — PNR Status Lives on a Separate Legacy Site

### Context
Problem 4 from `part-a/PROBLEMS.md`: PNR status redirects to a separate legacy domain, breaking continuity.

### After (proposed UI) — Integrated PNR check panel

```
┌─────────────────────────────────────────┐
│ IRCTC          [🔍]  [👤]              │
├─────────────────────────────────────────┤
│  My Trips   |  PNR Status                │
├─────────────────────────────────────────┤
│  Check PNR                                │
│  [ Enter 10-digit PNR ]  [ Check ]        │
│                                         │
│  Last checked: 1234567890  • Status: CNF  │
│  [View seat map]  [Share]                │
└─────────────────────────────────────────┘
```

Annotations
- PNR check performed inside main app via server-to-server call to Indian Railways enquiry API; results shown inline to maintain UX continuity.
- Include quick actions (seat map, cancellation rules) without leaving the site.

Edge states
- External API rate-limited: show cached PNR results and a timestamp, plus a retry option.

---

## Wireframe 5 — Reservation Chart Search Is Cryptic

### Context
Problem 5 from `part-a/PROBLEMS.md`: reservation chart page shows confusing empty-state and assumes train knowledge.

### After (proposed UI) — Guided Reservation Chart lookup

```
┌─────────────────────────────────────────┐
│ Reservation Chart                         │
├─────────────────────────────────────────┤
│ Train (name or number): [ Type to search ]│
│ Suggestions: 12622 Tamil Nadu Exp        │
│ Journey date: [YYYY-MM-DD]               │
│ Boarding station: [Select]               │
│ [Get Chart]                              │
├─────────────────────────────────────────┤
│ Hint: Type city or train number.         │
│ Example: "12622" or "Chennai Mail"    │
└─────────────────────────────────────────┘
```

Annotations
- Combobox shows helpful examples and recent searches; empty state reads as a prompt not an error.
- Accessibility: first focus shows a short instructional message rather than "0 results".

Edge states
- No matches: show suggestions and a link to help on finding train numbers.

---

## Wireframe 6 — AskDisha Support Layer Competes With Booking Flow

### Context
Problem 6 from `part-a/PROBLEMS.md`: the AskDisha assistant appears as a floating layer and causes visual/technical interference.

### After (proposed UI) — Non-intrusive Help (collapsible)

```
┌─────────────────────────────────────────┐
│ IRCTC          [🔍]  [👤]   [? help]     │
├─────────────────────────────────────────┤
│ [Search form]                             │
│ From: [    ]  To: [    ]  Date: [    ]   │
├─────────────────────────────────────────┤
│ • Results list                             │
│                                           │
├─────────────────────────────────────────┤
│ [ Help bottom-sheet collapsed ] [?]       │ ← tappable help icon
└─────────────────────────────────────────┘
```

Annotations
- Replace floating assistant with a single tappable help icon or bottom-sheet that expands on demand. Remove auto-loading embedded frames to avoid cross-origin frame errors.
- If user opens help, show lightweight FAQ and an option to open full AskDisha in a separate context (no frames blocking booking UI).

Edge states
- Frame blocked/cross-origin errors: fall back to server-served help content and log error for telemetry.

---

## Export checklist
- Create a Figma/Excalidraw file for each wireframe at 375px width following the ASCII layout above.
- Label components and add interactions in the design file.
- Export PNG named `wireframe-[short-name].png` and save to `assets/wireframes/`.
- Update the corresponding `![Wireframe]` image links in this file and in `part-b/SPECS.md`.
