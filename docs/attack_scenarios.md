# Cyber Attack Scenarios - TuringDB Graph

A **cyber attack knowledge base** as a graph: thousands of documented attack scenarios -
each with a description, step-by-step playbook, impact, detection method, and remediation -
linked to the **MITRE ATT&CK techniques** they use, the **tools** involved, and a topical
**category**.

- **Graph name:** `attack_scenarios`  ·  **Store:** [`graphs/attack_scenarios/`](../graphs/attack_scenarios)

## Schema

### Nodes (18,354)

| Label | Count | Key | Properties |
|---|--:|---|---|
| `Attack` | 14,133 | `attack_id` | `attack_id`, `name`, `attack_type`, `target_type`, `vulnerability`, `scenario_description`, `attack_steps`, `impact`, `detection_method`, `solution`, `tags`, `source`, `tools_raw` |
| `Tool` | 3,240 | `name` | `name`, `mentions` (how often the tool appears across attacks) |
| `MitreTechnique` | 918 | `code` | `code` (ATT&CK ID, e.g. `T1078`), `name` |
| `Category` | 63 | `name` | `name` (e.g. `Insider Threat`, `DFIR`, `DevSecOps & CI/CD Security`) |

Each `Attack` is a self-contained scenario: the long-form `attack_steps`, `detection_method`,
and `solution` fields make the dataset usable as a playbook / triage reference, not just a
link graph.

### Edges (60,014)

| Edge | From → To | Count | Meaning |
|---|---|--:|---|
| `USES_TOOL` | `Attack` → `Tool` | 30,935 | Tool used to carry out the attack |
| `USES_TECHNIQUE` | `Attack` → `MitreTechnique` | 14,946 | MITRE ATT&CK technique the attack maps to |
| `IN_CATEGORY` | `Attack` → `Category` | 14,133 | Topical category the attack belongs to |

### Shape

```
(Category) <--IN_CATEGORY-- (Attack) --USES_TECHNIQUE--> (MitreTechnique)
                               \--USES_TOOL------------> (Tool)
```

## What it enables

- **Attack playbooks & triage** - pull a full scenario (steps, impact, detection, fix) for any
  attack type, filterable by `Category` or `attack_type`.
- **MITRE ATT&CK mapping** - pivot from a technique (`T####`) to every attack that uses it, or
  profile an attack's technique coverage.
- **Tooling intelligence** - rank tools by how many attacks reference them (`mentions`), or find
  which attacks rely on a given tool (e.g. Burp Suite, Wireshark, sqlmap).
- **Defensive coverage analysis** - group by category/technique to see where detections and
  remediations are documented vs. sparse.

## Quick start

```bash
# from the repo root
turingdb start -turing-dir "$(pwd)" -ui -ui-port 8080
```

```python
from turingdb import TuringDB
c = TuringDB("json", host="http://localhost:6666")
c.load_graph("attack_scenarios"); c.set_graph("attack_scenarios")

# scenarios that use a given MITRE technique
c.query("""
  MATCH (a:Attack)-[:USES_TECHNIQUE]->(t:MitreTechnique {code:'T1078'})
  RETURN a.name, a.attack_type, a.impact LIMIT 20
""")

# most-referenced tools across all attacks
c.query("MATCH (t:Tool) RETURN t.name, t.mentions ORDER BY t.mentions DESC LIMIT 15")

# full playbook for one attack, with its tools and category
c.query("""
  MATCH (a:Attack)-[:IN_CATEGORY]->(cat:Category)
  OPTIONAL MATCH (a)-[:USES_TOOL]->(tool:Tool)
  WHERE a.name = 'Authentication Bypass via SQL Injection'
  RETURN a.scenario_description, a.attack_steps, a.detection_method, a.solution,
         cat.name, collect(tool.name) AS tools
""")
```

## License

Released under the **MIT License**. MITRE ATT&CK technique IDs/names referenced in the data
remain © The MITRE Corporation. Indicate the data was converted into a TuringDB graph.
