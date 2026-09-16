# Grid Balance

A phone-first operations app for allocating substation capacity across datacentre halls on the Irish grid.

When a cluster substation cannot carry every hall at once, something has to run on generator. Grid Balance decides *which* halls keep grid supply, using a nested knapsack over hall priority, power draw and time of day, and answers the operator's real question: **if a datacentre needs another 20 kW at 09:00, can the substation take it, and what gets displaced?**

Built as a design prototype. The allocation logic is real; the grid data is a modelled snapshot (see [Data](#data)).

---

## The model

The allocation is the classic 0/1 knapsack, mapped onto the grid problem:

| Knapsack | Grid Balance |
| --- | --- |
| Bag capacity | Substation firm capacity, in kW |
| Item weight | A hall's power draw, in kW |
| Item value | The hall's priority, scaled by the hour |
| Maximise value | Keep the most important halls on grid |

### Capacity is nested

Two limits apply at once, matching the physical topology:

- **Cluster substation** — one firm capacity for the whole site (default **65 kW**).
- **Per datacentre** — each datacentre's own limit (default **25 kW**).

Cluster headroom is allocated freely, so a datacentre holding high-priority halls can take more than an equal share of the cluster, as long as it stays inside its own limit.

### Value depends on the clock

Base priority scores:

| Priority | Value |
| --- | --- |
| HIGH | 12 |
| MEDIUM | 5 |
| LOW | 2 |

Each hall carries a workload profile:

- **Business hours** — value is multiplied by the current time window.
- **Batch** — flat ×0.6 at every hour, because the work is shiftable.

Default time windows (editable in Settings):

| Window | Hours | Multiplier |
| --- | --- | --- |
| Night trough | 00:00–06:00 | ×0.6 |
| Morning ramp | 06:00–10:00 | ×2.0 |
| Daytime | 10:00–17:00 | ×1.0 |
| Evening peak | 17:00–20:00 | ×1.8 |
| Wind-down | 20:00–24:00 | ×0.8 |

So a HIGH business-hours hall is worth 24 during the morning ramp and 12 at midday, while a LOW batch hall sits at 1.2 all day. In the hard windows the business halls win substation space; overnight the batch halls can take it back.

A hall already on grid carries a **+0.15 incumbency bonus** — just enough to break a tie between equal halls, so the solver does not churn switches for no gain.

### What happens to the rest

Halls the substation cannot carry fall through in order:

1. **Generator** — up to a per-site generator capacity (default **15 kW**). Halls are whole, so the solver packs them to leave as little load unpowered as possible.
2. **Unserved** — anything the generator cannot hold either is flagged in red, and the app computes the smallest single capacity increase (cluster, datacentre or generator) that would clear it.

The priorities are strict: first leave the fewest kW unserved, then keep the most priority value on grid, and only then prefer higher-value halls on the generator.

### How it is solved

```
for each datacentre:
    every hall → grid, generator or unserved
    DP over (grid load, generator load)  → best assignment at every exact grid load 0…dcLimit
then:
    knapsack over the datacentres        → distribute the cluster ceiling across those curves
```

Both passes are exact dynamic programming, not a greedy heuristic — checked against exhaustive search. The whole allocation recomputes on every render — changing the hour, a priority, a draw or a capacity re-solves immediately. Nothing switches until the operator presses **Apply**.

---

## Screens

**Home** — the five real-time series from the EirGrid public dashboard: system demand, wind generation, CO₂ intensity, solar generation and interconnection, plus an estimated fuel mix. Cards show the last four hours; tapping one opens the full 24 hours at 15-minute resolution with min, average and max. The fuel mix uses the latest wind, solar and import readings; gas, coal & peat and other renewables are modelled from demand.

**Grid** — the single-line diagram and the allocation in one screen, read top to bottom in the direction the power flows:

1. **Grid supply in** — the two incoming grid connections, each with a switch. Each carries half the substation's capacity, so opening one halves the ceiling.
2. **Cluster substation** — steps the supply down and shares it out below. Its bar is how much of the firm capacity the halls are currently drawing.
3. **Advice** — the solver's verdict for the current hour, the halls that cannot be served, the smallest capacity fix, and an **Apply** button. Nothing switches until it is pressed.
4. **Datacentres** — hanging off the bus, each with its live kW, a switch for the whole site, and a chip per hall. Tapping a chip moves that hall between grid and generator; its slider icon opens the hall detail. A line under each site shows what the solver would do differently.
5. **Can I add more load?** — pick a datacentre and a kW figure; the load enters the bag as a HIGH-priority item and the app reports whether it can be served on grid, only on generator, or not at all, and which halls it displaces. It is a what-if: the advice and **Apply** above never include it.

**Best hours** re-solves the next 24 hours and ranks them by priority value forfeited to the capacity squeeze. Tapping an hour plans for that hour; **Now** returns the advice to the clock.

**Settings** — theme, capacity limits, time-window multipliers, datacentre and hall configuration, data source, and install.

The hall detail panel holds the 24-hour draw curve, priority, power draw, the hall's current grid value and a manual grid/generator switch.

---

## Themes

Two themes, both built on EirGrid's teal:

- **Nocturne** — dark control-room, accent `#2ec4a8`.
- **Organic** — light and warm, accent `#00806d` for contrast on a pale ground.

The brand teal `#009982` anchors both. Toggle from the header or Settings; the mobile status bar colour follows.

---

## Data

The app requests the EirGrid Smart Grid Dashboard service on load and every 15 minutes. That service does not send CORS headers, so a browser request from another origin is refused — when that happens the badge reads **SNAPSHOT** and the charts fall back to a bundled model with the real all-island shape:

- Demand 3.6–5.5 GW on the true daily curve
- Wind up to 4.3 GW with realistic drift
- Solar following the solar day
- Carbon intensity 140–470 g/kWh, derived from the renewable share

The snapshot advances one 15-minute settlement period at a time, so the charts move as the clock does. To get genuinely live data, put a small server-side proxy in front of the EirGrid endpoint and point `tryLive()` at it — the parsing already matches the service's response shape.

Everything else in the app — the solver, the capacities, the switching — is live logic. Datacentres are named Datacentre 1, 2 and 3: this is a demo, not a model of anyone's real site.

---

## Running it

Any static web server works. No build step, no package install.

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

Opening the file directly with `file://` mostly works but disables installation.

## Installing it

Grid Balance is a PWA. Served over HTTPS:

- **Android / desktop Chrome** — the browser offers Install, and an Install button appears in Settings.
- **iPhone / iPad** — Share → Add to Home Screen.

It then opens full screen with no browser chrome, respecting the device's safe areas.

## Layout

Responsive, one breakpoint at 900 px.

- **Below** — single column, bottom tab bar, panels as bottom sheets.
- **Above** — centred ~960 px column, tabs in the header, charts three across, panels as centred dialogs. The datacentres stay stacked so the vertical bus of the diagram still reads.

---

## Files

```
index.html                     redirect to the app
EirGrid Load Balance.dc.html   the whole app — template, logic, styles
support.js                     component runtime
manifest.json                  PWA manifest
sw.js                          minimal service worker (enables install)
icon-180/192/512.png           app icons
_ds/                           Nocturne design system tokens
```

The app is a single file. Its logic class holds the solver (`siteKnapsack`, `solveFill`, `minFix`), the data model (`sample`, `snapshot`, `advance`, `tryLive`) and the view state; the template above it is plain inline-styled markup.

## Configuring

All of this is editable in the running app, under Settings:

- Cluster, per-datacentre and generator capacities
- Time-window multipliers
- Datacentres and halls — add, remove, rename

Per-hall priority, power draw and current source are set from the hall detail panel.

---

## Licence

MIT — see [LICENSE](LICENSE).

## Status

Design prototype. Not connected to a live EirGrid feed, not authenticated, and state is held in memory only — a reload restores the defaults.
