# RV-KGF vs. Established Graph Serialization Standards

Before settling on a custom schema, RV-KGF was evaluated against every major existing graph-serialization convention with a plausible claim to fit. This document records that comparison and the reasoning for keeping a custom format rather than adopting one wholesale.

**Conclusion: no exact match exists among current standards; RV-KGF remains a custom schema**, deliberately informed by the strongest ideas from each.

## Summary table

| Format | Verdict | Key reason |
|---|---|---|
| [PG-JSON](#pg-json-property-graph-exchange-format) | Closest **formal spec** match | Independently validates orphan/stub-node synthesis; diverges on multi-valued properties and a `labels[]` tag array |
| [Cytoscape.js](#cytoscapejs) | Closest **structural** analog | Shares the compound-node `parent` pattern and a separate style-lookup section, but its style section is for actual rendering — RV-KGF's is deliberately the opposite |
| [JGF (JSON Graph Format)](#jgf-json-graph-format) | Considered, not adopted | General-purpose node-link container; doesn't address compound/cluster nesting or RV-KGF's label-typing and furniture-filtering needs |
| [Node-link / D3 / NetworkX](#node-link--d3--networkx) | Considered, not adopted | Minimal, library-oriented shape; no standard convention for clusters, style references, or typed labels |
| [GraphSON](#graphson) | Considered, not adopted | Tied to Apache TinkerPop's property-graph model and multi-valued-property conventions; heavier than needed for a spreadsheet-sourced export |
| [JSON-LD / RDF](#json-ld--rdf) | **Worst fit** | Triples don't treat edges as first-class objects with their own properties — conflicts with RV-KGF's rich, repeatable, never-merged multi-edges |
| [GQL (ISO/IEC 39075:2024)](#gql-isoiec-390752024) | Not applicable as a serialization target | A query language, not a serialization format — though its underlying property-graph type model is conceptually compatible |

---

## PG-JSON (Property Graph Exchange Format)

PG-JSON is the closest **formal specification** match to RV-KGF: both represent a labeled property graph as nodes and edges with attached key/value data, both treat edges as first-class objects with their own properties (not just an id pair), and both are meant as an interchange format rather than a single tool's internal state.

**Independent validation:** PG-JSON's own robustness guidance recommends creating implicit/stub nodes for edge-referenced-but-undeclared ids — exactly the rule RV-KGF arrived at independently for orphan synthesis (see [Design Rationale §D](design-rationale.md#d-implicitorphan-node-synthesis-is-mandatory)). Two designs converging on the same answer from different starting points is a strong signal the rule reflects a real, general problem in graph interchange rather than an idiosyncratic choice.

**Where they diverge:**
- PG-JSON supports **multi-valued properties** (a single key mapping to a set of values) — RV-KGF's `properties` object does not; each key maps to exactly one scalar value, matching what a single spreadsheet cell can hold.
- PG-JSON uses a `labels[]` tag array to categorize elements (a node can carry multiple labels at once) — RV-KGF uses a single `style` string as a lookup key into `styles{}`, matching the tool's existing one-style-per-row worksheet model rather than a multi-label graph-database convention.

## Cytoscape.js

Cytoscape.js is the closest **structural** analog, for two specific reasons:

1. **Compound nodes** — Cytoscape.js models cluster/group containment via a `parent` pointer on the child node, exactly the flat parent-pointer pattern RV-KGF uses for `cluster` (see [Design Rationale §C](design-rationale.md#c-clusters-are-nodes-with-typecluster)), rather than recursive nesting.
2. **A separate style-lookup section** — Cytoscape.js elements reference named styles defined elsewhere in the document, structurally similar to RV-KGF's `style` field pointing into `styles{}`.

**Where they diverge — and it's the most important divergence in this whole comparison:** Cytoscape.js's style section defines **actual rendering** (colors, shapes, line styles) to be applied at draw time. RV-KGF's `styles{}` is deliberately the opposite: a **semantic description** of what a style name means, with the resolved visual attributes it maps to (colors, dash patterns, etc.) excluded entirely. Cytoscape.js and RV-KGF borrow the same *shape* of indirection for opposite *purposes* — one for rendering, one for meaning.

## JGF (JSON Graph Format)

JGF is a general-purpose, minimal node-link container (`graph.nodes[]` / `graph.edges[]`, each an arbitrary metadata bag). It was considered as a possible base but doesn't natively address several things RV-KGF requires: compound/cluster containment, a style-name-as-semantic-reference convention, typed (`text`/`html`) label objects, or the specific furniture-filtering rules needed to translate a DOT-oriented tool's internal row representation into clean output. Adopting it would mean re-inventing most of RV-KGF's actual content inside JGF's generic metadata bags, with none of the specificity that makes the schema legible to its target consumers.

## Node-link / D3 / NetworkX

The `{"nodes": [...], "links": [...]}` shape popularized by D3.js and NetworkX's JSON export is minimal and library-oriented — designed to be handed straight to a specific visualization or graph-analysis library's constructor. It has no standard convention for compound nodes/clusters, style references, or label typing, and its `links[]` (rather than `edges[]`) naming and `source`/`target` index-vs-id conventions vary between the D3 and NetworkX communities, making "the" node-link format less standardized in practice than it first appears. RV-KGF's `nodes[]`/`edges[]` naming and id-based (not array-index-based) references were chosen partly to avoid this ambiguity.

## GraphSON

GraphSON is Apache TinkerPop's serialization format for its property-graph model (used by Gremlin-compatible graph databases). It's a legitimate, mature property-graph format, but it carries TinkerPop-specific conventions — its own multi-valued-property representation, vertex/edge id-typing metadata, and a format versioning scheme tied to TinkerPop releases — that add real weight without adding value for a schema whose source data is a spreadsheet, not a running graph database. RV-KGF's property value-typing rules (see [Schema Reference §7](schema-reference.md#7-properties-object)) solve the same "what type is this value" problem with a much smaller, spreadsheet-appropriate rule set.

## JSON-LD / RDF

**The worst fit of anything considered.** RDF's triple model (subject–predicate–object) does not treat edges as first-class objects with their own attached properties — a "relationship" in RDF is a predicate connecting two nodes, and attaching rich metadata to that relationship itself (ports, tail/head labels, multiple attributes, and, critically, the ability for the *same* two nodes to have several distinct, never-merged relationships between them) requires reification patterns that add significant complexity RDF wasn't designed to carry naturally. This directly conflicts with a core RV-KGF requirement: edges as first-class, richly-attributed, never-merged objects (see [Design Rationale §B](design-rationale.md#b-edges-never-merge-regardless-of-dots-strict-keyword)).

## GQL (ISO/IEC 39075:2024)

GQL is the first ISO-standardized property-graph **query language** (roughly, "SQL for property graphs"), not a serialization/interchange format — comparing it to RV-KGF is comparing a query language to a data format, so it isn't a candidate to adopt directly. It's included here because its underlying property-graph **type model** (typed nodes and edges, each with their own property sets) is conceptually compatible with RV-KGF's data model, which is a useful signal that RV-KGF's shape isn't an idiosyncratic outlier relative to where the broader property-graph ecosystem has standardized — a future GQL-backed database import from RV-KGF would need a straightforward converter, not a conceptual reconciliation.

## Why keep a custom schema

Across every format compared, either the fit was structurally wrong for a requirement RV-KGF treats as non-negotiable (RDF's non-first-class edges; JGF/node-link's lack of a cluster or style-reference convention), or the fit was closer but paired with unrelated complexity from a different domain (PG-JSON's multi-valued properties and label arrays; GraphSON's TinkerPop-specific typing metadata). No existing format is a closer overall match to Excel to Graphviz's actual source data — a CSS-like styles worksheet, a DOT-shaped cluster/brace structure, and spreadsheet-typed property cells — than a schema built directly around that source data.

RV-KGF is more legible to its actual target consumers — an AI reading it directly, and future bespoke converters written against its specific fields — than a generic format would be, and no external consumer has asked for conformance to an existing standard. The decision was made deliberately, after comparison, not by default: **do not adopt any of these formats wholesale; keep the custom schema**, informed by the strongest applicable idea from each (PG-JSON's orphan-synthesis robustness rule; Cytoscape.js's compound-node containment pattern).
