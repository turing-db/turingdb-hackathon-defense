# Global Power Plant Database — TuringDB Graph

A graph view of the **Global Power Plant Database** (WRI, v1.3.0): ~34,900 power plants
worldwide with location, capacity, primary/secondary fuels, owner, and reported/estimated
generation.

- **Graph name:** `power_plants`  ·  **Store:** [`graphs/power_plants/`](../graphs/power_plants)

## Schema

### Nodes (45,262)

| Label | Count | Key | Properties |
|---|---|---|---|
| `PowerPlant` | 34,936 | `gppd_idnr` | `name`, `gppd_idnr`, `capacity_mw`, `latitude`, `longitude`, `primary_fuel`, `commissioning_year`, `owner`, `source`, `country_code`, `generation_gwh_2019`, `year_of_capacity_data` |
| `Country` | 167 | `country_code` (ISO3) | `name` (full country name), `country_code` |
| `Fuel` | 15 | fuel name | `name` (Solar, Hydro, Wind, Gas, Coal, Oil, Biomass, Waste, Nuclear, Geothermal, Storage, Other, Cogeneration, Petcoke, Wave and Tidal) |
| `Owner` | 10,144 | owner name | `name` |

**Every node has a `name` property.** For `PowerPlant` it is the plant name from the
dataset (falling back to `PowerPlant: <gppd_idnr>` if blank); for the other labels it is the
human-readable country / fuel / owner name.

### Edges (93,052)

| Edge | From → To | Count | Meaning |
|---|---|---|---|
| `LOCATED_IN` | `PowerPlant` → `Country` | 34,936 | Plant's country |
| `PRIMARY_FUEL` | `PowerPlant` → `Fuel` | 34,936 | Plant's main fuel |
| `ALSO_USES` | `PowerPlant` → `Fuel` | 2,312 | Secondary fuels (`other_fuel1/2/3`) |
| `OWNED_BY` | `PowerPlant` → `Owner` | 20,868 | Plant's owner (where known) |

### Shape

```
(Country) <--LOCATED_IN-- (PowerPlant) --PRIMARY_FUEL--> (Fuel)
                              |    \--ALSO_USES----------> (Fuel)
                              \--OWNED_BY--> (Owner)
```

## What it enables

- **Energy-security mapping** — generation capacity by country and fuel; fuel-mix and
  import/fuel-dependency profiles per country.
- **Ownership concentration** — which owners control the most capacity, and where.
- **Critical-infrastructure geolocation** — every plant carries lat/long for spatial joins.

## Quick start

```bash
# from the repo root
turingdb start -turing-dir "$(pwd)" -ui -ui-port 8080
```

```python
from turingdb import TuringDB
c = TuringDB("json", host="http://localhost:6666")
c.load_graph("power_plants"); c.set_graph("power_plants")

# plants in a country with fuel and capacity
c.query("""
  MATCH (p:PowerPlant)-[:LOCATED_IN]->(co:Country {country_code:'USA'}),
        (p)-[:PRIMARY_FUEL]->(f:Fuel)
  RETURN p.name, f.name, p.capacity_mw LIMIT 20
""")

# all plants for one owner
c.query("MATCH (p:PowerPlant)-[:OWNED_BY]->(o:Owner) WHERE o.name = 'EDF' RETURN p.name, p.capacity_mw")
```

## Notes & gotchas

- **Count edges with the arrow** `()-[r]->()`. The undirected form double-counts.
- **Grouped aggregation is limited** — `count(x)` grouped by a neighbour property does not
  group correctly in this engine version (it returns the global total per row). Pull the rows
  and aggregate in pandas instead.
- Always `set_graph("power_plants")` before querying, or you'll hit the `default` graph.
- Stray/malformed CSV rows are filtered: fuel values are validated against the 15 known
  categories, so column-shift artifacts don't create junk `Fuel` nodes.

## License

Source data: Global Power Plant Database, World Resources Institute —
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Retain attribution to WRI.
