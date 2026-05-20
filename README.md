# Living Factory Simulation

A browser-based, real-time system-dynamics simulation of the Ohalo seed production pipeline. Built to make the scheduling trade-offs in a high-throughput plant pollination operation viscerally visible — no installation, no build step, just open an HTML file in a browser.

---

## Quick Start

```
git clone https://github.com/slippery-eel/factory-simulation
```

Open either file directly in any modern browser and press **Start**.

| File | What it is |
|---|---|
| `roi.html` | **Opportunity Analysis & ROI tab** — worker-session model with SVG KPI cards. Primary working file. |
| `living-factory.html` | Clean simulation — continuous-flow model with labor utilization slider. |
| `index.html` | Generic 2-stock/3-flow prototype (Phase 1), kept for reference. |

The two tabs link to each other via a tab bar at the top of each page.

---

## Background: The Problem Being Modeled

Ohalo's "Living Factory" produces seeds at scale through a controlled pollination process. Plants move from propagation through a buffer warehouse, into a pollination zone, and then into post-pollination recovery. The critical constraints are:

1. **A 48-hour biological window** — certain plants are at peak flowering and must be pollinated within 48 hours of entering the buffer, or the breeding cycle is lost.
2. **Session-based labor** — technicians work in discrete pollination sessions: collect a cart of plants, spend time on setup and transport (15 min default), actively pollinate (30 min default), then return. This session structure means each worker processes a fixed batch size per cycle rather than a continuous stream.
3. **Shift scheduling** — work only happens during scheduled shifts. Plants age around the clock regardless, so long off-shift gaps erode the 48-hour biological window for priority plants.
4. **Output lag** — at the start of each shift, workers begin sessions immediately, but the first completions don't appear until one full flow-time cycle later (45 min by default). This lag is transient: sessions dispatched near shift end still complete after the shift closes, so it does not reduce weekly capacity. The simulation reveals whether the configured workers can actually hit the target rather than pre-compensating the model to guarantee it.

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

All quantities are in **k plants** (thousands of plants). Flow rates are displayed as **k plants / sim-hour** in the SVG labels and data panel. 1 sim unit = 1,000 real plants.

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

## Worker-Session Model (roi.html)

The ROI tab uses a **discrete-session model** for the pollination station rather than a continuous throughput rate.

### How it works

Each pollination session has two phases:
- **Setup / Transport** (default 15 min): retrieving carts, transit, staging
- **Active Pollination** (default 30 min): the actual pollination work

**Total flow time** = setup + pollination = 45 min (0.75 sim-hr) at defaults.

Workers are modeled as slots. At each simulation tick during a shift:
1. Completed batches are drained to Post-Pollination (F3).
2. Free worker slots are filled from the queue — each slot takes `plantsPerSession` k-plants and schedules completion at `elapsed + totalFlowTime`.

### Worker count derivation

The number of workers required is derived automatically from the target, session parameters, and schedule:

```
sessionsPerWeek = target / plantsPerSession
numWorkers      = ⌈ totalFlowTime × sessionsPerWeek / hoursPerWeek ⌉
```

Workers are derived from scheduled hours per week without subtracting any startup penalty. The output lag at shift start (one `totalFlowTime` cycle of zero F3) is a transient effect visible in the throughput chart, not lost weekly capacity — sessions dispatched near shift end still complete, offsetting it.

### Key derived quantities

| Quantity | Formula | Default value |
|---|---|---|
| Total flow time (output lag) | setup + pollination | 0.75 hr |
| Scheduled hrs/week | 5d × 2s × 8h | 80 hr |
| F1 rate | target ÷ hoursPerWeek | 6.25 k/hr |
| Sessions/week | target ÷ plantsPerSession | 10,000 |
| Workers required | ⌈ 0.75 × 10,000 ÷ 80 ⌉ | 94 |
| Theoretical capacity | workers × plantsPerSession ÷ flowTime | 6.27 k/hr |
| Max active WIP | workers × plantsPerSession | 4.70 k plants |
| Sessions/hr per worker | 1 ÷ flowTime | 1.333 |

Theoretical capacity (6.27 k/hr) slightly exceeds F1 rate (6.25 k/hr) due to the ceiling in the worker count formula, ensuring the station is never the bottleneck.

---

## Shift Scheduling

All inflows (F1) and dispatches (F2) are **shift-gated** — they only operate during scheduled working hours. F3 (batch completions) is **not shift-gated** — in-flight batches complete on time regardless of shift status.

**How the schedule works:**

The simulation models a repeating 7-day week of 168 sim-hours. Within each week:
- The first `Days per week` days are working days.
- Within each working day, the first `Shifts per day × Hours per shift` hours are active shift time.
- All remaining hours (overnight, weekends) are off-shift.

Example with defaults (5 days, 2 shifts, 8 hrs):
- Working hours per day: 16 (hours 0–15 of each working day)
- Off-shift each working day: 8 hours (hours 16–23)
- Weekend: 48 hours (days 5–6)
- Total working hours per week: 80 hrs (47.6% duty cycle)

When a shift ends, the F1 and F2 arrows dim in the SVG diagram and the **ON SHIFT / OFF SHIFT** indicator updates. Priority cohorts continue aging through off-shift periods.

---

## Time Scaling

The **Sim Speed** parameter controls how many sim-hours pass per 5 real-seconds.

| Sim Speed | Real time per 8-hr shift | Real time for 48-hr deadline |
|---|---|---|
| 1 | 40 seconds | 4 minutes |
| 8 (default) | 5 seconds | 30 seconds |
| 40 | 1 second | 6 seconds |
| 100 | 0.4 seconds | 2.4 seconds |

```
simDtHrs = wallDt × simSpeed / 5
```

Particle animation uses real wall-clock time (`wallDt`) directly, so particles always move at a visually smooth speed regardless of sim speed setting.

---

## UI Layout — Opportunity Analysis & ROI tab (`roi.html`)

The page is organized top-to-bottom into five sections.

---

### 1. Tab Bar

A pill-style navigation bar at the top lets you switch between the two simulations. The active tab has a white text highlight.

---

### 2. Controls Bar

A dark panel with four groups of inputs separated by dividers.

**Material Flows**
- **Target (k plants/wk)** — Weekly throughput goal. Default 500k. Changing this recalculates F1 rate, worker count, and station capacity automatically. A live conversion beneath the input shows the k/hr rate (target ÷ scheduled hours/week).

**Pollination Station**
- **Workers Required** — Derived display (not an input). Shows the number of workers calculated from the target, session parameters, and schedule. Also shows total sessions/week beneath.
- **Plants / Session** — How many plants each worker pollinates per session. Default 50 (= 0.05 k plants). Changing this alters session count and worker requirements.
- **Capacity** — Derived display showing `theoreticalPlantsPerHr` = workers × plantsPerSession ÷ flowTime. Updates live.

**Pollination Timing**
- **Setup / Transport (min)** — Non-productive time per session (cart retrieval, transit, staging). Default 15 min. Increasing this raises total flow time, reduces sessions/hr, and requires more workers.
- **Active Pollination (min)** — Time spent actively pollinating per session. Default 30 min.
- **Output Lag** — Derived display showing setup + pollination in minutes. Default 45 min. This is the delay between when a worker dispatches a session and when F3 records the completion — visible as a brief dip in F3 at each shift start.

**Schedule & Simulation**
- **Days / week** — Working days per 7-day cycle. Default 5. Range 1–7.
- **Shifts / day** — Shifts per working day. Default 2. Range 1–3.
- **Hours / shift** — Duration of each shift. Default 8. Range 1–24.
- **Sim Speed (sim-hrs / 5s)** — Acceleration factor. Default 8.

Changing any schedule parameter immediately recalculates scheduled hours, F1 rate, and worker count.

**Buttons**
- **Start / Pause / Resume** — Toggle the simulation running state.
- **Reset** — Returns all stock levels, elapsed clock, and particles to initial values. Parameters are preserved.

---

### 3. Schedule Visualizer

A thin horizontal bar representing the full 7-day week, positioned below the controls.

- Seven equal cells, one per calendar day, with day labels (Mon–Sun).
- **Green fill** within each cell shows scheduled shift hours proportional to working hours / 24.
- **Active day** is highlighted with a slightly lighter background.
- **White cursor needle** moves continuously left-to-right across the bar, completing one sweep per sim-week (168 sim-hours).
- Off-day labels are muted. Working-day labels are light grey.
- **Sim time counter** (right side) shows elapsed sim-hours.

---

### 4. SVG Flow Diagram

A wide SVG showing the four stocks as filled tanks connected by animated dashed arrows.

**Tanks (left to right):**

- **Propagation (S1)** — Small blue tank, shown at a constant 80% fill. Labeled ∞ to indicate the upstream supply is not the constraint. A card below the tank shows the ∞ k plants value.
- **Buffer Warehouse (S2)** — Larger tank with dual-layer fill. The **blue** portion (from bottom) represents regular plants; the **gold** portion (stacked on top) represents priority plants with active deadlines. Border turns red when buffer exceeds 90% capacity, amber below 10%.
- **Pollination Zone (S3)** — Purple tank. The darker purple layer (bottom) shows active in-flight batches; the slightly lighter layer above shows plants queued for a free worker slot. Border turns amber when nearly empty, red when near capacity.
- **Post-Pollination (S4)** — Green tank showing cumulative completed plants.

**Flow arrows:**
- **F1 (blue dashed)** — Propagation → Buffer. Dims to 35% opacity during off-shift.
- **F2 (purple dashed)** — Buffer → Pollination Zone. Dims during off-shift. Shows priority plant sub-rate (▲ figure) alongside the total rate.
- **F3 (teal dashed)** — Pollination Zone → Post-Pollination. Always at full opacity (completions are not shift-gated).
- **ON SHIFT / OFF SHIFT** label beneath F2 updates in real time.

**Animated particles:**
- **Blue circles** on F1 — regular plant inflow.
- **Gold diamonds** on F2 — priority plants being dispatched.
- **Purple circles** on F2 — regular plants following priority dispatch.
- **Teal circles** on F3 — completed plants exiting pollination.

Each particle represents 1,000 plants. Particle density reflects flow rate; particle speed slows slightly when the downstream stock is crowded.

**SVG KPI cards** — stacked below each stock, aligned to the stock's fill width:

*Under Buffer Warehouse:*
| Card | Shows |
|---|---|
| Buffer Warehouse | Total plants (k) · % full |
| Priority in Buffer | Priority plants (k) · % of buffer total. Value tints gold when priority plants are present. |
| Priority Age | Age of oldest surviving priority cohort as `X.X / 48h`. Color-coded green → amber → red. A compact progress bar beneath shows the fraction of the 48-hr window consumed. |

*Under Pollination Zone:*
| Card | Shows |
|---|---|
| Pollination Zone | Total plants in zone (k) · % of capacity |
| Queue Waiting | Plants waiting for a free worker slot (k) |
| Max Active WIP | Maximum simultaneous in-flight plants = workers × plantsPerSession (k). Updates when parameters change. |

*Under Post-Pollination:*
| Card | Shows |
|---|---|
| Post-Pollination | Cumulative completed plants (k) |

---

### 5. Data Panel

A grid of cards providing numerical readouts of flow rates and operational metrics. Cards with alert conditions show colored borders (amber = warning, red = alert, green = good).

| Card | Shows | Alert condition |
|---|---|---|
| F1 Inflow Actual | Current F1 rate (k/hr) + target rate | — |
| F2 Effective Rate | Current F2 rate (k/hr) + theoretical station capacity (k/hr) | — |
| F3 Completion Actual | Current F3 rate (k/hr) | — |
| Sim Time | Elapsed sim-hours | — |
| Deadline Failures | Cumulative k plants lost to deadline expiry | Red if > 0; green if 0 |
| Weekly Throughput | 7-day rolling average of F3 rate × 168 hrs (k/wk) | Green ≥ target; red < target |
| Queue Waiting | Plants waiting for a worker slot (k) | — |
| Active WIP | Plants currently in in-flight sessions (k) + active batch count | — |
| Sessions / Hour | Session throughput per worker (= 1 ÷ flowTime) | — |
| Free Worker Slots | Idle workers = numWorkers − activeBatches. | Red when 0 (fully saturated); amber < 10% free |

---

### 6. Throughput Chart

A live canvas line chart at the bottom showing the evolution of pollination output over time.

- **Y axis** — k plants per calendar week (0 to 1.6× target). Target shown as a green dashed horizontal line.
- **X axis** — Elapsed sim-hours, scrolling to show the last 336 sim-hours (2 sim-weeks). Ticks every 24 sim-hours.
- **Grey line** — 24-hour rolling average of F3 converted to weekly equivalent. Captures daily rhythm (rises and dips with shifts).
- **Blue line** — 7-day rolling average (168 sim-hours). Converges to annualized throughput. At steady state with defaults, settles near 500k/wk.

Both lines only appear once sufficient history exists (24h and 168h respectively) to avoid distorted early-average artifacts.

---

## UI Layout — Clean Simulation (`living-factory.html`)

The clean simulation tab uses a **continuous-flow model** with a labor utilization slider rather than the session-based worker model. The core stocks and flows are the same, but the F2 bottleneck is modeled differently:

```
F2 effective rate = Station Capacity × Labor Utilization
```

**Labor Utilization** represents the fraction of shift time spent on actual pollination vs. logistics. At 67% (default), workers spend 33% of their time on cart retrieval, transit, and staging — the same constraint as the session model, expressed as a rate multiplier rather than session timing.

The labor utilization slider is the primary "automation lever": dragging it from 67% → 100% models the effect of automating logistics (same staff, same stations, ~50% more throughput).

The controls, schedule visualizer, SVG diagram, data panel, and throughput chart follow the same structure as the ROI tab, with these differences:
- Controls include **F1 Inflow**, **Station Capacity**, and **Labor Utilization** inputs (no session timing inputs).
- All derived quantities (k/hr conversions) use raw `hoursPerWeek`.
- The data panel includes a **Station Utilization** card showing current labor utilization %.

---

## Scenarios to Explore

### 1. Verify Steady-State Target
Run at defaults for ~30 real seconds (several sim-weeks). The blue 7-day average line on the chart should converge to 500k/wk. The Weekly Throughput card turns green.

### 2. Triggering Priority Deadline Failures
Reduce **Plants / Session** to a very small value (e.g., 5), which sharply cuts station throughput. Watch the Priority Age bar fill toward red. After 48 sim-hours, the oldest cohort expires — Deadline Failures turns red and the pulsing alert banner fires. Restore Plants/Session to recover.

### 3. Session Time Impact
Increase **Setup / Transport** from 15 min to 60 min. Total flow time (output lag) rises to 90 min, sessions/hr drops from 1.33 to 0.67, and the required worker count roughly doubles. The output lag at each shift start also lengthens — visible in the throughput chart as a deeper F3 dip after each shift change. Station capacity and F1 rate recalculate automatically.

### 4. Weekend Gap Effect
Set **Days/week to 2**. Priority cohorts now age through a 5-day weekend gap. The Priority Age card rapidly approaches 48h and failures begin within 1–2 sim-weeks. Increasing the target rate (which increases worker count and station capacity) can clear the buffer faster before the gap.

### 5. Extended Shift Coverage
Set **Shifts/day to 3** (24-hour operation). The schedule bar shows full green fill across all working days. Scheduled hours/week rises from 80 to 120, so the same 500k target requires a lower instantaneous F1 rate (~4.17 k/hr) and fewer workers. Priority deadline pressure drops significantly because cohorts are processed within the same day.

### 6. Buffer Overflow Under Surge
Increase **Target** from 500k to 2000k/wk while leaving session parameters at defaults. Worker count scales to ~375. F1 rate rises to 25.0 k/hr. The buffer fills faster and priority plants cycle through more rapidly — watch the Pollination Zone saturate and Queue Waiting rise.

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

**Strict separation:** `updateSimulation(wallDt)` performs all math with no DOM access. `renderFrame()` performs all DOM writes with no math. This prevents rendering artifacts from frame-rate variation.

**Priority dispatch:** The cohort list is sorted by deadline ascending each tick. F2 drains the earliest-deadline cohort first, then the next, until the tick's allocated transfer is exhausted.

**F3 is not shift-gated:** Batches dispatched just before shift end complete `totalFlowTimeHrs` later (during off-shift). This means each shift's last dispatch wave completes after the shift, partially offsetting the output lag at the start of the next shift. Weekly capacity is therefore not reduced by the lag — it is revealed as a transient dip in the chart.

---

## Architecture

Both simulations are single self-contained HTML files with no external dependencies, build tools, or frameworks.

**Code structure** (roi.html, ~1,500 lines):

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

---

## Files

| File | Description |
|---|---|
| `roi.html` | Opportunity Analysis & ROI tab — worker-session model, SVG KPI cards |
| `living-factory.html` | Clean Simulation tab — continuous-flow model with labor utilization slider |
| `index.html` | Generic 2-stock/3-flow prototype (Phase 1), for reference |
| `plan.md` | Development roadmap with phase-by-phase completion notes |
| `CLAUDE.md` | Architecture guide for AI-assisted development sessions |
| `README.md` | This file |
