# SDO — Syntelligo Domain Ontology

A lightweight, multilingual domain taxonomy (depth 1–3) for **semantic tagging** and **discovery**.  
Canonical format: **JSON**. Also ships with **CSV** and **SKOS/Turtle** for interoperability.  
Graph-friendly: simple parent→child edges (`HAS_CHILD`) and stable IDs.

> **Why SDO?**  
> Clarity over complexity. Shallow hierarchy for reliable roll-up/drill-down, multilingual labels for direct UI use, and hassle-free imports into Memgraph/Neo4j.

---

## Features

- **Shallow hierarchy** (levels 1–3) → easy traversal and reasoning
- **Multilingual labels** (e.g., `en`, `nob`)
- **Stable IDs & slugs**, optional crosswalks (`ddc`, `mesh`, `eurovoc`, `wiki`)
- **Interoperable formats**: JSON (canonical), CSV, SKOS/Turtle
- **Graph-ready**: import & validation snippets included

---

## Canonical Data Format (JSON)

`./sdo.json`

```json
{
  "domains": [
    {
      "id": "SDO.01",
      "slug": "biology-life-sciences",
      "depth": 1,
      "scheme": "SDO",
      "labels": {
        "en": "Biology & Life Sciences",
        "nob": "Biologi og livsvitenskap"
      },
      "ext": { "ddc": ["570"], "wiki": "Biology", "eurovoc": [], "mesh": "A" },
      "status": "active",
      "version": "2025-09"
    }
  ],
  "edges": [
    {
      "parent": "SDO.01",
      "child": "SDO.01.01",
      "childLabel": { "en": "Botany", "nob": "Botanikk" }
    },
    {
      "parent": "SDO.01.01",
      "child": "SDO.01.01.01",
      "childLabel": {
        "en": "Plant Anatomy & Morphology",
        "nob": "Planteanatomi og morfologi"
      }
    }
  ]
}
```

**Notes**

- `edges` is a **flat array** of `{ parent, child, childLabel }`.
- `depth` can be derived from `id` (`len(id.split(".")) - 1`).
- `childLabel` helps create child nodes during import if not already in `domains`.

---

## Alternative Formats

### CSV (`./formats/csv/`)

**domains.csv**

```
id,slug,depth,scheme,labels_en,labels_nob,ext_ddc,ext_wiki,ext_mesh,status,version
SDO.01,biology-life-sciences,1,SDO,Biology & Life Sciences,Biologi og livsvitenskap,"[\"570\"]",Biology,A,active,2025-09
```

**edges.csv**

```
parent,child,child_en,child_nob
SDO.01,SDO.01.01,Botany,Botanikk
```

### SKOS/Turtle (`./formats/skos/sdo.ttl`)

```ttl
@prefix sdo:  <https://syntelligo.com/sdo/> .
@prefix skos: <http://www.w3.org/2004/02/skos/core#> .
@prefix dct:  <http://purl.org/dc/terms/> .

sdo:SDO.01 a skos:Concept ;
  skos:prefLabel "Biology & Life Sciences"@en ;
  skos:prefLabel "Biologi og livsvitenskap"@nb ;
  dct:identifier "SDO.01" ;
  skos:inScheme sdo:scheme .

sdo:SDO.01.01 a skos:Concept ;
  skos:prefLabel "Botany"@en ;
  skos:prefLabel "Botanikk"@nb ;
  skos:broader sdo:SDO.01 ;
  dct:identifier "SDO.01.01" ;
  skos:inScheme sdo:scheme .

sdo:scheme a skos:ConceptScheme ;
  dct:title "SDO — Syntelligo Domain Ontology"@en .
```

---

## Quick Start (Graph Import)

### Memgraph/Neo4j (Cypher)

```cypher
CREATE CONSTRAINT sdo_id_unique IF NOT EXISTS ON (n:SDO) ASSERT n.id IS UNIQUE;

UNWIND $domains AS row
WITH row, coalesce(row.depth, size(split(row.id, ".")) - 1) AS calc_depth
MERGE (n:SDO {id: row.id})
SET n.slug = row.slug, n.scheme = "SDO", n.labels = row.labels, n.ext = row.ext,
    n.depth = calc_depth, n.status = coalesce(row.status,"active"), n.version = coalesce(row.version,"2025-09");

UNWIND $edges AS e
WITH e, size(split(e.child, ".")) - 1 AS child_depth
MERGE (c:SDO {id: e.child})
ON CREATE SET c.labels = coalesce(e.childLabel, {}), c.slug = toLower(replace(coalesce(e.childLabel.en, e.child), " ", "-")),
              c.scheme = "SDO", c.ext = {}, c.depth = child_depth, c.status = "active", c.version = "2025-09";

UNWIND $edges AS e
MATCH (p:SDO {id: e.parent})
MATCH (c:SDO {id: e.child})
MERGE (p)-[:HAS_CHILD {scheme:"SDO"}]->(c);
```

### Validation Queries

```cypher
// Count per depth
MATCH (n:SDO) RETURN n.depth AS depth, count(*) AS cnt ORDER BY depth;

// Orphans (depth > 1 without incoming HAS_CHILD)
MATCH (n:SDO) WHERE n.depth > 1
OPTIONAL MATCH ()-[:HAS_CHILD]->(n)
WITH n, count(*) AS incoming
WHERE incoming = 0
RETURN n;

// Cycles (should return none)
MATCH p=(a:SDO)-[:HAS_CHILD*1..]->(a) RETURN p LIMIT 1;
```

---

## GitHub Pages (Optional Viewer)

1. Create `docs/` and place a simple `index.html` that loads `sdo.json` and renders a table (e.g., with DataTables).
2. **Settings → Pages** → _Deploy from a branch_ → **main** `/docs`.
3. Your SDO table will be live at `https://<org>.github.io/<repo>/`.

---

## Versioning & License

- **Versioning:** tag releases as `vX.Y.Z` and set `version` on nodes; deprecate instead of deleting.
- **License (data):** recommended **CC BY 4.0** (or MIT if you prefer maximum reuse).
- Include `CITATION.cff` for academic references.

---

## Contributing

Contributions welcome! Please:

- keep labels unambiguous (English required; other languages welcome),
- justify new/changed edges with a clear parent rationale,
- stay within depth ≤ 3 unless there’s a strong use case,
- run validation before submitting PRs.

---

## Links

- Website: **https://syntelligo.com**
- Org profile & other projects: see this GitHub organization
