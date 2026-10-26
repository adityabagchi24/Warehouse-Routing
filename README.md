# Warehouse Traffic Hotspots

**A data analytics study of congestion-aware routing for warehouse robots across four real warehouse layouts.**

![Python](https://img.shields.io/badge/Python-3.14-3776AB?logo=python&logoColor=white)
![Jupyter](https://img.shields.io/badge/Jupyter-Notebook-F37626?logo=jupyter&logoColor=white)
![NetworkX](https://img.shields.io/badge/NetworkX-graphs-2C7FB8)

---

## Overview

In a warehouse, every robot wants the shortest route to its goal. If every robot takes the fastest aisle, that aisle jams — the same failure mode you see on Google Maps when everyone picks the same road. Congestion-aware routing deliberately makes busy segments *slightly* more expensive, so later robots spread out.

This project asks three questions:

1. **Where** does traffic concentrate in a warehouse layout?
2. Does congestion-aware routing actually reduce hotspots?
3. Does the answer hold across **different layouts**, **different robot start/goal sets**, and **alpha values** — and what does it cost?

The analysis is fully reproducible in [`warehouse_congestion.ipynb`](warehouse_congestion.ipynb): 4 warehouse maps × 5 scenario files × 200 robots, routed twice (baseline vs. congestion-aware) in about a minute.

---

## Key findings

> **Congestion-aware routing cuts peak aisle load by ~45% for ~0.1% longer trips.**

| Finding | Result |
|---|---|
| Busiest segment reduction | **38–49%** per map (avg **45%**) |
| Cost in travel time | **+0.03% to +0.25%** per scenario (avg **+0.12%**) |
| Busiest segment, all 20 runs | **9.6 → 5.2 robots** on average |
| Top-10 busiest segments | **45–53%** quieter |
| Alpha sensitivity | Peak load drops sharply up to `α ≈ 0.1`, then flattens while cost keeps rising |
| Single-cell blockage | **15 of 200** robots rerouted, **+0.1%** total distance, **0** robots stranded |

The trade-off is heavily asymmetric: a sub-percent increase in distance buys roughly a 45% reduction in the worst-case aisle load. The effect is consistent — it appears in **all 20** map/scenario pairs, not just on average.

---

## Dataset

The notebook expects the [Moving AI Lab MAPF benchmark](https://movingai.com/benchmarks/mapf.html) warehouse maps and random scenario files:

```text
Warehouse/
├── warehouse_congestion.ipynb
├── README.md
├── mapf-map/          # warehouse-*.map       (grid layouts)
└── scen-random/       # warehouse-*-random-N.scen  (start/goal pairs)
```

| Map | Grid size (w × h) | Free cells | Possible moves | Connected regions |
|---|---:|---:|---:|---:|
| `10-20-10-2-1` | 161 × 63 | 5,699 | 8,778 | 1 |
| `10-20-10-2-2` | 170 × 84 | 9,776 | 16,902 | 1 |
| `20-40-10-2-1` | 321 × 123 | 22,599 | 35,158 | 1 |
| `20-40-10-2-2` | 340 × 164 | 38,756 | 67,412 | 1 |

**File formats**

- `.map` — grid of characters after a `map` header line: `.` = free floor, `@` = shelf/wall, `G`/`S` = walkable terrain.
- `.scen` — one line per robot; columns 5–8 hold `start_x start_y goal_x goal_y`. Each file contains 1,000 pairs, of which `NUM_ROBOTS` are sampled per run.
- Cells are indexed as **(x, y)** = (column, row).

---

## Methodology

### 1. Graph construction

Each map is converted **once** into a NetworkX grid graph:

- one **node** per free cell,
- one **edge** per up/down/left/right neighbour pair,
- obstacle cells (`@`) are removed,
- all four layouts are fully connected (single region), so every start/goal pair is reachable.

### 2. Routing model

Robots are routed **sequentially and greedily** using Dijkstra shortest paths. Two passes are compared:

| Pass | Edge cost |
|---|---|
| Baseline | `1` (pure shortest path, traffic ignored) |
| Congestion-aware | `1 + α × (robots already using that segment)` |

Because the cost is assessed at routing time, each robot *sees* the traffic left by earlier robots and can divert. Segment names are direction-agnostic, so an aisle is penalised in both directions.

### 3. Metrics

| Metric | Definition |
|---|---|
| `avg_trip` | Mean number of moves per robot |
| `peak_segment` | Robots on the single busiest segment |
| `top10_avg` | Mean load of the 10 busiest segments |
| `longer_%` | `100 × (trip_after / trip_before − 1)` — the *cost* |
| `peak_drop_%` | `100 × (1 − peak_after / peak_before)` — the *benefit* |

### 4. Experiment design

- **Breadth:** every map × 5 scenario files = 20 independent runs.
- **Sampling:** 200 robots sampled per scenario with a fixed seed (`42`), so runs are comparable.
- **Detail:** one map/scenario is examined with heatmaps and a hotspot ranking.
- **Sensitivity:** `α ∈ {0, 0.05, 0.1, 0.2, 0.5, 1.0}` on the example scenario.
- **Robustness:** the busiest cell is blocked to test how much slack the layout has.

---

## Results

### Congestion-aware routing vs. baseline
*Mean over 5 scenarios per map, α = 0.1, 200 robots.*

| Map | Trip before | Trip after | Longer % | Peak before | Peak after | Peak drop % | Top-10 drop % |
|---|---:|---:|---:|---:|---:|---:|---:|
| `10-20-10-2-1` | 81.37 | 81.55 | +0.22% | 13.6 | 7.4 | **45.6%** | 46.0% |
| `10-20-10-2-2` | 89.74 | 89.82 | +0.08% | 9.4 | 5.0 | **46.8%** | 45.1% |
| `20-40-10-2-1` | 158.10 | 158.29 | +0.12% | 7.8 | 4.8 | **38.5%** | 44.9% |
| `20-40-10-2-2` | 182.19 | 182.28 | +0.05% | 7.4 | 3.8 | **48.6%** | 53.4% |

Two patterns stand out:

- **Larger maps have lower absolute peaks** — the same 200 robots spread over more cells. The *relative* drop is therefore the fair comparison across layouts, and it is consistently large.
- **The cost barely moves** — every map stays under +0.25%, while the benefit is 38–49%.

### Alpha sweep (example scenario, `10-20-10-2-1`)

| α | Avg trip | Peak segment | Top-10 avg | Longer % |
|---:|---:|---:|---:|---:|
| 0.00 | 81.44 | 13 | 12.3 | 0.00% |
| 0.05 | 81.51 | 8 | 7.3 | +0.09% |
| **0.10** | **81.64** | **7** | **6.7** | **+0.25%** |
| 0.20 | 81.84 | 7 | 7.0 | +0.49% |
| 0.50 | 82.87 | 8 | 7.3 | +1.76% |
| 1.00 | 85.24 | 7 | 7.0 | +4.67% |

The curve has a clear knee: most of the benefit is captured by `α = 0.10`. Beyond that, peak load is flat or slightly worse while trip length keeps climbing — so `α ≈ 0.1` is the operating point.

### Where the traffic actually is

The busiest cells on `10-20-10-2-1` (excluding start/goal cells, which are busy by definition) are corridor pinch points rather than open floor:

| Rank | Cell (x, y) | Robots through it |
|---:|---|---:|
| 1 | (113, 43) | 15 |
| 2 | (47, 19) | 15 |
| 3 | (47, 34) | 15 |
| 4 | (47, 37) | 15 |
| 5 | (80, 4) | 15 |
| 6 | (47, 43) | 14 |
| 7 | (47, 16) | 14 |
| 8 | (69, 25) | 13 |
| 9 | (58, 19) | 13 |
| 10 | (69, 19) | 13 |

Heatmaps in the notebook show the before/after traffic field side by side. Note that the colour scales are independent per panel — compare the **maximum on each colour bar**, not the shades.

### Failure experiment: block the busiest cell

Blocking cell **(113, 43)** — simulating a broken robot or fallen box — and rerouting everyone on pure shortest paths:

| Metric | Result |
|---|---|
| Robots whose original route used the cell | 15 of 200 |
| Robots with no route afterwards | **0** |
| Total travel distance | 16,288 → 16,298 (**+0.1%**) |
| Busiest cell load | 15 → **15** |

The layout absorbs the loss with a negligible detour, but the peak load does **not** fall: diverted robots pile onto neighbouring cells and recreate the hotspot one aisle over. Blocking is survivable; it is not a congestion fix.

### Stability across scenarios

| Map | Longer % range | Peak drop % range |
|---|---|---|
| `10-20-10-2-1` | 0.2–0.3% | 33.3–50.0% |
| `10-20-10-2-2` | 0.1–0.1% | 37.5–54.5% |
| `20-40-10-2-1` | 0.1–0.2% | 28.6–50.0% |
| `20-40-10-2-2` | 0.0–0.1% | 42.9–57.1% |

The lower bound of `peak_drop` stays comfortably above zero on every map, so the improvement is not a lucky draw. Report the averages, not a single scenario.

---

## Project structure

```text
Warehouse/
├── warehouse_congestion.ipynb   # full analysis: setup → routing → visuals → findings
├── README.md                    # this file
├── mapf-map/                    # warehouse-*.map (4 layouts)
└── scen-random/                 # warehouse-*-random-N.scen (1,000 pairs each)
```

Notebook layout:

| Section | Contents |
|---|---|
| Setup | Imports and tunable parameters |
| Step 1 | Load maps/scenarios, build one graph per layout, map inventory table |
| Step 2 | Route all 20 map/scenario pairs twice, per-run and per-map comparison tables, charts |
| Step 3 | Deep dive: heatmaps, hotspot ranking, alpha sweep |
| Step 4 | Block the busiest cell and measure the damage |
| Step 5 | Auto-generated summary sentences |
| Extra | Stability/spread tables |
| Closing | Limitations and glossary |

---

## Getting started

### Requirements

```bash
python -m venv .venv
source .venv/bin/activate
pip install jupyter numpy pandas networkx matplotlib
```

### Data

Download the warehouse maps and random scenarios from the [Moving AI Lab MAPF benchmark](https://movingai.com/benchmarks/mapf.html) and place them as:

```text
mapf-map/warehouse-10-20-10-2-1.map
scen-random/warehouse-10-20-10-2-1-random-1.scen
...
```

The notebook discovers `warehouse-*.map` automatically and pairs each map with its `warehouse-<map>-random-*.scen` files.

### Run

```bash
jupyter lab warehouse_congestion.ipynb
```

Run all cells. With the defaults (4 maps × 5 scenarios × 200 robots, two passes) this takes roughly **one minute**.

### Configuration

Edit the block at the top of the notebook:

| Parameter | Default | Effect |
|---|---|---|
| `MAP_DIR` | `mapf-map` | Map directory |
| `SCEN_DIR` | `scen-random` | Scenario directory |
| `SCENS_PER_MAP` | `5` | Scenario files per map (`None` = all 25, slower) |
| `NUM_ROBOTS` | `200` | Robots sampled per scenario file |
| `ALPHA` | `0.1` | Congestion sensitivity (`0` = traffic-blind) |
| `SEED` | `42` | Fixes the robot sample and ordering |
| `FREE_CELLS` | `.GS` | Characters a robot may stand on |

---

## Limitations

- **No time dimension.** The model counts *route overlap*, not robots meeting at the same instant. There is no scheduling, waiting, or collision avoidance, so it identifies *where* traffic would build up — not literal jams.
- **Greedy, order-dependent routing.** Robots are routed one at a time; a different order gives slightly different loads. Fixed seeds and multiple scenarios smooth this out, but it is not a global optimum.
- **Sampled demand.** 200 of 1,000 pairs per scenario file are used, trading realism for runtime.
- **Limited failure testing.** One blocked cell on one map. A whole aisle, or all maps, would be a harder test.
- **Uniform terrain cost.** `.`, `G`, and `S` are treated as equally fast; real warehouses have slow zones, one-way aisles, and charging constraints.

## Next steps

- Add a time-expanded model with reservation tables (true MAPF) to measure delay and deadlocks, not just overlap.
- Sweep `α` across **all** scenarios and pick the value that maximises peak drop per unit of added distance.
- Test aisle-level and multi-cell closures rather than a single cell.
- Weight edges by real travel speed per terrain type.
- Compare against global methods (e.g. min-cost flow, ILP) to quantify the greedy gap.

---

## Glossary

| Term | Meaning |
|---|---|
| Node | One free cell of the map |
| Edge / segment | A move between two neighbouring cells |
| Path | The list of cells a robot walks through |
| Hotspot | A cell or segment used by many robots |
| `ALPHA` | How strongly robots avoid busy segments |
| Heatmap | The map coloured by number of robots per cell |

## References

- Moving AI Lab — MAPF benchmark maps and scenarios: <https://movingai.com/benchmarks/mapf.html>
- NetworkX documentation: <https://networkx.org/documentation/stable/>
