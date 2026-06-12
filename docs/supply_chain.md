# Aerospace Supply Chain - TuringDB Graph

A graph view of a synthetic but business-realistic **aerospace industrial supply chain**
(Tier-1 manufacturer / MRO context): low-volume, high-value parts, long and variable
supplier lead times, costly inventory, strict service levels, and quality/compliance risk.

- **Graph name:** `supply_chain`  ·  **Store:** [`graphs/supply_chain/`](../graphs/supply_chain)

## Schema

### Nodes (30,380)

| Label | Count | Key | Properties |
|---|---|---|---|
| `Part` | 300 | `part_id` | `name`, `part_family`, `criticality_class` (A=safety, B=operational, C=non-critical), `unit_cost`, `lead_time_days`, `supplier_risk_class`, `is_repairable`, `shelf_life_days` |
| `Supplier` | 40 | `supplier_id` | `name` |
| `Site` | 6 | `site_id` | `name` (assembly plants / MRO facilities) |
| `PurchaseOrder` | 29,666 | `po_id` | `name`, `order_date`, `promised_date`, `receipt_date`, `ordered_qty`, `received_qty` |
| `QualityIncident` | 368 | `incident_id` | `name`, `incident_date`, `defect_severity`, `defect_type`, `scrap_qty` |

Every node has a `name` display property (format `Label: id`, e.g. `Part: P00001`) used as the
label in the TuringDB visualizer.

### Edges (90,402)

| Edge | From → To | Count | Meaning |
|---|---|---|---|
| `SUPPLIED_BY` | `Part` → `Supplier` | 300 | Part's primary supplier |
| `FROM_SUPPLIER` | `PurchaseOrder` → `Supplier` | 30,034 | PO placed with supplier |
| `FOR_PART` | `PurchaseOrder` → `Part` | 29,666 | PO is for this part |
| `DELIVERED_TO` | `PurchaseOrder` → `Site` | 29,666 | PO destination site |
| `ABOUT_PART` | `QualityIncident` → `Part` | 368 | Incident concerns this part |
| `FROM_SUPPLIER` | `QualityIncident` → `Supplier` | 368 | Supplier responsible |
| `AT_SITE` | `QualityIncident` → `Site` | 368 | Where the incident occurred |

### Shape

```
(Supplier) <--SUPPLIED_BY-- (Part)
    ^                          ^
    |                          |
FROM_SUPPLIER             FOR_PART
    |                          |
   (PurchaseOrder) --DELIVERED_TO--> (Site)

(QualityIncident) --ABOUT_PART--> (Part)
                  --FROM_SUPPLIER--> (Supplier)
                  --AT_SITE--> (Site)
```

## What it enables

Graph traversal makes supplier-centric questions cheap: supplier reliability (OTIF via
PO promised vs. receipt dates), lead-time variability, supplier risk concentration, and
quality incident rates per supplier/part/site - and tracing a quality issue or delayed PO
back through the part to every affected site.

## Quick start

```bash
# from the repo root
turingdb start -turing-dir "$(pwd)" -ui -ui-port 8080
```

```python
from turingdb import TuringDB
c = TuringDB("json", host="http://localhost:6666")
c.load_graph("supply_chain"); c.set_graph("supply_chain")

# all POs for a part, with supplier and destination site
c.query("""
  MATCH (po:PurchaseOrder)-[:FOR_PART]->(p:Part {part_id:'P00001'}),
        (po)-[:FROM_SUPPLIER]->(s:Supplier),
        (po)-[:DELIVERED_TO]->(site:Site)
  RETURN po.po_id, s.supplier_id, site.site_id, po.ordered_qty
""")

# quality incidents per supplier (aggregate in pandas)
df = c.query("MATCH (qi:QualityIncident)-[:FROM_SUPPLIER]->(s:Supplier) RETURN s.supplier_id AS supplier")
df["supplier"].value_counts()
```

## License

Source dataset licensed under the **[MIT License](https://opensource.org/license/mit)**.
Retain the original copyright and MIT permission notice. Provided "as is", without warranty.

*Source data is fully synthetic and does not represent any real company.*
