# RV-KGF - Relationship Visualizer Knowledge Graph Format

**RV-KGF** is the JSON export schema produced by [Excel to Graphviz](https://exceltographviz.com/) ([jjlong150/ExcelToGraphviz](https://github.com/jjlong150/ExcelToGraphviz)) for representing a diagram's relationship data as a clean, AI- and graph-database-ready knowledge graph, independent of any rendering instructions.

This repository is the format's specification: the schema reference, the design rationale, a comparison against established graph-serialization standards, a JSON Schema for validation, and worked examples.

> RV-KGF is a **documentation and interoperability spec**, not a library. Excel to Graphviz is the reference producer; any tool can be a consumer.

## Why this exists

Excel to Graphviz already exports two artifacts from a workbook: a rendered image (for people) and a DOT file (rendering instructions for Graphviz). Both are downstream of a compilation step that turns semantic meaning — "this is a risky dependency" — into visual encoding — a red, dashed line. By the time you're looking at DOT or an image, an AI or graph tool has to reverse-engineer meaning from a color it should never have needed.

RV-KGF is exported **before** that compilation happens, straight from the same worksheet/view/SQL run used to build the diagram. It states what is semantically **true** — labels, relationships, containment, style *names* as references — and deliberately omits what a renderer would **do** — resolved colors, coordinates, layout engine choices. See [`docs/design-rationale.md`](docs/design-rationale.md) for the full reasoning.

The tool now produces three complementary artifacts from one source of truth: the **image** (for a human), the **DOT file** (the instruction set), and the **JSON** (for AI, analysis, and future converters to reason over facts without reverse-engineering them from rendering syntax). None is redundant with the others.

## At a glance

```json
{
  "format": "RV-KGF",
  "version": "1.1",
  "directed": true,
  "source_workbook": "supply-chain.xlsx",
  "view": "Full Map",
  "export_datetime": "2026-03-14T09:30:00",
  "styles": {
    "Flow": { "description": "A physical or informational transfer between two parties.", "type": "edge" }
  },
  "graph": {
    "label": { "value": "Supply Chain Overview", "type": "text" }
  },
  "nodes": [
    { "id": "supplierA", "style": "Vendor", "label": { "value": "Supplier A", "type": "text" } },
    { "id": "warehouse", "style": "Facility", "label": { "value": "Central Warehouse", "type": "text" } }
  ],
  "edges": [
    {
      "id": "supplierA->warehouse",
      "source": "supplierA",
      "target": "warehouse",
      "style": "Flow",
      "label": { "value": "ships components", "type": "text" }
    }
  ]
}
```

## Documentation

| Document | Contents |
|---|---|
| [`docs/schema-reference.md`](docs/schema-reference.md) | Full field-by-field schema: top-level document, `graph`, `nodes[]`, `edges[]`, `properties`, label typing, style lookups |
| [`docs/design-rationale.md`](docs/design-rationale.md) | The philosophy behind the format and the nine key architectural decisions (merge semantics, orphan synthesis, cluster modeling, label typing, furniture filtering, timestamp handling, and more), each with the reasoning and the alternative that was rejected |
| [`docs/industry-comparison.md`](docs/industry-comparison.md) | How RV-KGF was evaluated against JGF, D3/NetworkX node-link, Cytoscape.js, GraphSON, JSON-LD/RDF, PG-JSON, and GQL — and why a custom schema was kept |
| [`schema/rv-kgf.schema.json`](schema/rv-kgf.schema.json) | A JSON Schema (Draft 2020-12) for structural validation of RV-KGF documents |
| [`examples/`](examples/) | Worked, valid example exports covering ordinary nodes/edges, clusters, orphan synthesis, multi-edges, ports, and HTML labels |
| [`CHANGELOG.md`](CHANGELOG.md) | Version history |

## Producing and consuming RV-KGF

- **Producing:** In Excel to Graphviz, use the **Knowledge Graph** button in the ribbon's `Visualize` group, or target `JSON` from the existing `PUBLISH` SQL-pipeline directive. `Publish all views` emits one JSON file per view.
- **Consuming:** RV-KGF is plain JSON — read it with any language's standard JSON parser. It is deliberately **not** a 1:1 fit for any single existing graph-database import format (see the industry comparison), so a thin converter is typically needed to load it into a specific target (e.g., Neo4j/Cypher, RDF triples). Such converters are expected to live as separate, decoupled tools that consume this public schema — see [`docs/design-rationale.md#future-extensibility`](docs/design-rationale.md#future-extensibility).

## Versioning

RV-KGF follows the `"version"` field embedded in every document (currently `"1.1"`). Changes are recorded in [`CHANGELOG.md`](CHANGELOG.md). The format is intentionally conservative about adding fields — see the non-normative "Possible Future Extensions" note in the schema reference for the one currently-reserved-but-unimplemented key (`property_definitions`).

## License

[MIT](LICENSE) — the specification and examples in this repository are free to use, implement, and extend.

## Related

- [Excel to Graphviz](https://exceltographviz.com/) — the reference producer tool
- [jjlong150/ExcelToGraphviz](https://github.com/jjlong150/ExcelToGraphviz) — the tool's source repository
