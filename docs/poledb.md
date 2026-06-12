# POLE Crime Investigation - TuringDB Graph

A **POLE** (Person · Object · Location · Event) crime-investigation graph: a UK-style
policing dataset linking people, their contact details and associates, crimes and the
officers investigating them, vehicles, phone-call records, and geocoded locations.

- **Graph name:** `poledb`  ·  **Store:** [`graphs/poledb/`](../graphs/poledb)

## Schema

### Nodes (61,521)

| Label | Count | Key | Properties |
|---|--:|---|---|
| `Crime` | 28,762 | `id` | `id`, `type`, `date`, `last_outcome`, `charge`, `note`, `description` |
| `Location` | 14,904 | `address` | `address`, `latitude`, `longitude`, `postcode` |
| `PostCode` | 14,196 | `code` | `code` |
| `Officer` | 1,000 | `badge_no` | `badge_no`, `rank`, `name`, `surname` |
| `Vehicle` | 1,000 | `reg` | `make`, `model`, `reg`, `year` |
| `PhoneCall` | 534 | - | `call_date`, `call_time`, `call_duration`, `call_type` (`CALL`/`SMS`) |
| `Person` | 369 | `nhs_no` | `nhs_no`, `surname`, `name`, `age` |
| `Phone` | 328 | `phoneNo` | `phoneNo` |
| `Email` | 328 | `email_address` | `email_address` |
| `Area` | 93 | `areaCode` | `areaCode` |
| `Object` | 7 | - | `description`, `type` (e.g. seized `Evidence`) |

### Edges (105,840)

| Edge | From → To | Count | Meaning |
|---|---|--:|---|
| `OCCURRED_AT` | `Crime` → `Location` | 28,762 | Where the crime took place |
| `INVESTIGATED_BY` | `Crime` → `Officer` | 28,762 | Officer assigned to the crime |
| `LOCATION_IN_AREA` | `Location` → `Area` | 14,904 | Location's policing area |
| `HAS_POSTCODE` | `Location` → `PostCode` | 14,904 | Location's postcode |
| `POSTCODE_IN_AREA` | `PostCode` → `Area` | 14,196 | Postcode's area |
| `INVOLVED_IN` | `Vehicle` → `Crime` | 985 | Vehicle linked to a crime |
| `KNOWS` | `Person` → `Person` | 586 | Known associate |
| `CALLER` | `PhoneCall` → `Phone` | 534 | Call's originating phone |
| `CALLED` | `PhoneCall` → `Phone` | 534 | Call's destination phone |
| `CURRENT_ADDRESS` | `Person` → `Location` | 368 | Person's current address |
| `HAS_PHONE` | `Person` → `Phone` | 328 | Person's phone |
| `HAS_EMAIL` | `Person` → `Email` | 328 | Person's email |
| `KNOWS_SN` | `Person` → `Person` | 241 | Linked via social network |
| `FAMILY_REL` | `Person` → `Person` | 155 | Family relationship |
| `KNOWS_PHONE` | `Person` → `Person` | 118 | Linked via phone contact |
| `KNOWS_LW` | `Person` → `Person` | 80 | Linked - lives with |
| `PARTY_TO` | `Person` → `Crime` | 55 | Person is a party to a crime |

### Shape

```
(Email) <--HAS_EMAIL-- (Person) --HAS_PHONE--> (Phone) <--CALLER/CALLED-- (PhoneCall)
                          |  \--KNOWS/KNOWS_SN/KNOWS_PHONE/KNOWS_LW/FAMILY_REL--> (Person)
                          |  \--CURRENT_ADDRESS--> (Location)
                          \--PARTY_TO--> (Crime) <--INVOLVED_IN-- (Vehicle)
                                            |  \--INVESTIGATED_BY--> (Officer)
                                            \--OCCURRED_AT--> (Location) --HAS_POSTCODE--> (PostCode)
                                                                  |  \--LOCATION_IN_AREA--> (Area)
                                                                  (PostCode) --POSTCODE_IN_AREA--> (Area)
```

## What it enables

- **Link / association analysis** - traverse `KNOWS`, `FAMILY_REL`, and shared phones/addresses
  to map a person's network and find indirect connections between suspects.
- **Crime-to-people-to-assets investigation** - go from a `Crime` to its location, investigating
  officer, the people `PARTY_TO` it, and any `Vehicle` `INVOLVED_IN` it.
- **Communications intelligence** - reconstruct call/SMS patterns over `PhoneCall` → `Phone`,
  tying call records back to the people who own the numbers.
- **Geographic crime mapping** - every `Location` carries lat/long and rolls up through
  `PostCode` → `Area` for hotspot and area-level analysis.

## Quick start

```bash
# from the repo root
turingdb start -turing-dir "$(pwd)" -ui -ui-port 8080
```

```python
from turingdb import TuringDB
c = TuringDB("json", host="http://localhost:6666")
c.load_graph("poledb"); c.set_graph("poledb")

# crimes at a location with the investigating officer
c.query("""
  MATCH (cr:Crime)-[:OCCURRED_AT]->(l:Location),
        (cr)-[:INVESTIGATED_BY]->(o:Officer)
  RETURN cr.type, cr.date, l.address, o.surname, o.rank LIMIT 20
""")

# a person's known associates (any KNOWS-style link)
c.query("""
  MATCH (p:Person {surname:'Hamilton'})-[r:KNOWS|KNOWS_SN|KNOWS_PHONE|KNOWS_LW|FAMILY_REL]->(o:Person)
  RETURN p.name, type(r), o.name, o.surname
""")

# who a phone number called (call detail records)
c.query("""
  MATCH (call:PhoneCall)-[:CALLER]->(:Phone)<-[:HAS_PHONE]-(from:Person),
        (call)-[:CALLED]->(:Phone)<-[:HAS_PHONE]-(to:Person)
  RETURN from.surname, to.surname, call.call_type, call.call_date, call.call_duration LIMIT 20
""")
```

## License

POLE (Person–Object–Location–Event) is a well-known crime-investigation example dataset
derived from UK open police data ([data.police.uk](https://data.police.uk/about/)), published
under the [Open Government Licence v3.0](https://www.nationalarchives.gov.uk/doc/open-government-licence/version/3/);
personal entities (people, phones, emails) are synthetic. Retain the OGL attribution and
indicate the data was converted into a TuringDB graph.
