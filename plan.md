# Factory Simulation — Living Factory Roadmap

## Confirmed System Model

Based on Orchestrating-the-Living-Factory.txt + clarified assumptions:

```
[Propagation/Growth] --F1--> [Buffer Warehouse] --F2--> [Pollination Zone] --F3--> [Post-Pollination]
```

### Stocks

| # | Name | Description | Key Constraint |
|---|---|---|---|
| S1 | Propagation / Growth | Upstream source; plants ready for pollination staging | Inflow sustains steady-state; not the bottleneck |
| S2 | Buffer Warehouse | Primary holding area; two sub-populations co-exist | 48-hr max dwell for priority plants |
| S3 | Pollination Zone | WIP during active pollination sessions | Batch = 5 plants; session = 45 min |
| S4 | Post-Pollination | Downstream recovery/seed-development; exits active system | Not modeled in detail for Phase 2 |

### Flows

| # | From → To | Name | Key Modeling Note |
|---|---|---|---|
| F1 | Propagation → Buffer Warehouse | Inflow | Sustained at ~throughput demand; user-configurable |
| F2 | Buffer Warehouse → Pollination Zone | Transfer (bottleneck) | Only 66.7% efficient — 33% waste on prep + transit |
| F3 | Pollination Zone → Post-Pollination | Completion outflow | Rate limited by F2 actual + station capacity |

### Sub-population: Buffer Warehouse

- **Regular plants** — 80% of inflow; no hard deadline
- **Priority/flowering plants** — 20% of inflow; **48-hour dwell limit**
  - Same physical infrastructure; prioritized by scheduling logic
  - If not moved within 48 sim-hours → **deadline failure** (lost breeding cycle)

### The 33% Labor Utilization Problem

- All plants in a session are successfully pollinated — there is no scrap or yield loss
- The problem is **technician time**: 15 of every 45 minutes is spent on logistics (cart retrieval, transit, staging, setup), not pollination
- This caps F2's achievable throughput at 66.7% of theoretical station capacity
- Modeled as: `F2.maxRate = stationCapacity * laborUtilization` where `laborUtilization = 0.667`
- Logistics automation restores the 33% → same workers, same stations, ~50% more plants processed per shift
- UI label: "Station utilization: X%" — not "waste" or "efficiency factor"

---

## Phase 1 — Complete

**File:** `index.html`

Generic 2-stock / 3-flow SVG simulation:
- Animated particles, fill-level tanks, backpressure, drain guard
- Live flow rate controls, Start/Pause/Reset
- Data panel with 6 live readings

---

## Phase 2 — Living Factory Model

**File:** `living-factory.html` (new file; keep Phase 1 intact)

### New State Shape

```js
const state = {
  stocks: [
    { name: 'Propagation',      value: 500,  capacity: 1000 },
    { name: 'Buffer Warehouse', value: 200,  capacity: 800,
      priority: { value: 40, capacity: 200, ageHours: 0 } },  // priority sub-pop
    { name: 'Pollination Zone', value: 0,    capacity: 50  },
    { name: 'Post-Pollination', value: 0,    capacity: 1000 }
  ],
  flows: [
    { name: 'Inflow',      rate: 71,  actual: 0 },  // ~500k/week ÷ 7 days, sim units
    { name: 'Transfer',    rate: 71,  actual: 0, laborUtilization: 0.667 },  // max throughput = stationCapacity * laborUtilization
    { name: 'Completion',  rate: 47,  actual: 0 }
  ],
  deadlineFailures: 0,   // priority plants that exceeded 48-hr window
  priorityAge: 0,        // running sim-hours clock for oldest priority cohort in buffer
  simSpeed: 5,           // 1 real second = N sim-minutes
  ...
}
```

### Key New Mechanics

**Simulation time scaling**
- `simSpeed` slider: 1 real sec = N sim-minutes (default 5, range 1–60)
- 48-hour deadline plays out in ~9.6 real minutes at default speed
- All biological timers and age clocks run on sim-time, not wall-clock

**Priority deadline tracking**
- `priorityAge` advances by `dt * simSpeed / 60` sim-hours per frame
- When `priorityAge >= 48`: priority plants still in buffer → failed; `deadlineFailures++`; UI alert fires
- Age indicator turns red at 40 sim-hours (80% of window consumed)

**F2 labor utilization cap**
- `F2.actual = min(stationCapacity * laborUtilization, buffer.value / dt)`
- `stationCapacity` = theoretical max if workers spent 100% of time pollinating
- `laborUtilization` = 0.667 (baseline); slider to 1.0 models logistics automation
- UI shows: "Station capacity: X u/s", "Utilization: 66.7%", "Effective rate: Y u/s"
- No plants are lost — the cap is purely a throughput rate constraint on skilled labor time

**Priority routing**
- F2 pulls from priority sub-stock first; falls back to regular if priority is empty
- Priority particles: gold diamond shape
- Regular particles: colored circles (as Phase 1)

### SVG Layout (1200×500 viewBox)

```
[Propag.]  --F1-->  [Buffer Warehouse]       --F2-->  [Pollination Zone]  --F3-->  [Post-Poll.]
  x=70             x=270  (wider, dual fill)           x=680                         x=950
                   └─ gold band = priority plants (stacked at top of fill)
```

### New Data Panel Cards

| Card | Color |
|---|---|
| Priority in Buffer (count + %) | gold if > 0 |
| Deadline Failures | red if > 0, else green |
| Oldest Priority Age (sim-hrs) | red if > 40 |
| Effective Transfer Rate (vs. theoretical max) | normal |
| Sim Speed (multiplier) | normal |

---

## Build Order — Phase 2

- [x] Scaffold `living-factory.html` from Phase 1 structure
- [x] Add sim-time scaling + speed slider
- [x] Expand state to 4 stocks + priority sub-population
- [x] Update `updateSimulation` with labor utilization cap + priority routing
- [x] Add priority age clock + deadline failure counter + UI alert
- [x] Redesign SVG for 4 stocks; Buffer Warehouse dual-fill (blue regular + gold priority)
- [x] Add priority particle variant (gold diamond)
- [x] Add new data panel cards (12 cards, 4 columns)
- [x] Verify 500k/week steady-state at default rates
- [x] Verify deadline failures appear when F2 rate is too low

---

## Verification

| Scenario | Expected |
|---|---|
| Default rates | Buffer Warehouse steady; no deadline failures |
| F2 rate cut below F1 | Buffer fills; priority age accelerates; failures after 48 sim-hrs |
| laborUtilization → 1.0 | Transfer rate increases ~50% (logistics automation frees technician time) |
| Priority routing | Gold particles always depart before blue when both present |
| Sim speed × 10 | 48-hr deadline reached in ~4.8 real minutes |

---

## Phase 3: Unit Calibration (Complete)

**1 unit = 1,000 plants (k plants). Flow rates in k plants/sim-hr.**

- [x] `HOURS_PER_SHIFT=8`, `POLL_CAP=50`, `TARGET_WEEKLY=500`
- [x] Params: `f1Rate=6.25`, `stationCap=9.4`, `laborUtil=0.667`, `f3Rate=6.25`, `simSpeed=8` (sim-hrs per 5 real-secs), `daysPerWeek=5`, `shiftsPerDay=2`
- [x] `simDtHrs = wallDt * simSpeed / 5`
- [x] All stock updates and backpressure use `simDtHrs`; particle position still uses `wallDt`
- [x] Spawn accumulators use `simDtHrs`
- [x] `freshState`: `bufReg=200`, `bufPri=50`, `poll=5`
- [x] Days/week and Shifts/day inputs added to controls
- [x] All unit labels updated to "k plants" and "k/hr"
- [x] "Automation ROI" card replaced with "Weekly Throughput" (green ≥500k, red <500k)
- [x] Deadline text uses named DOM ref (`id="deadline-text"`)

**Key verification:**
- Default settings → Weekly Throughput card shows ~500 k plants/wk (green)
- Sim runs 5 real-secs → clock advances ~8 sim-hours
- Drop laborUtil to 40% → weekly output falls below 500k (red card)
- Raise shiftsPerDay 2→3 → target rate drops, system achieves 500k with lower F1

---

## Phase 4: UX Polish (Complete)

- [x] F1 and F3 inputs changed from k plants/hr to **k plants/week** (default 500, max 5000, step 25)
  - `syncFlowsFromWeekly()` converts weekly input → k/hr and stores in `params.f1Rate`/`params.f3Rate`
  - Conversion display (`.rate-convert`) shown below each input: "= X.XX k/hr"
  - daysPerWeek and shiftsPerDay listeners also call `syncFlowsFromWeekly()` so conversion updates when schedule changes
- [x] `CLAUDE.md` added to repo root with architecture guide for future Claude sessions

---

## Phase 5: Shift-Aware Scheduling, Weekly Station Capacity & Per-Cohort Priority Deadlines (Complete)

- [x] `HOURS_PER_SHIFT` constant removed; replaced by `params.hoursPerShift = 8` (user-adjustable)
  - New input field: "Hours / shift" (min 1, max 24, default 8) in Schedule & Simulation section
  - `syncFlowsFromWeekly()` now uses `params.hoursPerShift`; listener calls `syncFlowsFromWeekly()` on change
- [x] Station capacity changed from k plants/hr to **k plants/week** (default 752 = 9.4 k/hr × 80 hrs/wk)
  - `syncFlowsFromWeekly()` extended to convert capWkly → `params.stationCap` (k/hr) and update `#stationCap-convert` display
  - Data panel and SVG util label still show derived k/hr value
- [x] Shift-aware F2 gating via `isShiftActive()` helper
  - Working days: first `daysPerWeek` of each 168-sim-hr week; shifts: first `shiftsPerDay × hoursPerShift` hrs of each working day
  - F2 = 0 when `!shiftOn`; F1 and biological aging run continuously
  - `sim.shiftOn` stored in state for `renderFrame()` — dims F2 path + label to 35% opacity off-shift
  - "ON SHIFT" / "OFF SHIFT" text indicator (`#shift-status`) added to SVG below F2 arrow
- [x] Per-cohort priority deadline tracking replaces global `prAge` clock
  - `sim.priorityCohorts: [{amount, deadline}]` — 1 k-plant batches, created via `priArrivalAccum` accumulator
  - Each cohort's `deadline = arrival_time + 48 sim-hrs` (fixed at batch creation, not a rolling reset)
  - Dispatch order: earliest deadline first (most urgent priority cohorts), then regular plants FIFO
  - `sim.prAge` derived each tick from oldest surviving cohort; deadline bar and data panel unchanged
  - `sim.bufPri` recomputed from cohort sum + `priArrivalAccum` each tick
