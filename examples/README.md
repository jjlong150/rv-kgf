# Examples

Each file here is a complete, valid RV-KGF document that validates against [`../schema/rv-kgf.schema.json`](../schema/rv-kgf.schema.json).

| File | Demonstrates |
|---|---|
| [`context-diagram.json`](context-diagram.json) | A typical system-context diagram: actors, applications, a data store, styled edges, and an edge carrying `properties`. Also demonstrates **style-level `properties`** (`https-post-json` carries `encrypted`/`status` facts distinct from its `description`) coexisting with an edge's own instance-specific `properties` (`sync`). Good starting point for reading the format for the first time. |
| [`handwashing-flowchart.json`](handwashing-flowchart.json) | Two-level cluster nesting (`cluster_2` inside `cluster_1`) via the flat `cluster` parent-pointer, differentiated `taillabel`/`headlabel` on a loop-back edge, and named ports used for real routing (not just decoration). |
| [`stress-test.json`](stress-test.json) | Edge cases: a self-loop, an HTML label with the outer delimiter correctly stripped, three never-merged parallel edges between the same two nodes (`kkk->mmm`) each with distinct ports and labels, and a chain of synthesized `defined: false` orphan nodes. |

To validate any example (or your own RV-KGF export) yourself with Python:

```bash
pip install jsonschema
python3 -c "
import json, jsonschema
schema = json.load(open('../schema/rv-kgf.schema.json'))
doc = json.load(open('context-diagram.json'))
jsonschema.validate(doc, schema)
print('valid')
"
```
