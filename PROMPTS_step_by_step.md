# libontology → pyoxigraph: short prompts for Qwen / MiniMax (OpenCode)

Use **one prompt per session**. Run the tests yourself after each one, and commit when they're green.
If a model breaks tests or wanders off, `git checkout .` and start the step again; don't argue with it.

---

## Step 0: no model needed (do it yourself)

```bash
cd tools/libontology
git checkout -b pyoxigraph
# unzip libontology-pyox-seed.zip and copy libontology/, tests/, bench/ into tools/libontology
uv add pyoxigraph reasonable==0.4.4
uv run python -m unittest tests.test_ontology_reasoning tests.test_ontology_parity
```
All 16 tests must pass before you go on.

---

## Prompt 1: inventory (read-only)

```
Read every file in tools/libontology/libontology/ and tools/libontology/tests/.
Do NOT modify any file.
Write docs/libontology_migration/00_inventory.md with a table: one row per module,
columns: classes/functions, CLI command, every rdflib method it uses (parse, serialize,
query, update, namespaces, bind, isomorphic, ...), and whether it calls owlrl.
Then list the OWL constructs used in ontology/dseco-latest.ttl
(subClassOf, inverseOf, propertyChainAxiom, FunctionalProperty, unionOf, ...).
```

---

## Prompt 2: base class

```
Rewrite tools/libontology/libontology/base_ontology.py so BaseOntology loads the
ontology into a pyoxigraph.Store (self.store) instead of an rdflib Graph.
Keep: the same class name, the same method names, read_text_file, the error messages
and their colors, the Orange license header, and the numpydoc docstrings style.
Keep the prefixes of the input file (we need them when saving turtle).
Look at libontology/ontology_reasoning.py: it is the style to follow.
Do not touch any other file. Do not modify any test.
```

---

## Prompt 3: update + bulk_update (the slow part)

```
Port libontology/ontology_update.py and libontology/ontology_bulk_update.py to pyoxigraph.
Replace owlrl.DeductiveClosure(...).expand(self.graph) by
OntologyReasoner().materialize(self.store) from libontology/ontology_reasoning.py.
Keep exactly the same order: infer, then run the UPDATE query, then save.
Keep the same class names, method names, CLI arguments, messages, docstrings style
and license header. Save turtle with the original prefixes.
Do not modify the tests. When done, run:
uv run python -m unittest tests.test_ontology_update tests.test_ontology_bulk_update tests.test_ontology_parity
and fix your code (never the tests) until they pass.
```

---

## Prompt 4…n: one module at a time

Repeat this prompt for each module, in this order: `ontology_query`, `ontology_ask`, `ontology_describe`, `ontology_stats`, `ontology_convert`, `ontology_compare`, `ontology_smart_diff`, `ontology_validate`, `ontology_pull`.

```
Port libontology/<MODULE>.py to pyoxigraph, using self.store from BaseOntology.
If it calls owlrl, replace it with OntologyReasoner().materialize(self.store).
Keep the same class/function names, CLI arguments, output format, messages,
numpydoc docstrings and license header, exactly like the old file.
Only modify <MODULE>.py. Do not modify the tests. Then run
uv run python -m unittest tests.test_<MODULE>
and fix your code (never the tests) until it passes.
```

---

## Leave these as they are (rdflib needed)

- `ontology_shacl.py`: pyshacl needs rdflib. Only convert the store to an rdflib Graph at the start, if needed.
- `ontology_infer.py`: keep pellet / hermit (owlready2). Add `reasonable` as a 4th choice in `REASONER_CLASSES` only if Didier wants it.
- `ontology_add_swrl_rule.py`: owlready2, don't touch.

---

## Final check (yourself)

```bash
uv run python -m unittest discover -s tests
uv run python bench/bench_update_loop.py --ontology <real build>.ttl --queries-dir <queries_update dir>
LIBONTOLOGY_PARITY_KG=<real build>.ttl uv run python -m unittest tests.test_ontology_parity
```
