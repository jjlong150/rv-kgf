# Contributing to RV-KGF

RV-KGF is the JSON export schema for [Excel to Graphviz](https://exceltographviz.com/). This repository documents the format; the format itself is produced by that tool's VBA codebase, so most schema changes originate there.

## Reporting an issue with the spec

If you find output that doesn't match what's documented here, or documentation that doesn't match real tool output, please open an issue with:

- The relevant field(s) and the section of [`docs/schema-reference.md`](docs/schema-reference.md) or [`docs/design-rationale.md`](docs/design-rationale.md) involved.
- A minimal example JSON snippet (redact any sensitive workbook content).
- Whether you believe the *documentation* or the *tool's actual output* is wrong — these have different fixes.

## Proposing a schema change

Before proposing a new field or a change to existing behavior, please check:

1. **[`docs/design-rationale.md`](docs/design-rationale.md)** — many things that look like omissions (no resolved colors, no coordinates, no timezone offset, no recursive cluster nesting) are deliberate, documented decisions, not gaps. If your proposal contradicts one of these, please address the stated rationale directly rather than re-proposing the rejected alternative.
2. **The non-normative "Possible Future Extensions" note** at the end of [`docs/schema-reference.md`](docs/schema-reference.md) — some ideas (like `property_definitions`) are already identified as open, reserved-but-undesigned extension points. If your proposal is in this space, it's welcome, but expect a design discussion about scope (per-node vs. per-edge vs. per-graph typing, etc.) rather than a quick merge.
3. **[`docs/industry-comparison.md`](docs/industry-comparison.md)** — if your proposal is "just adopt format X instead," please read why that specific format was already evaluated and set aside.

Schema changes that add new optional fields should:
- Preserve backward compatibility — existing consumers should still be able to parse a document with new optional fields added, ignoring what they don't recognize.
- Bump `version` per semantic versioning norms for the format (a purely additive, optional field is a minor version bump; a change to the meaning or shape of an existing field is a major version bump).
- Come with an update to [`schema/rv-kgf.schema.json`](schema/rv-kgf.schema.json) and at least one example in [`examples/`](examples/) demonstrating the new field.
- Be recorded in [`CHANGELOG.md`](CHANGELOG.md).

## Out of scope for this repository

Converter tools that consume RV-KGF and emit another format (Cypher/Neo4j, RDF, etc.) are welcome as separate, independent projects that depend on this schema — they are intentionally not part of this repository or the Excel to Graphviz tool itself. Feel free to link your converter from this README via a pull request once it exists.
