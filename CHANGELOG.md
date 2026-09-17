# Changelog

All notable changes to the RV-KGF specification are documented here. This tracks the format's `"version"` field, not the Excel to Graphviz application's own version number.

## [1.1] - 2026-09-17

- Added optional `properties` object to style definitions in the `styles{}` dictionary, using the same shape and value-typing rules as graph/node/edge `properties`. Lets a style carry organization-specific, policy-dependent facts (e.g. `encrypted`/`status` on a network-call style) separate from its universal, objective `description` — see [Design Rationale §J](docs/design-rationale.md#j-styles-may-carry-their-own-properties-separate-from-description).
- Backward compatible: a 1.0 consumer that ignores unrecognized keys can still parse a 1.1 document correctly, just without seeing the new field.

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
