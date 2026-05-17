# CHAIN — Cooperative Heterogeneous Agents for Industrial Waste Collection in Hostile Environments

> Multi-Agent Systems project — Master IA, 2026  
> **Group 25** — Mathys Bagnah, Xavier Plantier, Meriem Jelassi

---

## What is this project?

CHAIN simulates a team of autonomous robots working together to collect, process, and safely dispose of hazardous radioactive waste. The environment is a radioactive grid divided into three danger zones. Robots cannot enter zones beyond their radiation tolerance, so they must cooperate in a relay chain to bring waste from the most dangerous areas to the disposal site.

The project compares two scenarios:
- **Without communication** — each robot acts on its own observations only
- **With communication** — robots share information (waste locations, map data, handoff signals)

Built with [Mesa](https://mesa.readthedocs.io/) (agent-based modeling) and [Solara](https://solara.dev/) (interactive visualization).

---

## Quick Start

```sh
cd 25_robot_mission_MAS2026
pip install -r requirements.txt

# Interactive visualization in the browser
python run.py

# Single run with metrics saved to results/plots/
python run.py --batch --scenario "With communication" --steps 500

# Compare both scenarios over multiple random seeds
python run.py --compare --seeds 20 --steps 500
```

Open `http://localhost:8765` after launching the visualization.

---

## How the Environment Works

The grid is divided into three vertical zones of increasing radioactivity:

```
  Z1 (west)       Z2 (middle)      Z3 (east)
 low radiation   mid radiation   high radiation
  0.00 – 0.33     0.33 – 0.66     0.66 – 1.00
```

- **Green waste** spawns randomly in Z1 at the start
- **Disposal site** is a single cell on the far east column (Z3)
- Each cell has a `RadioactivityAgent` that robots read to know which zone they are in

---

## The Waste Processing Chain

Waste cannot jump zones — it must be transformed and relayed step by step:

```
Z1                     Z1/Z2                  Z1/Z2/Z3
─────────────────────────────────────────────────────────
2× green waste  →  1× yellow waste  →  1× red waste  →  disposed
  (GreenAgent)       (YellowAgent)       (RedAgent)
```

Starting from `n` green wastes (must be a multiple of 4), exactly `n / 4` red wastes end up disposed.

---

## The Three Robot Types

| Robot | Allowed zones | Collects | Transforms | Max inventory |
|-------|--------------|----------|------------|---------------|
| `GreenAgent` | Z1 only | green waste | 2 green → 1 yellow | 2 |
| `YellowAgent` | Z1 + Z2 | yellow waste | 2 yellow → 1 red | 2 |
| `RedAgent` | Z1 + Z2 + Z3 | red waste | none (disposes directly) | 1 |

All robots follow the same **perceive → deliberate → act** loop each simulation step.

---

## How a Robot Thinks (Deliberation)

Each robot maintains a private knowledge dictionary updated from its percepts. The `deliberate()` method never accesses global state — it only reads this dictionary.

Decision priorities, in order:

1. **Deliver a produced waste** — if carrying a transformed waste, head to the handoff column (or disposal site for Red)
2. **Transform** — if holding 2 target wastes, transform immediately
3. **Pick up** — if standing on a target waste and inventory allows, pick it up
4. **Go to a known waste** — navigate toward the nearest previously spotted waste
5. **Deadlock release** — if holding a single waste for more than 8 steps, drop it at the handoff column for a teammate
6. **Explore** — move to the least-visited neighboring cell, biased toward expected waste sources

The **handoff column** (where transformed waste is dropped for the next robot tier) alternates between two x-positions every 35 steps to avoid congestion.

---

## Communication Protocol

When communication is enabled, robots exchange messages through a central mailbox:

| Message | Sent by | Received by | Effect |
|---------|---------|-------------|--------|
| `WASTE_FOUND` | Any robot spotting a target waste | Same-type teammates | Share waste position |
| `WASTE_GONE` | Robot whose shared waste disappeared | Same-type teammates | Invalidate stale info |
| `DISPOSAL_FOUND` | First robot to see the disposal site | Same-type teammates | Propagate location |
| `MAP_SHARE` | Every robot, every 4 steps | Same-type teammates | Share up to 6 observed cells |
| **Handoff notification** | Green/Yellow after dropping | Yellow/Red | Direct pointer to dropped waste — avoids exploration |

The handoff notification is the most impactful message: instead of the next tier discovering the dropped waste through random exploration, it receives exact coordinates and goes directly there.

---

## Anti-Deadlock Mechanism

Without communication, robots risk getting stuck: a robot holding one waste waits indefinitely for a second one that never comes. To prevent this:

- Each robot tracks `single_hold_steps` — how many consecutive steps it has held exactly one target waste
- After **8 steps**, it drops the waste at the handoff column regardless
- This frees the waste for another robot to pick up and combine with its own

---

## Project Structure

```
25_robot_mission_MAS2026/
├── agents.py      — GreenAgent, YellowAgent, RedAgent with full deliberation logic
├── model.py       — RobotMission: grid setup, action execution, data collection
├── objects.py     — Passive agents: RadioactivityAgent, WasteAgent, WasteDisposalAgent
├── server.py      — Solara web visualization (grid + real-time charts)
├── run.py         — CLI: visualization, batch run, multi-seed comparison
└── results/plots/ — Generated CSVs and PNG charts
```

---

## CLI Reference

```sh
python run.py [--batch | --compare] [OPTIONS]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--width` | 30 | Grid width (must be a multiple of 3) |
| `--height` | 10 | Grid height |
| `--green-robots` | 3 | Number of GreenAgents |
| `--yellow-robots` | 3 | Number of YellowAgents |
| `--red-robots` | 3 | Number of RedAgents |
| `--wastes` | 12 | Initial green wastes (must be a multiple of 4) |
| `--scenario` | `"With communication"` | `"No communication"` or `"With communication"` |
| `--seed` | 42 | Random seed for reproducibility |
| `--steps` | 500 | Maximum steps before timeout |
| `--seeds` | 10 | Number of seeds for `--compare` mode |
| `--output-dir` | `results/plots` | Where to save CSV and PNG outputs |

---

## Visualization

Launch with `python run.py`, then open `http://localhost:8765`.

The grid shows:
- **Radiation heatmap** (green → red background across zones)
- **Robot squares** — dark green (G), amber (Y), dark red (R); size grows with inventory
- **Waste circles** — green, yellow, red dots
- **Disposal site** — black ✕ marker on the east edge

Three real-time charts update every step:
- Waste distribution (counts by type)
- Mission progress (disposed wastes + exploration coverage)
- Coordination (message volumes by type)

---

## Results

Default parameters: `30×10` grid, 3 robots per type, 12 initial green wastes, 500 max steps, 10 seeds.

| Scenario | Completion rate | Avg steps to finish | Avg messages sent |
|----------|----------------|--------------------|--------------------|
| No communication | 100% | ~185 steps | 0 |
| With communication | 100% | ~135 steps | ~980 |

Communication gives approximately a **26% speedup**, driven mainly by handoff notifications eliminating exploratory search after each waste transformation.

To reproduce: `python run.py --compare --seeds 10 --steps 500`

---

## Dependencies

```
mesa[viz] >= 3.5.1, < 4
solara >= 1.32
matplotlib >= 3.7
pandas >= 2.0
numpy >= 1.24
networkx
```
