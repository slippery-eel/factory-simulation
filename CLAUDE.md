# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A browser-based system-dynamics simulation for the Ohalo "Living Factory" seed production case study. No build step, no dependencies — open the HTML files directly in a browser.

- `index.html` — Phase 1: generic 2-stock / 3-flow prototype (keep intact, do not extend)
- `living-factory.html` — Phase 2+3: the production simulation (primary working file)
- `plan.md` — project roadmap with all phases and their completion status

## Architecture: living-factory.html

The entire simulation lives in a single HTML file. Code is organized into clearly labeled sections (using `═══` banner comments):

1. **CSS** (lines ~1–530) — inline styles only; dark theme (`#0f172a` base)
2. **SVG diagram** (`id="diagram"`) — static markup for stocks, flow paths, labels; no JS-generated SVG structure except particles
3. **`STOCKS_GEO`** array — geometry constants (fillX, fillW, fillBottom, fillHeight, cx) that mirror each stock's SVG rect; used by `renderFrame()` to compute fill heights
4. **Constants** — `BUFFER_CAP`, `POLL_CAP`, `PRIORITY_RATIO`, `DEADLINE_HRS`, `WARN_HRS`, `HOURS_PER_SHIFT`, `TARGET_WEEKLY`, etc.
5. **`params`** object — user-adjustable parameters (rates, labor utilization, schedule). Survives resets. All rates stored internally as **k plants / sim-hr**.
6. **`freshState()`** — returns a new `sim` object. Called on reset. Contains stocks (`bufReg`, `bufPri`, `poll`, `postPoll`), flows, age clock, failure counter, particles array, spawn accumulators.
7. **`dom`** object — all `getElementById` calls in one place; reference this before adding any new DOM-touching code.
8. **`updateSimulation(wallDt)`** — simulation math only; no DOM writes. Uses `simDtHrs = wallDt * params.simSpeed / 5`.
9. **`renderFrame()`** — DOM writes only; reads `sim` and `params`, writes to `dom.*`. No simulation math.
10. **`tick(timestamp)`** — RAF loop; calls `updateSimulation` then `renderFrame`; caps `wallDt` at 50ms; guarded by `document.visibilitychange`.
11. **Controls section** — event listeners + `syncFlowsFromWeekly()` helper.

## Critical Invariants

**Time scaling:** `simSpeed` = sim-hours per 5 real-seconds. Default 8 means one 8-hour shift passes in 5 real seconds. All stock math uses `simDtHrs`; particle visual movement uses `wallDt` directly (so particle speed looks smooth regardless of simSpeed).

**Unit system:** 1 sim unit = 1,000 plants (k plants). Flow rates in **k plants / sim-hr** in `params` and all internal math. F1/F3 UI inputs accept **k plants / week**; `syncFlowsFromWeekly()` converts to k/hr before storing in `params`.

**Labor utilization (F2):** `f2Effective = stationCap * laborUtil`. This is a throughput rate cap on skilled labor time — NOT a yield loss. All plants are eventually processed; the constraint is that workers spend 33% of their time on logistics, not pollination. Do not model as plant scrap or efficiency multiplier on plant quantity.

**Priority routing:** F2 pulls from `bufPri` first, then `bufReg`. Priority plants have a 48-sim-hour biological deadline tracked via `prAge`. When `prAge >= DEADLINE_HRS`, all remaining `bufPri` plants count as failures and `prAge` resets.

**Spawn accumulators:** Particles are spawned when the accumulator (in k plants) exceeds 1. One particle represents 1,000 plants. Accumulators use `simDtHrs` so particle density reflects actual biological throughput, not wall time.

## Key Formulas

```
simDtHrs     = wallDt * params.simSpeed / 5
f2Effective  = params.stationCap * params.laborUtil
weeklyOutput = sim.flows.f3 × (daysPerWeek × shiftsPerDay × HOURS_PER_SHIFT)
targetRate   = TARGET_WEEKLY / hoursPerWeek   // in k/hr
```

Default steady-state: 5 days × 2 shifts × 8 hrs = 80 working hrs/week → 500k ÷ 80 = **6.25 k/hr** for F1/F3; stationCap = 6.25 ÷ 0.667 = **9.4 k/hr**.

## Adding New Features

- **New control input:** Add HTML, add `dom.xyzInput` ref, add event listener in the Controls section.
- **New data panel card:** Add HTML card, add `dom.dpXyz` ref, write to it in `renderFrame()` only.
- **New SVG element that needs JS updates:** Give it an `id`, add it to `dom`, update in `renderFrame()`.
- **Simulation logic change:** Only touch `updateSimulation()`. Never read `sim` in `renderFrame()` for math — just display.
