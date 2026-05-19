# Living Factory Simulation

A browser-based, real-time system-dynamics simulation of the Ohalo seed production pipeline. Built to make the scheduling trade-offs in a high-throughput plant pollination operation viscerally visible — no installation, no build step, just open the HTML file in a browser.

---

## Quick Start

```
git clone https://github.com/slippery-eel/factory-simulation
```

Open `living-factory.html` directly in any modern browser. Press **Start** and watch the system evolve.

`index.html` is a simpler generic prototype (Phase 1) kept for reference.

---

## Background: The Problem Being Modeled

Ohalo's "Living Factory" produces seeds at scale through a controlled pollination process. Plants move from propagation through a buffer warehouse, into a pollination zone, and then into post-pollination recovery. The critical constraints are:

1. **A 48-hour biological window** — certain plants are at peak flowering and must reach the pollination zone within 48 hours of entering the buffer, or the breeding cycle is lost.
2. **A labor utilization bottleneck** — pollination technicians spend roughly 33% of their shift time on logistics (cart retrieval, staging, transit) rather than active pollination. This caps throughput at ~67% of theoretical station capacity.
3. **Shift scheduling** — work only happens during scheduled shifts. Plants age around the clock regardless, so long off-shift gaps erode the available 48-hour window for priority plants.

The simulation lets you tune every parameter and watch the consequences play out in accelerated time.

---

## The System Model

```
[Propagation] --F1--> [Buffer Warehouse] --F2--> [Pollination Zone] --F3--> [Post-Pollination]
```

### Stocks (the four tanks)

| Stock | Description | Capacity |
|---|---|---|
| **S1 — Propagation** | Upstream source of plants ready for staging. Treated as effectively infinite in this model. | Unlimited |
| **S2 — Buffer Warehouse** | Primary holding area before pollination. Contains two sub-populations: regular (blue) and priority (gold). | 800 k plants |
| **S3 — Pollination Zone** | Work-in-progress during active pollination sessions. | 50 k plants |
| **S4 — Post-Pollination** | Cumulative count of plants that have completed pollination and entered seed development. | Unbounded (cumulative) |

### Flows

| Flow | Path | Description |
|---|---|---|
| **F1 — Inflow** | Propagation → Buffer | Plants arriving from the growth stage. Configurable as k plants/week. Only active during scheduled shifts. |
| **F2 — Transfer** | Buffer → Pollination Zone | The bottleneck. Capped by station capacity × labor utilization. Only active during scheduled shifts. Dispatches priority cohorts first (earliest deadline first), then regular plants. |
| **F3 — Completion** | Pollination Zone → Post-Pollination | Plants leaving the pollination zone after their session. Configurable as k plants/week. |

### Units

All quantities are in **k plants** (thousands of plants). Flow rates are in **k plants / sim-hour** internally, but displayed and entered as **k plants / week** in the UI. 1 sim unit = 1,000 real plants.

---

## Priority Plants and the 48-Hour Deadline

When plants enter the Buffer Warehouse, 20% are flagged as **priority** (flowering-stage plants with a hard biological deadline). The remaining 80% are regular plants with no strict time constraint.

**How priority tracking works:**

- Priority plants arrive in 1 k-plant cohort batches. Each cohort is stamped with a deadline: `arrival_time + 48 sim-hours`.
- The cohort queue is sorted by deadline ascending — the most urgently expiring cohort is always dispatched first when F2 has capacity.
- Cohorts age continuously, even during off-shift hours. Biological clocks do not pause for weekends.
- When any cohort's deadline passes before it is moved, those plants are counted as **deadline failures** — a lost breeding cycle.
- The **Priority Age** display shows the elapsed sim-hours since the oldest surviving cohort arrived (i.e., how close the most urgent cohort is to its 48-hour limit).
- A warning threshold is set at 40 sim-hours (83% of the window consumed). The priority age card turns amber at 40h and red beyond that.
- If failures occur, a red pulsing alert banner appears at the top of the page.

The gold band in the Buffer Warehouse tank represents priority plants stacked on top of the regular (blue) fill.

---

## The Labor Utilization Bottleneck

Pollination technicians spend approximately 15 of every 45 working minutes on non-pollination logistics: retrieving carts, transit between zones, staging, and setup. This means only ~67% of their time is spent on actual pollination work.

**In the model:**
```
F2 effective rate = Station Capacity (k/hr) × Labor Utilization (%)
```

This is a **throughput rate cap on skilled time**, not a yield loss. Every plant that enters the pollination zone is successfully pollinated — the constraint is how fast plants can be moved through. Raising labor utilization to 100% (by automating logistics) restores the lost 33%, enabling ~50% more plants per shift with the same headcount and station count.

The **Labor Utilization slider** is the primary "automation lever" in the simulation. Moving it from 67% to 100% directly increases F2's effective throughput ceiling.

---

## Shift Scheduling

All flows (F1 and F2) are **shift-gated** — they only operate during scheduled working hours. Plant aging continues around the clock regardless of shift status.

**How the schedule works:**

The simulation models a repeating 7-day week of 168 sim-hours. Within each week:
- The first `Days per week` days are working days.
- Within each working day, the first `Shifts per day × Hours per shift` hours are active shift time.
- All remaining hours (overnight, weekends) are off-shift.

Example with defaults (5 days, 2 shifts, 8 hrs):
- Working hours per day: 16 (hours 0–15 of each working day)
- Off-shift each working day: 8 hours (hours 16–23)
- Weekend: 48 hours (days 5 and 6 of each week)
- Total working hours per week: 80 / 168 = 47.6% duty cycle

When a shift ends, the F2 arrow and label dim in the SVG diagram, and the **ON SHIFT / OFF SHIFT** indicator below the F2 arrow updates. Any priority cohorts already in the buffer continue aging through the off-shift period.

---

## Time Scaling

The simulation uses accelerated sim-time. The **Sim Speed** parameter controls how many sim-hours pass per 5 real-seconds.

| Sim Speed | Real time for 1 shift (8 sim-hrs) | Real time for 48-hr deadline |
|---|---|---|
| 1 | 40 seconds | 4 minutes |
| 8 (default) | 5 seconds | 30 seconds |
| 40 | 1 second | 6 seconds |
| 100 | 0.4 seconds | 2.4 seconds |

The internal simulation time step is:
```
simDtHrs = wallDt × simSpeed / 5
```

Particle animation uses real wall-clock time (`wallDt`) directly, so particles always move at a visually smooth speed regardless of sim speed. Only the biological clocks, stock levels, and flow rates are governed by sim-time.

---

## UI Layout

The page is organized top-to-bottom into five sections:

### 1. Controls Bar

A dark panel with three groups of inputs separated by dividers:

**Material Flows**
- **F1 Inflow (k plants/wk)** — Rate at which plants arrive from propagation into the Buffer Warehouse. Default 500k/wk. A live conversion shows the equivalent k/hr rate based on the current schedule.
- **F3 Completion (k plants/wk)** — Maximum rate at which plants can exit the Pollination Zone. Default 500k/wk.

**Pollination Station**
- **Station Capacity (k plants/wk)** — Theoretical throughput if technicians spent 100% of their time on pollination. Default 752k/wk (= 9.4 k/hr × 80 hrs/wk). A live conversion shows the k/hr equivalent.
- **Labor Utilization slider** — The fraction of shift time spent on actual pollination (vs. logistics). Default 67%. Moving to 100% models full logistics automation. Displayed as a percentage with color coding: red below 50%, amber below 80%, green at 80%+.

**Schedule & Simulation**
- **Days / week** — Number of working days per 7-day cycle. Default 5. Range 1–7.
- **Shifts / day** — Number of shifts per working day. Default 2. Range 1–3.
- **Hours / shift** — Duration of each shift in sim-hours. Default 8. Range 1–24.
- **Sim Speed (sim-hrs / 5s)** — Simulation acceleration factor. Default 8. Range 1–100.

All weekly inputs (F1, F3, Station Capacity) automatically recalculate their k/hr conversions when the schedule changes.

**Buttons**
- **Start / Pause / Resume** — Toggle the simulation running state.
- **Reset** — Returns all stock levels and the elapsed clock to their initial values. User-set parameters (rates, schedule) are preserved across resets.

---

### 2. Schedule Visualizer Bar

A thin horizontal bar showing the entire 7-day week at a glance, positioned between the controls and the flow diagram.

- Each of the 7 day cells occupies an equal 1/7 of the bar width (matching the 1/7 time fraction exactly).
- **Green fill** within each day cell represents scheduled shift hours. The fill width is proportional to working hours / 24.
- **Day labels** (Mon–Sun) appear below each cell. Working days are shown in a lighter color; off days are muted.
- **Active day** is highlighted with a slightly lighter background.
- **White cursor needle** moves left-to-right across the bar based on `sim.elapsed mod 168 sim-hours`, completing one full sweep per sim-week and then resetting to the left.
- Changing Days/week, Shifts/day, or Hours/shift immediately updates the fill blocks (with a smooth 0.25s CSS transition).

This visualizer makes it immediately clear how much of the week is productive time vs. idle, and where the cursor is within the current week.

---

### 3. SVG Flow Diagram

A 1200×470 SVG showing the four stocks as filled tanks connected by animated flow paths.

**Tanks (left to right):**

- **Propagation (S1)** — Small blue tank, shown at a constant 80% fill representing the upstream supply. Labeled ∞ to indicate it is not the system's constraint.
- **Buffer Warehouse (S2)** — Larger tank with dual-layer fill. The **blue** portion (bottom) represents regular plants; the **gold** portion (stacked on top) represents priority plants. A border appears red when the buffer exceeds 90% capacity, amber when it drops below 10%. A separate progress bar beneath the tank shows the 48-hour priority deadline, colored green → amber → red as the oldest cohort ages toward its limit.
- **Pollination Zone (S3)** — Purple tank representing active work-in-progress. Border turns amber when nearly empty (< 10%), red when near capacity (> 90%).
- **Post-Pollination (S4)** — Green tank showing cumulative completed plants (fills toward a reference of 2,000 k plants).

**Flow paths (dashed arrows):**

- **F1 (blue)** — Propagation to Buffer. Dims to 35% opacity during off-shift.
- **F2 (purple)** — Buffer to Pollination Zone. Dims to 35% opacity during off-shift. Shows "ON SHIFT" (green) or "OFF SHIFT" (slate) below the arrow. Includes a readout of station utilization %, effective rate, and theoretical capacity.
- **F3 (teal)** — Pollination Zone to Post-Pollination.

**Animated particles:**

Particles travel along each flow path to give an intuitive sense of throughput rate:
- **Blue circles** on F1 — regular plant inflow.
- **Gold diamonds** on F2 — priority plants being dispatched (earliest deadline first).
- **Purple circles** on F2 — regular plants following priority dispatch.
- **Teal circles** on F3 — completed plants exiting the pollination zone.

Each particle represents 1,000 plants. Particle density is proportional to flow rate — more particles = faster flow. Particle speed slows slightly when the downstream stock is crowded (backpressure effect).

---

### 4. Data Panel

Twelve cards in a 4×3 grid providing numerical readouts of all key simulation variables. Card borders change color (amber = warning, red = alert, green = good) to draw attention to out-of-spec conditions.

**Row 1 — Stock Levels**

| Card | Shows | Alert condition |
|---|---|---|
| Buffer Warehouse | Total plants in buffer (k) + % capacity | Red border > 90% full; amber < 10% |
| Priority in Buffer | Priority plants in buffer (k) + % of buffer total | Amber border whenever > 0 |
| Pollination Zone | Plants currently in pollination (k) + % capacity | Red > 90%; amber < 10% |
| Post-Pollination | Cumulative completed plants (k) | — |

**Row 2 — Flow Rates**

| Card | Shows |
|---|---|
| F1 Inflow Actual | Current F1 rate (k/hr) + target rate for the schedule |
| F2 Effective Rate | Current F2 rate (k/hr) + theoretical station capacity (k/hr) |
| Station Utilization | Labor utilization % — amber < 80%, red < 50% |
| F3 Completion Actual | Current F3 rate (k/hr) |

**Row 3 — Biological Clock & Output**

| Card | Shows | Alert condition |
|---|---|---|
| Sim Time | Elapsed sim-hours + current sim speed setting | — |
| Priority Age | Age of the oldest surviving priority cohort in sim-hours (out of 48) | Amber > 30h; red > 40h |
| Deadline Failures | Cumulative k plants lost to deadline expiry | Red if > 0; green if 0 |
| Weekly Throughput | F3 rate × working hours/week (k plants/wk) | Green ≥ 500k; red < 500k |

---

### 5. Throughput Chart

A live canvas-based line chart at the bottom of the page showing the evolution of pollination throughput over time.

**Y axis** — k plants per calendar week (0 to 800k). The target of 500k/wk is shown as a **green dashed line**.

**X axis** — Elapsed sim-hours, scrolling to show the last 336 sim-hours (2 sim-weeks). Tick marks every 24 sim-hours.

**Two lines:**

- **Grey — 24-hour rolling average.** Average F3 rate over the past 24 sim-hours, converted to a weekly equivalent. This line captures the recent daily rhythm: it rises during productive shifts and dips during off-shift gaps, giving a feel for day-to-day variability.

- **Blue — 7-day rolling average.** Average F3 rate over the past 168 sim-hours (1 full sim-week), converted to a weekly equivalent. This line smooths out the shift oscillation and converges to the true annualized throughput rate. At steady state with default settings, it settles near 500k/wk.

**Display rules:**
- The grey line only appears once 24 sim-hours of data exist (no distorted early-average artifact).
- The blue line only appears once 168 sim-hours of data exist.
- The chart stores up to 504 sim-hours of history (336 display + 168 buffer) so rolling averages are always fully computed for all visible points once the chart is scrolling — no edge distortion at the left boundary.
- The chart is DPR-aware and renders crisply on retina displays.

---

## Default Steady-State

With all defaults, the simulation reaches equilibrium at the 500k plants/week target:

| Parameter | Value | Derived from |
|---|---|---|
| Working hours/week | 80 hrs | 5 days × 2 shifts × 8 hrs |
| F1 target rate | 6.25 k/hr | 500k ÷ 80 hrs |
| Station capacity | 9.4 k/hr | 752k ÷ 80 hrs |
| Effective F2 rate | 6.27 k/hr | 9.4 × 0.667 |
| F3 target rate | 6.25 k/hr | 500k ÷ 80 hrs |
| Priority arrivals | 1.25 k/hr | 6.25 × 20% |
| Buffer utilization | ~31% | 250k / 800k |

F2 effective rate (6.27 k/hr) slightly exceeds F1 rate (6.25 k/hr), so the buffer stays stable rather than accumulating. Priority plants cycle through faster than their 48-hour deadline.

---

## Scenarios to Explore

### 1. The Labor Utilization Intervention
Run at defaults until stable (~30 real seconds). Drag the **Labor Utilization slider** from 67% to 100%. Watch F2 rate jump from ~6.3 to ~9.4 k/hr, the Pollination Zone fill up briefly, and Weekly Throughput overshoot 500k before settling. This models the effect of automating cart logistics — same staff, same stations, ~50% more throughput.

### 2. Triggering Priority Deadline Failures
Reduce **Station Capacity** from 752k to ~300k/wk. At the reduced F2 rate, priority cohorts cannot be dispatched fast enough. Watch the priority age bar fill toward red. After 48 sim-hours the oldest cohort expires, Deadline Failures turns red, and the alert banner fires. The 7-day average line on the chart begins trending below 500k.

### 3. Buffer Overflow Under Surge
Increase **F1 Inflow** to 2000k/wk while leaving F2 at defaults. The buffer fills rapidly (red border). F1 automatically throttles once the buffer hits capacity, but the high buffer level means priority plants spend longer waiting, stressing the 48-hour deadline.

### 4. Weekend Gap Effect
Set **Days/week to 2** (a skeleton schedule). Watch the OFF SHIFT indicator dominate the schedule bar. Priority cohorts age aggressively through the long weekend gap. At default F2 rates, failures begin to accumulate within a few sim-weeks. Increasing Station Capacity to clear priority plants before shift end mitigates the failure rate.

### 5. Extended Shift Coverage
Set **Shifts/day to 3** and **Hours/shift to 8** (24-hour operation). The schedule bar shows full green fill across all working days. F1/F3/StationCap k/hr conversions automatically drop (because more hours are available), meaning the same weekly targets require less aggressive instantaneous rates. Priority deadline failures become very rare.

---

## Architecture Notes

The simulation is a single self-contained HTML file with no external dependencies, build tools, or frameworks.

**Code structure** (`living-factory.html`, ~1,400 lines):

| Section | Purpose |
|---|---|
| CSS (~300 lines) | Dark theme styles, all inline |
| SVG markup | Static diagram structure; no JS-generated SVG except particles |
| `STOCKS_GEO` | Pixel geometry constants mirroring SVG rect positions |
| Constants | `BUFFER_CAP`, `POLL_CAP`, `PRIORITY_RATIO`, `DEADLINE_HRS`, etc. |
| `params` | User-adjustable parameters; survives resets |
| `freshState()` | Returns a new `sim` object; called on reset |
| `dom` | All `getElementById` calls in one place |
| `isShiftActive()` | Pure function: is `sim.elapsed` inside a scheduled shift? |
| `updateSimulation(wallDt)` | All simulation math; no DOM writes |
| `renderFrame()` | All DOM writes; reads `sim` and `params`; no simulation math |
| `renderChart()` | Canvas throughput chart; called from `renderFrame()` |
| `tick(timestamp)` | RAF loop; caps `wallDt` at 50ms |
| Controls section | Event listeners + `syncFlowsFromWeekly()` |

**Key invariants:**
- Simulation math and DOM rendering are strictly separated (`updateSimulation` vs `renderFrame`).
- All rates are stored internally as k plants / sim-hr. Weekly UI inputs are converted via `syncFlowsFromWeekly()`.
- `simDtHrs = wallDt × simSpeed / 5` governs all stock math. Particle visual position uses `wallDt` directly.
- Priority cohort dispatch is O(n log n) per tick (sort + linear scan); chart rolling averages are O(n) sliding window.

---

## Files

| File | Description |
|---|---|
| `living-factory.html` | Main simulation (Phases 2–5) — open this |
| `index.html` | Generic 2-stock/3-flow prototype (Phase 1) |
| `plan.md` | Full development roadmap with phase-by-phase completion notes |
| `CLAUDE.md` | Architecture guide for AI-assisted development sessions |
| `Orchestrating-the-Living-Factory.txt` | Source document describing the real-world operation |
