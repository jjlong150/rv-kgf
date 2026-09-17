# RV-KGF Design Rationale

This document explains *why* RV-KGF is shaped the way it is. Every rule in [`schema-reference.md`](schema-reference.md) traces back to one of the decisions below. Where a decision reflects a real defect that was found and corrected during design, that's called out explicitly — it's evidence the rule matters in practice, not just in theory.

## 1. Starting question and scope

Excel to Graphviz is a ten-year-old Excel/VBA tool that turns a data worksheet plus a CSS-like styles worksheet into a Graphviz DOT file and a rendered diagram. The starting question for this format was: *what can be built to extend this tool for the age of AI, without duplicating something a generic tool already does?*

Several ideas were considered and rejected using a consistent filter — **"could Excel's own Copilot/Claude-for-Excel, Power Query, or a generic tool already do this?"** If yes, it was out of scope:

- Chat-with-your-diagram as a built-in feature — Excel AI plugins already do this.
- Text-to-graph authoring (paste messy text, AI extracts rows) — a generic capability, not specific to this tool's data.
- Graph critique/QA on raw data — works on any spreadsheet, not Graphviz-specific.
- A standalone SQL-to-JSON tool — Excel/Power Query/Copilot already do generic SQL→JSON.
- An MCP server exposing the workbook — too much surface area for unconfirmed demand.
- Image-to-graph reverse engineering, a layout-engine AI advisor, structural diff/changelog narration, an accessibility alt-text generator — each discussed and set aside for small or uncertain value relative to effort.

## 2. The idea that survived: export before compilation, not after

DOT is a *compiled, lossy* representation. The pipeline is:

```
data worksheet → styles worksheet (CSS-like name→visual-attribute lookup) → DOT → rendered image
```

By the time a diagram reaches DOT, semantic meaning has already been compiled into visual encoding — "this is a risky dependency" becomes a red, dashed line. Critically, the compiled `Style Name` itself is usually **already human-readable text** (e.g. `"gets from via REST over HTTPS"` vs. `"gets from via CGI over HTTP"`) — meaning is present *before* compilation, not lost, just about to be thrown away in favor of a color. Handing an AI plain DOT means asking it to reverse-engineer meaning from a color it never should have had to derive meaning from in the first place.

**Resulting core philosophy:** export data **before** style/DOT compilation. RV-KGF's job is to state what is semantically **true**, never what a renderer should **do**. Resolved colors, coordinates, layout-engine choices, and rendering hints are excluded on principle; labels, tooltips, relationships, and style *names* (as semantic references, not resolved attributes) are included on the same principle. Every decision below is an application of this one rule.

## 3. Scope vs. quality

A real concern was raised early on: shipping something half-baked risks a community "black eye," and there's no second chance at a first impression for a ten-year-old tool's reputation. The resolution was to separate two different axes:

- **Scope** — how much functionality to build.
- **Quality** — how well the built scope actually works.

The verdict was to build the **full, correct version of a deliberately smaller scope**: ship the validated JSON schema as a `Publish` format, launched quietly (documented, not headlined), rather than either rushing a rough version across a wide scope or delaying indefinitely waiting for a bigger one.

## 4. Three complementary artifacts, one source of truth

The tool produces three artifacts from the same worksheet, the same SQL run, the same moment:

- The **rendered image** — for a human to look at.
- The **DOT file** — the instruction set, for re-rendering.
- The **JSON (RV-KGF)** — for AI, analysis, and future converters to reason over facts without reverse-engineering them from rendering syntax.

None is redundant with the others; each serves a distinct consumer.

---

## Key architectural decisions

### A. Nodes merge on re-declaration, last-value-wins

Graphviz itself only ever recognizes one node per name — re-declaration overwrites attributes in place, it does not create a second node. RV-KGF mirrors this exactly, because the format's job is to stay faithful to what actually renders (and would actually exist as one entity in a graph database). A node object in `nodes[]` therefore represents the final, merged state of everything ever declared under that id.

### B. Edges never merge, regardless of DOT's `strict` keyword

Graphviz's `strict` keyword is a rendering-only decluttering setting that can silently discard attribute data when collapsing parallel edges into one visual line. Since RV-KGF's entire purpose is to preserve every authored fact pre-compilation, it must diverge from `strict` behavior: **always** emit one array entry per edge, unconditionally. This is, notably, *less* code to implement than replicating strict-merge would be — not a tradeoff against effort, just the correct behavior.

**Concrete defect this fixed:** an early prototype keyed edges in a dictionary by `"source->target"`. A second `AppA->AppB` edge with different attributes silently overwrote the first, losing data. This was the single most important structural correction made during the format's design — `edges` was converted from a dictionary to an array specifically to make loss like this structurally impossible.

### C. Clusters are nodes with `type: "cluster"`

Clusters need the identical shape (label, tooltip, parent pointer) as ordinary content nodes, so they're modeled as nodes with `type: "cluster"` rather than a separate top-level array. This matches Cytoscape.js's compound-node precedent and DOT's own flat-namespace model, where a cluster is just a subgraph that happens to be drawn as a box.

Containment uses a **flat parent-pointer** (`cluster: "cluster_1"`) rather than recursive JSON nesting, because arbitrary-endpoint edges require flat global node identity to work at all — an edge from a deeply-nested node to a top-level node can't be expressed cleanly if nodes only exist inside their parent's nested structure. This is the same reasoning GraphML, Neo4j, and Cytoscape all apply in avoiding recursive containment for graph data.

Cluster ids **reuse the tool's own internal unique subgraph-name counter** — a mechanism that already exists to satisfy Graphviz's requirement that every subgraph have a unique name. Cluster ids are never derived from label text, since labels can legitimately repeat (two different clusters can both be labeled "Extract," for instance) while ids cannot.

### D. Implicit/orphan node synthesis is mandatory

DOT allows `a -> b` to implicitly create nodes `a` and `b` if they were never explicitly declared. RV-KGF must not replicate this silently, because the entire value of `nodes[]` is that a consumer can trust it as a **complete, closed set** without separately scanning every edge for endpoints that might not appear there.

**Rule:** every edge endpoint must resolve to a `nodes[]` entry. If no real declaration exists, a stand-in is synthesized: `{"id": "x", "defined": false}` — with **no fabricated label**. Setting `label` to the id itself would be replicating a Graphviz rendering fallback, not preserving authored content, and would blur the line between "this is what the user wrote" and "this is what the renderer guessed." If a real declaration for the same id appears elsewhere in the source, it merges into the stub (per rule A), and the `defined` field disappears entirely — the node is no longer distinguishable from one that was always fully declared.

The tool already has orphan-detection logic, originally built for an existing "filter orphans from the diagram" feature. The fix here is architectural, not new detection: the DOT/rendering path keeps its existing filtering behavior unchanged (orphan filtering only affects what's *drawn*), but the JSON path must **always** retain orphans regardless of that filter setting, tagged `defined: false` — because suppressing them from the JSON would silently delete real relationship facts, not just clean up a picture.

**Independent validation:** the Property Graph Exchange Format (PG-JSON) — the closest formal spec to RV-KGF — recommends this exact same implicit-node-creation behavior in its own robustness guidance, arrived at independently. See [`industry-comparison.md`](industry-comparison.md).

### E. Label typing — uniform `{value, type}` shape

`label`, `xlabel`, `taillabel`, `headlabel`, `debuglabel`, and `tooltip` all use the same `{value, type: "text"|"html"}` shape wherever they appear (graph, node, or edge).

- **`text`** values are run through a normalize step: literal two-character Graphviz justification codes (`\l`, `\n`, `\r`) collapse to a space, then any run of whitespace — including real embedded line breaks used to force multi-line tooltips — collapses to one space, then the result is trimmed.
- **`tooltip`** is handled the same way as other labels, even though SVG does not interpret HTML for tooltips. Graphviz still removes the outermost `< >` pair from a tooltip written in the HTML-like alternate grammar and applies the same quoting rules as any other string attribute, so treating tooltips as labels keeps one normalization/inheritance code path instead of two, at no cost in correctness.
- **`debuglabel`** captures a string comparable to — though not necessarily identical in format to — what the tool's debug-mode toggle appends to rendered labels (source-workbook row numbers, for traceability). RV-KGF prefers to leave the real `label` untouched by debug information but preserves the traceability value separately, since it's genuinely useful for tracking an element back to its source row.
- **`html`** values are deliberately **not** decomposed or interpreted (tables, paragraphs, bold, nesting are passed through raw) — full decomposition into structured data is a genuine, use-case-dependent judgment call the exporter should not make on a consumer's behalf. Exactly the **one outer `<...>` delimiter pair** (Graphviz's own HTML-label wrapper) is stripped.

  **Bug found and fixed during design:** an early prototype failed to strip this delimiter, storing `"<<b>Apple</b> Pie>"` instead of `"<b>Apple</b> Pie"`. Verified via `html.parser` that the un-stripped version breaks standalone parseability — the outer literal `<`/`>` leaks into extracted text as if it were content. The fix strips exactly one outer pair, no more, no less.
- **Text vs. HTML detection** in production reads the tool's existing **Format Column** — already referenced in the base tool's Graph Options/SQL documentation alongside the Style Column and Object-Type Column — the same signal the DOT builder already uses to decide between `<...>` and `"..."` syntax. Naive angle-bracket string-sniffing is explicitly rejected as a production technique (it was only ever a conversation-demo stand-in).

### F. Unified `style` field name

Nodes and edges both use `style` as a lookup key into the top-level `styles{}` dictionary (an early prototype inconsistently used `style` on nodes and `style_name` on edges — unified during design). The value is always a name to be looked up, never an inlined or duplicated visual attribute — this mirrors the tool's existing CSS-like styles-worksheet architecture, where a style name is defined once and referenced by many rows.

`styles{}` includes only entries actually **referenced** by a node, edge, or the graph object in that specific export — not the entire worksheet. Purely structural style names (cluster brace markers, legend rows, transparent helper edges) correctly receive no description automatically, since they never survive furniture filtering to be referenced in the first place — no special-casing is needed.

### G. Furniture/non-semantic row filtering

Validated against a real "context diagram" example converted from the tool's public GitHub example repository.

- The tool's raw internal row export already encodes DOT brace-tokens as pseudo-rows (e.g. `{"item": "{", "styleName": "Outer Grouping Begin"}`) — a replay log of the exact push/pop the DOT builder already performs when it enters and leaves a cluster. RV-KGF's cluster-nesting logic **reuses this same traversal/stack** rather than building new logic — it just re-points emission at JSON `cluster` parent-pointers instead of literal DOT braces.
- Node vs. edge detection is free: a row with a `relatedItem` value is an edge; a row without one is a node.
- Rows fall into three categories: **(1) structural tokens** (`{`/`}`) — consumed only to drive the stack, never emitted; **(2) real content** — the entire point of the export; **(3) furniture** — excluded outright.
- `legend node` / `legend native` rows are excluded entirely — pure human-visual furniture with no semantic content.
- `Page Border Begin/End` is treated as a non-semantic frame — pushed/popped as a no-op, with no cluster node created for it, since it wraps everything at the top level and omitting it is behaviorally identical to including and then ignoring it.
- **Note rows, and edges between them — even purely-decorative "Transparent Edge" layout hacks — are kept, not excluded.** This was a deliberate pushback during design against an instinct to filter them out: the knowledge lives in the Note's label text, and even a layout-hack edge conveys real sequencing or context an AI consumer might use, even though its original DOT *purpose* was purely a rendering spacing trick.
- An edge's `cluster` attribute (recording that the edge statement was textually written inside a cluster's braces, for rank/layout purposes) is **excluded** — this reflects DOT source placement for layout, not a semantic fact; the edge's `source`/`target` already fully describes what it connects, independent of where in the file it happened to be declared.
- Rendering/layout attributes (`layout=fdp`, `splines=ortho`, etc.) are excluded by default, for the same reason resolved colors and coordinates are excluded. If ever needed, these belong in an optional sibling object (e.g. `"render": {...}`) at the top level, clearly separated from identity and semantic fields — not built, not currently needed.

### H. `export_datetime` timestamp has no timezone offset

`export_datetime` is plain local ISO 8601 with no offset: `Format$(Now, "yyyy-mm-dd\Thh:nn:ss")`. Two alternatives were explored and rejected:

1. The Windows `kernel32` API for a proper offset — rejected as Mac-incompatible, and the tool must run on both platforms.
2. A cross-platform shell-out via PowerShell or `MacScript`/AppleScript — rejected as "the juice isn't worth the squeeze": slow, carries a Mac permission-prompt risk, and corporate security tooling may flag VBA code spawning `powershell.exe` as suspicious behavior.

**Final decision:** no offset. The field is documented as local time on the generating machine, unspecified time zone — a deliberate, documented limitation rather than an oversight.

### I. Default-attribute (`node[]`/`edge[]`/`graph[]`) scoping

Graphviz's shorthand for setting default attributes on subsequently-declared elements within a scope (e.g. `node[label="Unknown"]`) is currently passed straight through to DOT and resolved by Graphviz at render time. RV-KGF instead resolves these into final values, but **only for content fields** — the `label`/`xlabel`/`taillabel`/`headlabel` family — and never for visual attributes like `color`. Unlike the id-as-label rendering fallback (never resolved — see §D), a `node[label=...]` statement is genuine authored intent, just expressed at a category level rather than per-instance, and belongs in the export.

Graphviz's own documentation confirms these defaults are **scoped to the current subgraph/cluster**, not global for the rest of the file: a subgraph inherits attributes from its parent, and defaults apply "within the same (sub-)graph." Within one scope, the most recently set default wins, with no way to unset it mid-scope. But **closing a cluster must revert active defaults to the parent scope's prior state** — a flat, non-scoped dictionary would incorrectly let an inner cluster's default leak into a later sibling or outer-scope element that never should have inherited it.

This scoping requirement is met with a **stack of scopes** — the same stack structure already needed for cluster-parent tracking, unified into one data structure rather than kept as two structures that would need to be manually synchronized.

### J. Styles may carry their own `properties`, separate from `description`

`description` documents what a style **means**, objectively, in a way that holds regardless of who's reading the diagram.

For example, a style named `https-post-json` can represent making a web request sending JSON via POST over HTTPS. But two organizations can look at that same objective fact and reach opposite conclusions about it. One permits HTTPS-only internal traffic and disallows plaintext HTTP entirely; another, migrating a legacy system, currently allows HTTP with a remediation deadline. Neither organization's policy changes what the style *means*, it changes how that meaning is **evaluated**, and that evaluation is exactly the kind of subjective, deployment-specific fact that has always lived in the DOT-level rendering, not in the workbook — green for `https-post-json`, red for `http-post-json` — because a color was the only place available to put it.

That's a structural problem for the entire premise of RV-KGF: an AI consumer asked "does this design comply with corporate standards?" or "evaluate the security of this design" has no way to answer without access to the color a specific organization's Graphviz theme happened to assign. This ambiguity is the exact kind of rendering-layer reverse-engineering RV-KGF exists to eliminate (see [§2, the core philosophy](#2-the-idea-that-survived-export-before-compilation-not-after)).

The fix is a style-level `properties` object, using the same open, typed key/value shape already defined for graph/node/edge `properties` (see [Schema Reference §7](schema-reference.md#7-properties-object)) — not a new mechanism, just the existing one made available in a new place. `https-post-json` keeps one universal `description`; each organization's own `styles` worksheet then attaches its own `properties`, e.g. `encrypted=true status="allowed"` at one company, `encrypted=true status="allowed, pending TLS 1.3 upgrade"` at another. The description never has to change to reflect a policy difference, and the policy fact becomes a real, queryable, typed value instead of a color that only means something to whoever built the legend.

These subjective values are deliberately **not** folded into `description` as more prose, for the same reason node/edge `properties` were never folded into `label` as more prose: free text is for a human to read, and a typed key/value pair is for a machine to filter, aggregate, and reason over. 

Keeping `properties` structurally identical across graph, node, edge, *and now style* means a consumer needs exactly one parsing rule for "how do I read a properties object," not a special case for styles.

## Future extensibility

Standalone converter modules — for example, a Cypher/Neo4j exporter, or an RDF/JSON-LD converter — that consume RV-KGF as input and emit another graph format as output are explicitly **future, user-built work, not part of the internal roadmap**. They are meant to be decoupled tools fed only by this public schema, never touching the producing tool's internals. This keeps RV-KGF's own scope bounded to "export the facts correctly" and leaves format-specific conversion to tools purpose-built for each target ecosystem.
