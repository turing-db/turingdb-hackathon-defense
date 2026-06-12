# Drone Swarm Coordination — TuringDB Graph

A graph view of a **synthetic drone-swarm coordination** dataset: 20 drones tracked across
1,000 timesteps (20,000 telemetry readings), each capturing 3D position, velocity, battery,
signal strength, collision warnings, and the swarm's current formation and mission. Modeled
so you can both pivot by formation/mission and **walk each drone's trajectory in time**.

- **Graph name:** `drone_swarm`  ·  **Store:** [`graphs/drone_swarm/`](../graphs/drone_swarm)

## Schema

### Nodes (21,028)

| Label | Count | Key | Properties |
|---|---|---|---|
| `Reading` | 20,000 | `reading_id` (`D{drone}_T{ts}`) | `name`, `timestamp`, `x`, `y`, `z`, `velocity`, `battery_level`, `signal_strength`, `collision_warning` (0/1) |
| `Drone` | 20 | `drone_id` | `name` |
| `TimeStep` | 1,000 | `t` | timestep index 0–999 |
| `FormationPattern` | 4 | `name` | `random`, `line`, `circle`, `grid` |
| `MissionStatus` | 4 | `name` | `return`, `delivery`, `exploration`, `surveillance` |

`Reading` and `Drone` carry a `name` display property (`Label: id`, e.g. `Reading: D0_T0`,
`Drone: 0`) for the visualizer; `FormationPattern` and `MissionStatus` use their value as
`name`. `TimeStep` has no `name`.

Per-reading numeric telemetry lives **on the `Reading` node**; identifiers and the
(time-varying) formation/mission labels are shared nodes.

### Edges (99,980)

| Edge | From → To | Count | Meaning |
|---|---|---|---|
| `OF_DRONE` | `Reading` → `Drone` | 20,000 | which drone produced the reading |
| `AT_TIME` | `Reading` → `TimeStep` | 20,000 | the timestep of the reading |
| `IN_FORMATION` | `Reading` → `FormationPattern` | 20,000 | swarm formation at that moment |
| `HAS_MISSION` | `Reading` → `MissionStatus` | 20,000 | mission at that moment |
| `NEXT` | `Reading` → `Reading` | 19,980 | next reading of the **same drone** (time-ordered) |

### Shape

```
(Drone) <--OF_DRONE-- (Reading) --AT_TIME--> (TimeStep)
                       |  |  ^
          IN_FORMATION |  | HAS_MISSION
                       v  |  v
        (FormationPattern) (MissionStatus)

  trajectory:  (Reading t0) --NEXT--> (Reading t1) --NEXT--> (Reading t2) ...   (per drone)
```

## What it enables

- **Trajectory traversal** — follow a drone's `NEXT` chain to reconstruct its flight path,
  battery decay, or signal profile over time.
- **Collision analysis** — find `collision_warning = 1` readings and pivot to the formation,
  mission, drone, and timestep where they occur.
- **Formation / mission breakdowns** — how readings (and warnings) distribute across the
  four formations and four mission types.
- **Cross-drone snapshots** — all readings sharing a `TimeStep` give the swarm's full state
  at one instant.

## Quick start

```bash
# from the repo root
turingdb start -turing-dir "$(pwd)" -ui -ui-port 8080
```

```python
from turingdb import TuringDB
c = TuringDB("json", host="http://localhost:6666")
c.load_graph("drone_swarm"); c.set_graph("drone_swarm")

# walk a drone's trajectory (first few hops)
c.query("MATCH (r:Reading {reading_id:'D0_T0'})-[:NEXT]->(b)-[:NEXT]->(d) RETURN r.x, b.x, d.x")

# collision-warning readings with formation + mission
df = c.query("""
  MATCH (r:Reading)-[:IN_FORMATION]->(f:FormationPattern),
        (r)-[:HAS_MISSION]->(m:MissionStatus)
  WHERE r.collision_warning = 1
  RETURN r.reading_id, f.name AS formation, m.name AS mission
""")
df["formation"].value_counts()
```

## License

Source dataset licensed under **[Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)**.
Give appropriate credit, link the license, and indicate if changes were made.

*Source data is synthetic, generated for coordination/telemetry analysis.*
