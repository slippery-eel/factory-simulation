# Living Factory Simulation

A browser-based system-dynamics simulation of the Ohalo "Living Factory" seed production pipeline. Built to make the scheduling trade-offs in a high-throughput plant pollination operation viscerally visible — no installation, no build step, just open an HTML file in a browser.

---

## Quick Start

```
git clone https://github.com/slippery-eel/factory-simulation
```

Open any file directly in any modern browser.

| File | What it is |
|---|---|
| `simulation.html` | **Dynamic Simulation** — real-time worker-session model with animated SVG diagram, shift scheduling, and throughput chart. Primary working file. |
| `opportunity-analysis.html` | **Opportunity Analysis & ROI** — static side-by-side scenario comparison calculator for logistics vs. workstation automation. |
| `layout.html` | **Layout** — interactive floorplan view of the pollination facility with routing logic and ergonomics comparison. |
| `index.html` | Generic 2-stock/3-flow prototype (Phase 1), kept for reference. |
| `living-factory.html` | Continuous-flow model with labor utilization slider (Phase 2/3), kept for reference. |

The three main pages link to each other via a tab bar at the top of each page.

---

## Background: The Problem Being Modeled

Ohalo's "Living Factory" produces seeds at scale through a controlled pollination process. Plants move from propagation through a buffer warehouse, into a pollination zone, and then into post-pollination recovery. The critical constraints are:

1. **A 48-hour biological window** — certain plants are at peak flowering and must be pollinated within 48 hours of entering the buffer, or the breeding cycle is lost.
2. **Session-based labor** — technicians work in discrete pollination sessions: collect a cart of plants, spend time on setup and transport (15 min default), actively pollinate (30 min default), then return. This session structure means each worker processes a fixed batch size per cycle rather than a continuous stream.
3. **Shift scheduling** — work only happens during scheduled shifts. Plants age around the clock regardless, so long off-shift gaps erode the 48-hour biological window for priority plants.
4. **Output lag** — at the start of each shift, workers begin sessions immediately, but the first completions don't appear until one full flow-time cycle later (45 min by default). This lag is transient: sessions dispatched near shift end still complete after the shift closes, so it does not reduce weekly capacity.

---

## The System Model

```
[Propagation] --F1--> [Buffer Warehouse] --F2--> [Pollination Zone] --F3--> [Post-Pollination]
```

### Stocks (the four tanks)

| Stock | Description | Capacity |
|---|---|---|
| **S1 — Propagation** | Upstream source of plants ready for staging. Treated as effectively infinite in this model. | Unlimited |
| **S2 — Buffer Warehouse** | Primary holding area before pollination. Contains two sub-populations: regular (blue fill) and priority (gold fill stacked on top). | 800 k plants |
| **S3 — Pollination Zone** | Plants currently in active pollination sessions. Subdivided into a **queue** (waiting for a free worker slot) and **active WIP** (in-flight batches). | Bounded by numWorkers × plantsPerSession |
| **S4 — Post-Pollination** | Cumulative count of plants that have completed pollination and entered seed development. | Unbounded (cumulative) |

### Flows

| Flow | Path | Description |
|---|---|---|
| **F1 — Inflow** | Propagation → Buffer | Plants arriving from the growth stage. Rate is derived from the weekly target divided by scheduled hours per week. Only active during scheduled shifts. |
| **F2 — Transfer** | Buffer → Pollination Zone | Plants moving from the buffer into the pollination queue. Capped by queue capacity (numWorkers × plantsPerSession). Only active during shifts. Dispatches priority cohorts first (earliest deadline first), then regular plants. |
| **F3 — Completion** | Pollination Zone → Post-Pollination | Batches completing their session and exiting the pollination zone. **Not shift-gated** — batches dispatched near the end of a shift complete after shift end. |

### Units

All quantities are in **k plants** (thousands of plants). Flow rates are displayed as **k plants / sim-hour** in the SVG labels. 1 sim unit = 1,000 real plants.

---

## Priority Plants and the 48-Hour Deadline

When plants enter the Buffer Warehouse via F1, 20% are flagged as **priority** — flowering-stage plants with a hard biological deadline. The remaining 80% are regular plants with no strict time constraint.

**How priority tracking works:**

- Priority plants accumulate in a cohort accumulator. When the accumulator crosses 1 k-plant, a new cohort is created and stamped with a deadline: `arrival_time + 48 sim-hours`.
- An initial cohort of 50 k-plants is seeded at t=0 with a 48-hr deadline.
- The cohort queue is sorted by deadline ascending — the most urgently expiring cohort is always dispatched first when F2 has capacity.
- Cohorts age continuously, **including during off-shift hours**. Biological clocks do not pause for weekends or overnight gaps.
- When any cohort's deadline passes before it is moved, those plants are counted as **deadline failures** — a lost breeding cycle.
- The **Priority Age** SVG card shows the elapsed sim-hours since the oldest surviving cohort arrived (i.e., how close the most urgent cohort is to its 48-hour limit). Color transitions: green (≤ 30h) → amber (30–40h) → red (> 40h).
- If failures occur, a red pulsing alert banner appears at the top of the page.
- On Reset, the initial 50 k-plant cohort is restored.

The gold band in the Buffer Warehouse tank represents priority plants stacked on top of the regular (blue) fill.

---

## Dynamic Simulation (`simulation.html`)

The simulation tab uses a **discrete-session model** for the pollination station.

### How it works

Each pollination session has two phases:
- **Setup / Transport** (default 15 min): retrieving carts, transit, staging
- **Active Pollination** (default 30 min): the actual pollination work

**Total flow time** = setup + pollination = 45 min (0.75 sim-hr) at defaults.

Workers are modeled as slots. At each simulation tick during a shift:
1. Completed batches are drained to Post-Pollination (F3).
2. Free worker slots are filled from the queue — each slot takes `plantsPerSession` k-plants and schedules completion at `elapsed + totalFlowTime`.

### Worker count derivation

```
sessionsPerWeek = target / plantsPerSession
numWorkers      = ⌈ totalFlowTime × sessionsPerWeek / hoursPerWeek ⌉
```

Workers are derived from scheduled hours per week without subtracting any startup penalty. The output lag at shift start (one `totalFlowTime` cycle of zero F3) is a transient effect visible in the throughput chart, not lost weekly capacity.

### Key derived quantities (defaults)

| Quantity | Formula | Default value |
|---|---|---|
| Total flow time | setup + pollination | 0.75 hr |
| Scheduled hrs/week | 5d × 2s × 8h | 80 hr |
| F1 rate | target ÷ hoursPerWeek | 6.25 k/hr |
| Sessions/week | target ÷ plantsPerSession | 10,000 |
| Workers required | ⌈ 0.75 × 10,000 ÷ 80 ⌉ | 94 |
| Theoretical capacity | workers × plantsPerSession ÷ flowTime | 6.27 k/hr |

---

## Shift Scheduling

All inflows (F1) and dispatches (F2) are **shift-gated** — they only operate during scheduled working hours. F3 (batch completions) is **not shift-gated** — in-flight batches complete on time regardless of shift status.

The simulation models a repeating 7-day week of 168 sim-hours. Within each week:
- The first `Days per week` days are working days.
- Within each working day, the first `Shifts per day × Hours per shift` hours are active shift time.
- All remaining hours (overnight, weekends) are off-shift.

When a shift ends, the F1 and F2 arrows dim in the SVG diagram and the **ON SHIFT / OFF SHIFT** indicator updates. Priority cohorts continue aging through off-shift periods.

---

## Time Scaling

The **Sim Speed** parameter controls how many sim-hours pass per 5 real-seconds.

| Sim Speed | Real time per 8-hr shift | Real time for 48-hr deadline |
|---|---|---|
| 1 | 40 seconds | 4 minutes |
| 8 (default) | 5 seconds | 30 seconds |
| 40 | 1 second | 6 seconds |

```
simDtHrs = wallDt × simSpeed / 5
```

Particle animation uses real wall-clock time (`wallDt`) directly, so particles always move at a visually smooth speed regardless of sim speed setting.

---

## UI Layout — Simulation (`simulation.html`)

### 1. Tab Bar

A pill-style navigation bar at the top lets you switch between the three pages. The active tab has a white text highlight.

### 2. Controls Bar

A dark panel with four groups of inputs:

**Material Flows**
- **Target (k plants/wk)** — Weekly throughput goal. Default 500k. Changing this recalculates F1 rate, worker count, and station capacity automatically.

**Pollination Station**
- **Workers Required** — Derived display. Shows workers calculated from target, session parameters, and schedule.
- **Plants / Session** — How many plants each worker pollinates per session. Default 50 (= 0.05 k plants).
- **Capacity** — Derived display showing `workers × plantsPerSession ÷ flowTime`. Updates live.

**Pollination Timing**
- **Setup / Transport (min)** — Non-productive time per session. Default 15 min.
- **Active Pollination (min)** — Time spent actively pollinating per session. Default 30 min.
- **Output Lag** — Derived display showing setup + pollination in minutes.

**Schedule & Simulation**
- **Days / week**, **Shifts / day**, **Hours / shift**, **Sim Speed** inputs.

**Buttons:** Start / Pause / Resume, Reset (preserves parameters).

### 3. Schedule Visualizer

A thin horizontal bar representing the full 7-day week. Green fill shows scheduled shift hours. A white cursor needle sweeps one full sim-week. Sim time counter on the right.

### 4. SVG Flow Diagram

Four tanks (left to right) connected by animated dashed arrows with flowing particles.

- **Propagation (S1)** — Small blue tank at constant 80% fill, labeled ∞.
- **Buffer Warehouse (S2)** — Dual-layer fill: blue (regular) + gold stacked on top (priority). Border turns red when > 90% full.
- **Pollination Zone (S3)** — Purple tank. Darker layer = active in-flight batches; lighter layer = queue.
- **Post-Pollination (S4)** — Green tank, cumulative.

Flow arrows dim during off-shift. **ON SHIFT / OFF SHIFT** label beneath F2 updates in real time.

**SVG KPI cards** stacked below each stock show live values for buffer fill, priority age, pollination zone capacity, and cumulative output.

### 5. Throughput Chart

A live canvas line chart at the bottom:
- **Y axis** — k plants per calendar week (0 to 1.6× target). Target shown as a green dashed line.
- **Grey line** — 24-hour rolling average of F3.
- **Blue line** — 7-day rolling average (168 sim-hours). Converges to ~500k/wk at defaults.

---

## Opportunity Analysis & ROI (`opportunity-analysis.html`)

A static (non-animated) side-by-side scenario comparison calculator. No simulation loop — all values update instantly as inputs change.

### Structure

**Baseline Parameters panel** (shared inputs across all scenarios):
- Volume & Geometry: weekly plant volume, plants per session, plants per cart
- Time Breakdown: total session time, movement/prep time, active pollination time
- Schedule & Cost: operating days, shifts/day, hrs/shift, technician hourly rate, priority plant %, cost per failed cross

**Scenario Comparison panel** (three columns: Baseline, Scenario A, Scenario B):

| Column | Description |
|---|---|
| **Baseline** | Current state — no automation |
| **Scenario A** (blue) | **Logistics Automation** — reduces movement/prep time. Inputs: automation %, capex, annual opex |
| **Scenario B** (green) | **Workstation Automation** — reduces active pollination time. Inputs: automation %, capex, annual opex |

**Comparison table sections:**
- **Session Time Breakdown** — Total, setup, and active pollination time per session
- **Labor** — Labor hrs/week, hrs saved, technicians required, weekly cost, annual cost, annual savings vs baseline
- **Capacity & ROI** — Weekly capacity at same headcount, capacity surplus vs target, payback period
- **Biological Risk (48-hr Deadline)** — Priority plants/week, system utilization, 48-hr processing capacity, estimated risk cost/week

Delta badges beside each scenario value show the change vs baseline (green = improvement, red = degradation).

---

## Layout (`layout.html`)

An interactive floorplan visualization of the pollination facility.

### Structure

- **Controls strip** — Mode buttons to switch between layout views; stat chips showing key derived values.
- **Floorplan** — SVG diagram of the physical floor plan showing workstation positions, travel paths, and flow routing.
- **Routing panel** — Shows dispatch rules (priority ordering logic), formula breakdowns, and key notes.
- **Ergonomics Comparison** — Side-by-side current state vs. future state workflow steps with KPI deltas (movement time, utilization).
- **Legend / Key Principles strip** — Color key and design principles.

---

## Scenarios to Explore

### 1. Verify Steady-State Target
Run at defaults for ~30 real seconds (several sim-weeks). The blue 7-day average line on the chart should converge to 500k/wk.

### 2. Triggering Priority Deadline Failures
Reduce **Plants / Session** to 5, which sharply cuts station throughput. Watch the Priority Age bar fill toward red. After 48 sim-hours, the oldest cohort expires — Deadline Failures turns red and the pulsing alert banner fires.

### 3. Session Time Impact
Increase **Setup / Transport** from 15 min to 60 min. Total flow time rises to 90 min, sessions/hr drops, and the required worker count roughly doubles. The output lag at each shift start also lengthens — visible as a deeper F3 dip in the chart.

### 4. Weekend Gap Effect
Set **Days/week to 2**. Priority cohorts now age through a 5-day weekend gap. The Priority Age card approaches 48h and failures begin within 1–2 sim-weeks.

### 5. Extended Shift Coverage
Set **Shifts/day to 3** (24-hour operation). Scheduled hours/week rises from 80 to 120, so the 500k target requires a lower instantaneous F1 rate (~4.17 k/hr) and fewer workers.

### 6. ROI Comparison
Open `opportunity-analysis.html`. Set **cost per failed cross** to a non-zero value and adjust Scenario A's automation % and capex. Watch the payback period and annual savings update instantly.

---

## Model Invariants and Key Formulas

```
simDtHrs            = wallDt × simSpeed / 5
hoursPerWeek        = daysPerWeek × shiftsPerDay × hoursPerShift
f1Rate              = target / hoursPerWeek                    (k/hr)
sessionsPerWeek     = target / plantsPerSessionK
numWorkers          = ⌈ totalFlowTimeHrs × sessionsPerWeek / hoursPerWeek ⌉
theoreticalCapacity = numWorkers × plantsPerSessionK / totalFlowTimeHrs   (k/hr)
maxActiveWIP        = numWorkers × plantsPerSessionK           (k plants)
outputLag           = totalFlowTimeHrs                         (completions trail dispatches by this amount)
weeklyOutput        = avgF3Rate × 168                          (k/wk, calendar week basis)
```

**Strict separation:** `updateSimulation(wallDt)` performs all math with no DOM access. `renderFrame()` performs all DOM writes with no math.

**Priority dispatch:** The cohort list is sorted by deadline ascending each tick. F2 drains the earliest-deadline cohort first until the tick's allocated transfer is exhausted.

**F3 is not shift-gated:** Batches dispatched just before shift end complete `totalFlowTimeHrs` later (during off-shift), partially offsetting the output lag at the next shift start.

---

## Architecture

All pages are single self-contained HTML files with no external dependencies, build tools, or frameworks.

**Code structure** (simulation.html):

| Section | Purpose |
|---|---|
| CSS | Dark theme (`#0f172a` base), inline only |
| SVG markup | Static diagram; clip-paths for stock fills; KPI card rects and text |
| `STOCKS_GEO` | Pixel geometry constants mirroring SVG rect positions |
| Constants | `BUFFER_CAP`, `PRIORITY_RATIO`, `DEADLINE_HRS`, `WARN_HRS`, etc. |
| `params` | All user-adjustable and derived parameters; survives resets |
| `freshState()` | Returns a new clean `sim` object; called on Reset |
| `dom` | All `getElementById` calls in one place |
| `isShiftActive()` | Pure function: is `sim.elapsed` inside a scheduled shift? |
| `updateSimulation(wallDt)` | All simulation math; zero DOM writes |
| `renderFrame()` | All DOM/SVG writes; zero simulation math |
| `renderChart()` | Canvas throughput chart; called from `renderFrame()` |
| `tick(timestamp)` | RAF loop; caps `wallDt` at 50ms; guarded by `visibilitychange` |
| `syncAll()` | Reads all control inputs, derives all params, updates displays |
| Controls section | Event listeners wired to `syncAll()` |

**Code structure** (opportunity-analysis.html):

| Section | Purpose |
|---|---|
| CSS | Same dark theme; comparison table grid layout |
| HTML | Baseline parameters panel + scenario comparison table |
| `recalc()` | Single JS function; reads all inputs, computes all derived values, writes all output cells |
| Event listeners | Wire all inputs to `recalc()`; runs once on load |

---

## Files

| File | Description |
|---|---|
| `simulation.html` | Dynamic Simulation — worker-session model, animated SVG, throughput chart |
| `opportunity-analysis.html` | Opportunity Analysis & ROI — static scenario comparison calculator |
| `layout.html` | Layout — interactive floorplan and routing logic visualization |
| `living-factory.html` | Continuous-flow model with labor utilization slider (Phase 2/3 reference) |
| `index.html` | Generic 2-stock/3-flow prototype (Phase 1 reference) |
| `plan.md` | Development roadmap with phase-by-phase completion notes |
| `CLAUDE.md` | Architecture guide for AI-assisted development sessions |
| `README.md` | This file |
