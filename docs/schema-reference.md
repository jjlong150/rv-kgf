# RV-KGF Schema Reference

Version documented: **1.1**

This document specifies every field in an RV-KGF document. For *why* the schema is shaped this way, see [`design-rationale.md`](design-rationale.md). For a machine-readable version of these rules, see [`../schema/rv-kgf.schema.json`](../schema/rv-kgf.schema.json).

## Conventions used in this document

- Fields marked **required** are always present in a valid export.
- Fields marked **optional** are omitted entirely when they don't apply — RV-KGF does not use `null` or empty objects/strings to mean "not applicable." An empty `properties` object, for instance, is invalid; the whole `properties` key is simply absent.
- `{value, type}` denotes the **label object** shape described in [Label fields](#label-fields), used for every piece of user-authored text in the format.

---

## 1. Top-level document

```json
{
  "format": "RV-KGF",
  "version": "1.1",
  "directed": true,
  "source_workbook": "<Excel workbook file name>",
  "view": "<view name>",
  "export_datetime": "yyyy-mm-ddThh:nn:ss",
  "styles": { "...": "..." },
  "graph": { "...": "..." },
  "nodes": [ "..." ],
  "edges": [ "..." ]
}
```

| Field | Required | Type | Notes |
|---|---|---|---|
| `format` | required | string | Always the literal string `"RV-KGF"`. Lets a consumer that ingests multiple graph formats dispatch on this field before parsing further. |
| `version` | required | string | Schema version of this document, e.g. `"1.0"`. Not the tool's own application version. |
| `directed` | required | boolean | Whether edges are directed. Excel to Graphviz diagrams are conventionally directed graphs; the field is included explicitly rather than assumed, since RV-KGF is meant to be legible standalone. |
| `source_workbook` | required | string | File name of the originating Excel workbook, for traceability back to the source of truth. |
| `view` | required | string | Name of the view (yes/no inclusion toggle set on the `styles` worksheet) that produced this export, e.g. `"Piccadilly Line"` vs. the full map. |
| `export_datetime` | required | string | Local time on the generating machine, **no timezone offset**: `yyyy-mm-ddThh:nn:ss`. See [Design Rationale §H](design-rationale.md#h-export_datetime-timestamp-has-no-timezone-offset) for why no offset is included. |
| `styles` | optional | object | Dictionary of style names actually referenced by this export's nodes/edges/graph, mapping to their documentation. See [Styles dictionary](#2-styles-dictionary). |
| `graph` | optional | object | Metadata about the graph as a whole (0 or 1 present). See [Graph object](#3-graph-object). |
| `nodes` | required | array | Complete, closed set of every node and cluster referenced anywhere in this export — see [Nodes](#4-node-object). May be empty (`[]`) for a graph with no nodes, but the key itself is always present. |
| `edges` | required | array | Every edge, as an array, **never** a dictionary — duplicates are expected and meaningful. See [Edges](#5-edge-object). May be empty. |

---

## 2. Styles dictionary

```json
"styles": {
  "Call Center Agent": { "description": "A human employee who answers telephone calls from customers.", "type": "node" },
  "Flow": { "description": "A physical or informational transfer between two parties.", "type": "edge" },
  "https-post-json": {
    "description": "Makes a web request sending JSON via POST method over HTTPS.",
    "type": "edge",
    "properties": { "encrypted": true, "status": "allowed" }
  }
}
```

- Keyed by style name, exactly as authored on the workbook's `styles` worksheet.
- `description` — free text from the `styles` worksheet's `description` column, documenting what the style/category *means* — the JSON/text equivalent of a visual legend (which is itself excluded from JSON as furniture; see [Design Rationale §G](design-rationale.md#g-furniture-non-semantic-row-filtering)).
- `type` — one of `"node"`, `"edge"`, or `"cluster"`, describing what kind of element the style applies to.
- `properties` — optional. Same shape and value-typing rules as the [Properties object](#7-properties-object) used elsewhere in the format. Lets a style category carry structured, **organization-specific facts** that are true of every element using that style, distinct from what `description` documents and distinct from the style's visual rendering. See [Design Rationale §J](design-rationale.md#j-styles-may-carry-their-own-properties-separate-from-description) for why this is a separate field from `description` rather than folded into it.
- **Scope:** only style names actually **referenced** by a node, edge, or the graph object in *this specific export* are included — not the full worksheet. Purely structural style names (cluster brace markers, legend rows, transparent helper edges) are excluded from the export entirely (see furniture filtering) and so never appear here.
- A style with no authored description simply has no `description` key, and a style with no authored properties simply has no `properties` key (never `"properties": {}`) — or the whole style is omitted from `styles` if never referenced. There is no placeholder value for either field.
- The entire `styles` key is omitted only when no referenced style has a `description` or `properties` to report.

---

## 3. Graph object

```json
"graph": {
  "label": { "value": "...", "type": "text" },
  "debuglabel": { "value": "...", "type": "text" },
  "properties": { "...": "..." }
}
```

Represents the entire graph. **0 or 1** graph object is present per document — Graphviz's top-level graph is the outermost scope; clusters/subgraphs without their own label inherit from a surrounding cluster or, ultimately, this object.

| Field | Required | Notes |
|---|---|---|
| `label` | optional | See [Label fields](#label-fields). |
| `debuglabel` | optional | See [Label fields](#label-fields) and [debuglabel](#debuglabel). |
| `properties` | optional | See [Properties object](#7-properties-object). Omitted entirely if the source properties column was empty. |

---

## 4. Node object

```json
{
  "id": "a",
  "type": "cluster",
  "style": "Extract",
  "label": { "value": "...", "type": "text" },
  "xlabel": { "value": "...", "type": "text" },
  "tooltip": { "value": "...", "type": "text" },
  "debuglabel": { "value": "...", "type": "text" },
  "cluster": "cluster_1",
  "defined": false,
  "properties": { "...": "..." }
}
```

| Field | Required | Notes |
|---|---|---|
| `id` | required | The node's identity. Unique within `nodes[]` (nodes, unlike edges, merge on re-declaration — see rationale §A). |
| `type` | optional | **Omitted entirely for ordinary nodes** — absence means "ordinary node." Present and equal to `"cluster"` only for clusters. Clusters are modeled as nodes with `type: "cluster"`, not a separate array — see [Design Rationale §C](design-rationale.md#c-clusters-are-nodes-with-typecluster). |
| `style` | optional | Lookup key into the top-level `styles{}` dictionary. Never an inlined/resolved visual attribute. Unified field name across nodes and edges (see rationale §F). |
| `label` | optional | See [Label fields](#label-fields). |
| `xlabel` | optional | External/floating label variant, same shape as `label`. |
| `tooltip` | optional | See [Label fields](#label-fields) — handled identically to other labels, including HTML-delimiter stripping, even though tooltips are not rendered as HTML. |
| `debuglabel` | optional | See [debuglabel](#debuglabel). |
| `cluster` | optional | Flat parent-pointer to the immediate containing cluster's `id`. Omitted when the node is not nested. Arbitrary nesting depth is supported by chaining pointers (`cluster_2`'s own node entry points at `cluster_1`), never by recursive JSON nesting. See [Design Rationale §C](design-rationale.md#c-clusters-are-nodes-with-typecluster). |
| `defined` | optional | **Only ever present with value `false`.** Marks a synthesized stand-in for a node referenced by an edge but never declared — see [Design Rationale §D](design-rationale.md#d-implicitorphan-node-synthesis-is-mandatory). A fully-known node has no `defined` field at all (not `true`). |
| `properties` | optional | See [Properties object](#7-properties-object). **Never present on a `defined: false` stand-in** — there is no source row to read properties from. |

---

## 5. Edge object

```json
{
  "id": "a->b",
  "source": "a",
  "target": "b",
  "tailport": "n",
  "headport": "s",
  "style": "Flow",
  "label": { "value": "...", "type": "text" },
  "xlabel": { "value": "...", "type": "text" },
  "taillabel": { "value": "...", "type": "text" },
  "headlabel": { "value": "...", "type": "text" },
  "debuglabel": { "value": "...", "type": "text" },
  "tooltip": { "value": "...", "type": "text" },
  "properties": { "...": "..." }
}
```

| Field | Required | Notes |
|---|---|---|
| `id` | optional | Human-readable identifier, conventionally `"<source>-><target>"`. **Not required to be unique** — duplicate `id`s across array entries are expected and correct under multi-edges (see below). |
| `source` | required | `id` of the tail-end node. Guaranteed to resolve to an entry in `nodes[]` (synthesizing a `defined: false` stand-in if necessary). |
| `target` | required | `id` of the head-end node. Same resolution guarantee as `source`. |
| `tailport` / `headport` | optional | Named port on the source/target node, for edges that connect to a specific compass point or record field rather than the node as a whole. |
| `style` | optional | Lookup key into `styles{}`. Same semantics as on nodes. |
| `label` | optional | Edge's main label. See [Label fields](#label-fields). |
| `xlabel` | optional | External/floating label variant. |
| `taillabel` / `headlabel` | optional | Labels positioned near the tail/head end of the edge specifically (distinct from the edge's overall `label`). |
| `debuglabel` | optional | See [debuglabel](#debuglabel). |
| `tooltip` | optional | See [Label fields](#label-fields). |
| `properties` | optional | See [Properties object](#7-properties-object). |

**Edges is always an array, never a dictionary keyed by `"source->target"`.** A dictionary keying scheme silently overwrites a second edge between the same two nodes if it carries different attributes — a real defect found and corrected during this format's design. RV-KGF always emits one array entry per authored edge, unconditionally, **regardless of Graphviz's `strict` keyword**, because `strict` is a rendering-only decluttering setting that can discard attribute data when collapsing parallel edges, and RV-KGF's entire purpose is preserving every authored fact pre-compilation. See [Design Rationale §B](design-rationale.md#b-edges-never-merge-regardless-of-dots-strict-keyword).

---

## 6. Label fields

Every piece of user-authored text in RV-KGF — `label`, `xlabel`, `taillabel`, `headlabel`, `tooltip`, and `debuglabel`, wherever they appear on the graph, a node, or an edge — uses the same two-field shape:

```json
{ "value": "...", "type": "text" }
```

or

```json
{ "value": "<b>Apple</b> Pie", "type": "html" }
```

| Field | Notes |
|---|---|
| `value` | The label content. See type-specific normalization below. |
| `type` | Either `"text"` or `"html"`. Detected from the tool's existing **Format Column** — the same signal already used by the DOT builder to decide between `<...>` (HTML-like label) and `"..."` (quoted text) syntax. |

**`type: "text"` normalization:** literal two-character Graphviz justification codes (`\l`, `\n`, `\r` — the literal backslash-letter sequence, not real control characters) are collapsed to a space; any run of whitespace (including real embedded line breaks used to force multi-line tooltips) is then collapsed to a single space; the result is trimmed.

**`type: "html"` handling:** the markup (tables, paragraphs, bold, nesting, etc.) is **not** decomposed or interpreted — passed through raw, since deciding how to flatten arbitrary HTML is a use-case-dependent judgment call the exporter should not make on the consumer's behalf. Exactly the **one outer `<...>` delimiter pair** — Graphviz's own HTML-label wrapper — is stripped before the value is stored; a value must never retain that wrapper (e.g., `<b>Apple</b> Pie`, never `<<b>Apple</b> Pie>`).

### `tooltip`

Handled identically to other labels — including HTML-delimiter stripping and text normalization — even though tooltips are not rendered as HTML by the SVG viewer. Graphviz still applies the same outer-delimiter-stripping and quoting rules to tooltip values as to any other string attribute, so treating `tooltip` as a label field keeps one normalization/inheritance code path instead of two.

### `debuglabel`

Captures a diagnostic string comparable to — though not necessarily identical in format to — what Excel to Graphviz's debug-mode toggle appends to rendered node/edge labels (source-workbook row numbers, for traceability). RV-KGF deliberately leaves the *real* `label` field untouched by debug information, since debug annotations aren't authored content, but preserves the traceability value separately in `debuglabel`.

### Default-attribute inheritance (`node[]` / `edge[]` / `graph[]`)

Graphviz's shorthand for setting default attributes on all subsequently-declared nodes/edges within a scope (e.g. `node[label="Unknown"]`) is resolved into final values in the exported `label`/`xlabel`/`taillabel`/`headlabel` family — never for visual attributes like `color`. This is genuine authored intent, just expressed at a category level rather than per-instance. Resolution respects Graphviz's actual scoping rules: defaults apply within the current subgraph/cluster and are inherited by nested subgraphs, but closing a cluster reverts active defaults back to the parent scope's state — a default set inside one cluster must never leak into a later sibling or outer scope. See [Design Rationale §I](design-rationale.md#i-default-attribute-graphviz-nodeedgegraph-scoping).

---

## 7. Properties object

```json
"properties": {
  "population": 4300000,
  "domestic": true,
  "growth_rate": -0.012,
  "region_name": "Detroit, MI",
  "founded": "1701-07-24"
}
```

Holds the non-Graphviz, user-defined attribute pairs from a row's properties column (e.g. `weight=200 domestic=true costperunit=25.45`), and can appear on the graph, a node, or an edge.

> **The key names are not part of the schema.** `properties` is an open, user-defined bag — its keys are whatever text the workbook author typed into that row's properties column, and differ per node/edge/graph. Every key name shown in this document (`population`, `domestic`, etc.) is illustrative only. A consumer must not assume any particular key is present and must tolerate arbitrary, previously-unseen keys.

- **Real JSON types, not label objects.** Unlike every other attribute in RV-KGF, `properties` values are emitted as native JSON numbers/booleans/strings — not wrapped in a `{value, type}` envelope — so a consumer can read a numeric property as a number without further parsing.
- The `properties` key is **omitted entirely** when the source column was blank or produced no pairs. `"properties": {}` never appears in a valid export.
- Key order is not significant and may vary between exports.
- A property with no value at all in the source text (a bare `note=` or `note`) is dropped during parsing and never appears.

### Property value type rules

| Source column text (key names arbitrary) | JSON representation | Notes |
|---|---|---|
| `someKey=200` | `"someKey": 200` | Unquoted integer within the 32-bit signed range → JSON number, no decimal point. |
| `someKey=25.45` | `"someKey": 25.45` | Unquoted decimal or scientific notation → JSON number. |
| `someKey=true` / `someKey=false` | `"someKey": true` | Case-insensitive `true`/`false`, unquoted → JSON boolean. |
| `someKey=2019-03-14` | `"someKey": "2019-03-14"` | Unquoted ISO-8601 date (`yyyy-mm-dd`) or datetime (`yyyy-mm-dd hh:mm[:ss]`, `T` or space separator) → JSON **string**, verbatim. JSON has no native date type; no timezone conversion or reformatting is applied. |
| `someKey="Detroit, MI"` | `"someKey": "Detroit, MI"` | Double-quoting in the source column is an explicit "keep as text" override — the value becomes a string even if it would otherwise parse as a number, boolean, or date (`someKey="200"` stays the string `"200"`, not the number `200`). |
| `someKey="She said ""hi"" back"` | `"someKey": "She said \"hi\" back"` | A doubled quote (`""`) inside a quoted source value decodes to one literal `"`, then is re-encoded per normal JSON string escaping. |
| anything else | JSON string | Fallback for all other text. |

Only the single, unambiguous ISO-8601 shape above is recognized for dates; other date-like text (e.g. `3/14/2019`) is locale-ambiguous and is exported as a plain string rather than guessed at.

---

## 8. Furniture: what never appears in RV-KGF output

Certain rows in the tool's internal representation exist purely to drive DOT rendering and are excluded from every export:

- **Structural brace tokens** (`{` / `}`) that mark cluster begin/end — consumed only to drive cluster-nesting logic, never emitted as content.
- **Legend rows** (`legend node` / `legend native`) — pure human-visual furniture.
- **Page border begin/end** — a non-semantic top-level frame; omitting it is behaviorally identical to including and then ignoring it.
- **Edge-level `cluster` attribute** (recording that an edge statement was textually written inside a cluster's braces, for layout/rank purposes) — this reflects DOT source placement for layout, not a semantic fact; the edge's `source`/`target` already fully describes what it connects.
- **Rendering/layout attributes** (`layout=fdp`, `splines=ortho`, resolved colors, coordinates, etc.) — excluded by design; see [Design Rationale](design-rationale.md).

Note rows and edges between them — including layout-hack edges styled as invisible/transparent for spacing purposes — are **kept, not excluded**: the knowledge lives in the label text, and even a purely-cosmetic edge conveys real sequencing or context that a consumer might use, despite its original DOT purpose being layout only.

---

## Non-normative: possible future extensions

*The following describes no field that exists in version 1.0. It is a placeholder note for contributors, not a specification.*

`styles` documents the meaning of style names in use across a graph. A natural counterpart would be a `property_definitions` object (a sibling of `styles` at the top level) documenting the meaning of `properties` keys in use — but the producing tool does not currently collect or publish this information, and the right shape for such an object is an open question (for example: whether a property's meaning is global or can vary by node vs. edge vs. graph scope, and what metadata beyond a description — type, units, allowed values — would be useful). Until that is designed against a real authoring workflow, **the key name `property_definitions` is reserved** and should not be reused for an unrelated purpose in derivative or future schemas.

Similarly, a `render` object (a sibling of `graph`/`nodes`/`edges` at the top level) has been discussed as the eventual home for rendering/layout attributes that some consumer might genuinely want (`layout=fdp`, `splines=ortho`, etc.) — clearly separated from the identity/semantic fields described above. This is not built and not currently needed; it is noted here so a future implementer does not reach for those top-level attribute names for something else.
