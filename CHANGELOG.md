# Changelog

All notable changes to the RV-KGF specification are documented here. This tracks the format's `"version"` field, not the Excel to Graphviz application's own version number.

## [1.0] - Initial publication

- Initial publication of the RV-KGF schema: top-level document (`format`, `version`, `directed`, `source_workbook`, `view`, `export_datetime`, `styles`, `graph`, `nodes[]`, `edges[]`).
- Nodes merge on re-declaration (last-value-wins); edges never merge and are always emitted as an array.
- Clusters modeled as nodes with `type: "cluster"` and a flat `cluster` parent-pointer.
- Mandatory synthesis of `defined: false` stand-in nodes for edge-referenced-but-undeclared endpoints.
- Uniform `{value, type: "text"|"html"}` label object applied to `label`, `xlabel`, `taillabel`, `headlabel`, `tooltip`, and `debuglabel`.
- Open, typed `properties` object with documented value-coercion rules (integer, decimal, boolean, ISO-8601 date/datetime, quoted-string override).
- `styles{}` dictionary of referenced style names with prose `description` and `type`.
- Furniture-filtering rules (legend rows, page border frame, structural brace tokens, edge-level layout `cluster` attribute excluded; Note rows and layout-hack edges retained).
- `property_definitions` reserved as a future top-level key; not implemented in this version.
- JSON Schema (Draft 2020-12) added for structural validation.
