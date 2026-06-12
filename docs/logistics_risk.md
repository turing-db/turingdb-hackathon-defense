# Supply Chain Risk & Performance - TuringDB Graph

A graph view of the **Supply Chain Risk and Performance Indicators** dataset: 113,097
logistics instances enriched with operational metrics (inventory, shipping cost, lead time,
handling-equipment availability, delivery deviation) and external risk context (delay
probability, disruption likelihood, route risk, customs clearance, weather severity), each
tied to a supplier, product, and country and carrying a risk classification.

- **Graph name:** `logistics_risk`  ·  **Store:** [`graphs/logistics_risk/`](../graphs/logistics_risk)

## Schema

### Nodes (117,718)

| Label | Count | Key | Properties |
|---|---|---|---|
| `Shipment` | 113,097 | `shipment_id` (`INST######`) | `name`, `warehouse_inventory_level`, `handling_equipment_availability`, `order_fulfillment_status`, `weather_condition_severity`, `shipping_costs`, `supplier_reliability_score`, `lead_time_days`, `historical_demand`, `cargo_condition_status`, `route_risk_level`, `customs_clearance_time`, `disruption_likelihood_score`, `delay_probability`, `delivery_time_deviation` |
| `Supplier` | 3,524 | `supplier_id` | `name` (each maps to exactly one product and one country) |
| `Product` | 1,000 | `product_id` | `name` |
| `Country` | 94 | `name` | supplier location |
| `RiskClassification` | 3 | `name` | `Low Risk`, `Moderate Risk`, `High Risk` |

`Shipment`, `Supplier`, and `Product` carry a `name` display property (`Label: id`,
e.g. `Shipment: INST000001`) for the visualizer; `Country` and `RiskClassification` use
their value as `name`.

Every per-row numeric metric lives **on the `Shipment` node**; the identifiers and the risk
label are pulled out into shared nodes.

### Edges (233,242)

| Edge | From → To | Count | Meaning |
|---|---|---|---|
| `FROM_SUPPLIER` | `Shipment` → `Supplier` | 113,097 | which supplier handled the instance |
| `CLASSIFIED_AS` | `Shipment` → `RiskClassification` | 113,097 | assigned risk label |
| `SUPPLIES` | `Supplier` → `Product` | 3,524 | the product a supplier provides |
| `LOCATED_IN` | `Supplier` → `Country` | 3,524 | supplier's country |

### Shape

```
(RiskClassification) <--CLASSIFIED_AS-- (Shipment) --FROM_SUPPLIER--> (Supplier)
                                                                          |
                                                            SUPPLIES /        \ LOCATED_IN
                                                                    v          v
                                                              (Product)    (Country)
```

Because each `Supplier` resolves to a single `Product` and `Country`, a shipment reaches its
product and country with one extra hop through the supplier - no redundant direct edges.

## What it enables

- **Risk by geography / product** - traverse `RiskClassification` → `Shipment` →
  `Supplier` → `Country`/`Product` to see where high-risk instances concentrate.
- **Supplier reliability vs. outcomes** - compare `supplier_reliability_score`,
  `delay_probability`, and `delivery_time_deviation` across a supplier's shipments.
- **Country risk profiles** - aggregate route risk, customs clearance, and disruption
  likelihood per country.
- **Predictive-model feature pulls** - the `Shipment` nodes carry a ready-made feature
  vector with the `risk_classification` target one hop away.

## Risk distribution

| Risk | Shipments |
|---|---|
| High Risk | 84,368 |
| Moderate Risk | 17,783 |
| Low Risk | 10,946 |

## Quick start

```bash
# from the repo root
turingdb start -turing-dir "$(pwd)" -ui -ui-port 8080
```

```python
from turingdb import TuringDB
c = TuringDB("json", host="http://localhost:6666")
c.load_graph("logistics_risk"); c.set_graph("logistics_risk")

# high-risk shipments with their supplier and country
c.query("""
  MATCH (s:Shipment)-[:CLASSIFIED_AS]->(:RiskClassification {name:'High Risk'}),
        (s)-[:FROM_SUPPLIER]->(sup:Supplier)-[:LOCATED_IN]->(co:Country)
  RETURN s.shipment_id, sup.supplier_id, co.name, s.delivery_time_deviation
""")

# pull a feature frame for shipments of one product (aggregate in pandas)
df = c.query("""
  MATCH (s:Shipment)-[:FROM_SUPPLIER]->(:Supplier)-[:SUPPLIES]->(:Product {product_id:'P0353'})
  RETURN s.delay_probability, s.route_risk_level, s.customs_clearance_time
""")
df.describe()
```

## License

Source dataset licensed under the **[Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)**.
Retain the copyright and license notice (and any `NOTICE` file) and state the changes made.
Provided "as is", without warranty.

*Source data is provided for analytics/modeling; values represent synthetic supply-chain
instances.*
