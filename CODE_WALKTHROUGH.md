# Code walkthrough (Leibniz Lab Challenge)

This document is a file-by-file guide to the assessment codebase: what runs when, what each module owns, and how data moves between disk, PyReason, OpenAI, and the Streamlit UI.

## How to run things

- **Fresh GraphML from YAGO (Hugging Face):** python -m src.buildYagoSubsetFromHuggingFace
- **End-to-end inference (OpenAI + PyReason + JSON/CSV outputs):** python -m src.runInference
- **Delete generated files under src/output/ only:** python -m src.cleanOutputs
- **Dashboard:** ./run_dashboard.sh or streamlit run src/dashboard.py from the repo root
- **Frozen demo artifacts:** set LEIBNIZ_USE_BRUNO_OUTPUT=1 (dashboard reads src/brunoOutput/; the pipeline still writes to src/output/)

---

## src/paths.py

Single place for filesystem layout.

- **Constants:** dataDirectory, writableOutputDirectory, brunoOutput path, and filenames (inferenceResultFileName, etc.).
- **graphmlPath() / rulesPath():** resolved paths under src/data/.
- **artifactDirectory(write=False|True):** when write=True, always src/output. When write=False, src/output unless the demo env flag selects src/brunoOutput.
- **filterShowableRules(rules):** keeps rule dicts where showable is not explicitly false; used by inference, LLM layer, and dashboard so hidden rules never get prompts or UI rows.

---

## src/rule_display.py

Presentation and export metadata for rules.

- **displayTitleForRule:** uses display_title from JSON if present; otherwise derives a spaced title from name.
- **headPredicateFromRuleText:** first predicate in the rule head (before <- or <-1).
- **predicateToWatchableTitles:** maps head predicate strings to one or more display titles (joined with ·).
- **inferredEdgesWatchableList:** turns inferredEdgesByPredicate into list blocks with watchableRuleTitles, ruleNames, and edges for inference_result.json.

---

## src/llmClient.py

All OpenAI traffic for grounding facts.

- **loadRepositoryEnvironment:** loads .env from repo root (and cwd) for OPENAI_API_KEY and options.
- **resolveLlmPayload:** validates key and GraphML, loads showable rules, calls buildPayloadWithOpenAi (one HTTP request per rule), returns (payload, llmGenerationMetadata).
- **buildPromptForRule / supplementalNotesForRuleBody:** prompt text; special case when the rule body mentions citizenOf so the model does not invent citizen edges that PyReason should infer.
- **normalizePyreasonFactText / sanitizePyreasonFactRows:** enforce pred(A,B):[lo,hi] and strip stray }]} glued after the interval (fixes PyReason parse errors).
- **writePyreasonFactFileFromPayload:** writes cleaned rows to llm_facts_pyreason.json and mutates the in-memory payload fact list to match.

---

## src/runInference.py

Orchestrates the full pipeline and writes under src/output only.

1. **ensureGraphmlExists:** existing yago_subset.graphml, or HF stream via buildYagoSubsetFromHuggingFace.
2. **resolveLlmPayload:** LLM facts + NL + transcript paths.
3. **Output writes:** llm_facts_pyreason.json, llm_full_payload.json.
4. **stripRulesForPyreason:** writes temporary rules_for_pyreason_loader.json (PyReason loader only needs syntactic fields).
5. **PyReason steps:** pr.reset, load_graphml, add_rule_from_json, add_fact_from_json, reason.
6. **get_rule_trace:** writes CSV inference_rule_trace_edges.csv.
7. **collectInferredEdges:** last non-empty timestep per head predicate in headPredicateLabels.
8. **inferredEdgesWatchableList:** enrich for UI.
9. **inference_result.json:** bundle with paths, trace sample, and inferred edges.

Helper **ruleTraceEdgeToRecords:** flattens PyReason trace tuples into JSON-serializable dicts.

---

## src/dashboard.py

Read-only Streamlit app; never calls PyReason or OpenAI.

- **loadJson:** safe read of optional artifacts.
- **combinedLayoutGraph:** GraphML to MultiDiGraph with edge_kind kg; merges LLM facts from JSON as edge_kind llm.
- **mergePyreasonInferredIntoCombined:** adds inference_result.json edges as edge_kind inferred with watchable_label.
- **inferNodeEntityKinds:** votes per node from predicates (person/city/country/company/language).
- **graphmlToPyvisHtml:** spring layout, node/edge styling, physics off, returns HTML string for st.components.v1.html.
- **addRuleTitleColumn:** inserts From Rule using showable titles and Fact vs Rule rows.
- **main:** sidebar copy + five tabs: graph, rules list, LLM facts/payload/transcript, inference JSON viewers, trace CSV preview (500 rows).

Repo root is pushed onto sys.path so from src import paths works when Streamlit sets cwd elsewhere.

---

## src/cleanOutputs.py

Deletes a fixed list of filenames under writableOutputDirectory (src/output) so you can rerun inference without stale LLM or trace files. Never deletes src/data/rules.json or yago_subset.graphml.

---

## src/buildYagoSubsetFromHuggingFace.py

Streams the VLyB/YAGO3-10 dataset until required triples appear, maps YAGO relations to PyReason edge attribute names (cityIn, officialLanguage, companyHeadquarteredIn), writes yago_subset.graphml via writeGraphmlFromHuggingFace.

---

## Important Data files

| Location | Role |
|----------|------|
| src/data/rules.json | PyReason rules + display_title, showable |
| src/data/yago_subset.graphml | YAGO subset as PyReason KG |
| src/output/* | Live pipeline outputs |
| src/brunoOutput/* | Checked-in snapshot for demo mode |

---

## JSON keys (external contracts)

PyReason and persisted files use snake_case keys in several places (rule_text, fact_text, infer_edges) on purpose: they must match what PyReason and your notebooks expect. Python locals in this repo trend camelCase (graphPath, showableList, inferencePath) per project preference; dict keys wired to those APIs stay as documented in the assignment.
