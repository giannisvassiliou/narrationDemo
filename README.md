# InsideTheGraph? Inspecting Evidence in Knowledge-Graph-Grounded Narratives

Demo accompanying the WWW 2027 demo paper *InsideTheGraph? Inspecting Evidence in Knowledge-Graph-Grounded Narratives* (Vassiliou, Troullinou, Trouli, Batsakis, Papadakis, Kondylakis — ECE-HMU, FORTH-ICS, CSD-UOC).

When a language model turns a knowledge graph into text, it may add claims that the supplied triples do not support. Such claims can sound convincing and may even be true, which makes them hard to notice. InsideTheGraph? lets you:

1. **Choose the evidence**: pick a seed entity, extract its k-hop neighbourhood from DBpedia or Wikidata, and select a budgeted subgraph.
2. **Compare narratives**: generate narratives from the same triples under different prompts (graph-only, plain, closed-book) and view them side by side.
3. **Trace claims to evidence**: click a claim to see the triples and inference rules behind it, or click an edge to see which claims use it.
4. **Inspect and correct the auditor**: override a label and add a note. The original automatic decision is kept.

Unsupported claims are also looked up in the wider graph. If evidence turns up there, the claim is reported as having candidate evidence outside the input, but it stays **unsupported relative to the input**: support in the wider graph is not grounding, and a failed lookup does not show that a claim is false.

- Live prototype: http://139.91.210.46:8000/
- Video: *(add link)*

---

## Quick start

Requires Python 3.10+ and an Anthropic API key.

```bash
pip install -r requirements.txt

# Put your Anthropic API key (and nothing else) in a file, e.g. ./key
echo "sk-ant-..." > key && chmod 600 key

# Local use
python3 app.py --key-file key                      # http://127.0.0.1:5000

# Serve on all interfaces, port 8000, in the background (how the public prototype runs)
nohup python3 app.py --key-file key --port 8000 --host 0.0.0.0 &
```

| Option | Default | Meaning |
|---|---|---|
| `--key-file PATH` | none | File containing only the Anthropic API key (recommended) |
| `--api-key KEY` | none | Key on the command line (ends up in your shell history) |
| `--host` | `127.0.0.1` | Interface to bind; `0.0.0.0` exposes the server on the network |
| `--port` | `5000` (or `$PORT`) | HTTP port |

The key is looked up in this order: `--api-key` / `--key-file`, `ANTHROPIC_API_KEY`, the file named in `KGN_API_KEY_FILE`, then `anthropic_key.txt` in the project folder.

> **Security.** Never commit or share your key file. `.gitignore` covers `anthropic_key.txt` and `*.key`; if you name the file `key`, add `key` to `.gitignore` too. With `--host 0.0.0.0`, anyone who can reach the port can start runs that are billed to your key. The built-in Flask server is for demos only; put it behind a reverse proxy (HTTPS, rate limiting) for anything longer-lived.

### Without an API key or network

```bash
KGN_OFFLINE=1 KGN_MOCK_LLM=1 python3 app.py
```

This mode uses a hand-made DBpedia-style fixture (`fixtures/Albert_Einstein.json`) and fake model output. It is for UI work only. Prepared examples in `data/gallery/` load without a key.

### Tests

```bash
python3 -m pytest -q
```

---

## How it works

```
seed entity ──► k-hop neighbourhood N_k(s) ──► selector σ_{b,c} ──► input S
                    (DBpedia / Wikidata)          (budget b, relation cap c)
                                                        │
              ┌─────────────────────────────────────────┤
              ▼                                         ▼
      Writer (prompt π, S) ──► narrative      Reasoner: D = Cl_R(S ∪ Σ) \ (S ∪ Σ)
              │                                         │
              └──────────► Verifier: split into claims, align with S ∪ D, label
                                         │
                       unsupported claims ──► wider-graph lookup in G
                                         │
                         Interface: inspect, trace, correct
```

### 1. Extraction and selection

- **DBpedia**: `dbo:` predicates, minus Wikipedia bookkeeping.
- **Wikidata**: truthy `wdt:` statements of item, time, quantity, string and monolingual-text types. External identifiers, URLs and media are excluded, as is incoming authorship (P50). Statement qualifiers are **not** retrieved.
- Seeds are English Wikipedia titles for both sources, so the same seed list works for both KGs.
- Hop 1 adds up to 150 incoming edges, spread across the 25 most frequent relations and preferring notable neighbours. Deeper hops follow outgoing edges and expand at most 40 nodes per hop.

**Selectors** (exploration controls; the paper does not propose a new selection algorithm):

| id | Selector |
|---|---|
| `random` | Random |
| `degree` | Degree (hubs first) |
| `ppr` | Personalized PageRank top-k |
| `connected-ppr` | Connected growth with PageRank prizes (default) |
| `wiki2vec`, `connected-wiki2vec` | Wikipedia2Vec similarity (optional; needs `KGN_WIKI2VEC`) |

The selection holds at most `budget` triples (default 25), with at most `per_relation` triples sharing a relation on the same entity (default 3; 0 disables the cap). If the cap leaves too few admissible triples, it is relaxed so that every selector returns the requested number.

### 2. Generation conditions

| Mode | Writer receives | Purpose |
|---|---|---|
| `graph_only` | triples + instruction to assert nothing beyond them | main condition |
| `plain` | the same triples, no instruction | isolates the effect of the instruction |
| `closed_book` | no triples | contrast only; its lack of support is expected |

The model, the triples and the requested length stay fixed; only the prompt changes. Changing the selector changes the evidence instead, which answers a different question.

### 3. Rule reasoner (`kgnarr/reasoner.py`)

Before verification, the reasoner computes a closure over the selected triples plus schema facts. **Derived triples go to the verifier only, never to the writer.** Each derived triple stores its rule and premises, so it can be traced back to the input.

- RDFS: subproperty (rdfs7), domain (rdfs2), range (rdfs3), subclass (rdfs9). DBpedia provides `rdfs:subPropertyOf/domain/range/subClassOf` from its ontology. Wikidata provides P1647 (subproperty) and P279 (subclass, two levels above P31 classes). Wikidata has no RDFS domain/range, so those two rules fire only on DBpedia.
- Containment (a domain rule, not RDFS): place-of relations (born in, died in, …) carry up containment chains (country, state, P131, …). Containment followed by containment gives a generic *located in*.

This is a restricted, configured rule set, not complete RDFS reasoning.

### 4. Audit labels

The interface uses more detailed labels than the paper. The mapping:

| Paper | Interface label (code id) | Meaning |
|---|---|---|
| `dir` | Stated in graph (`supported`) | One supplied triple supports the complete claim |
| `rule` | Derived by rules (`derived`) | A derived triple with a recorded proof supports it |
| `judg` | Judged to follow (`entailed`) | A combination of evidence is accepted by model judgement; no single triple suffices |
| `unsup` + *external match* | Leaked from memory (`leakage`) | Not in the input; candidate evidence found in the wider KG |
| `unsup` + *no match* | Not found in KG (`not_in_kg`) | Not in the input; the bounded wider-KG search found nothing |
| `unsup` + *unresolved* | Unsupported, unchecked (`unchecked`) | Not in the input; the lookup could not run or timed out |
| connective text | Narrative glue (`glue`) | No independently checkable assertion; excluded from shares |

A valid match must support the **complete** claim, including dates, quantities and superlatives. Sharing an entity is not enough, and triple IDs cited by the writer count only as candidates. If the wider-KG lookup finds a fact that was in fact in the input or derived from it, the verifier missed an alignment, and the label is corrected.

The **wider-graph lookup** resolves a claim's subject and object, then searches for a direct edge, a two-edge path in one direction, or a normalised literal match. A semantic check then assesses predicate, direction, value and temporal scope. Facts expressed in other ways are reported as *not found*.

### 5. Summary measures

Each narrative reports evidence-backed share `(n_dir + n_rule)/N`, model-judged share, external-match share, unsupported share, and triple coverage (the fraction of input triples used by direct or rule-derived claims). These describe the auditor's decisions; they are **not** accuracy scores.

---

## Preparing a demo session

Live runs take tens of seconds (SPARQL queries plus model calls). Prepared examples load instantly, so precompute them before a session:

```bash
python3 precompute.py --key-file key --seeds seeds.txt --preset prompts
python3 precompute.py --key-file key --seeds seeds.txt --preset memory
python3 precompute.py --key-file key --seeds seeds.txt --preset selectors --source wikidata
```

| Preset | Setting A | Setting B |
|---|---|---|
| `prompts` | graph-only | plain, same subgraph (the paper's main comparison) |
| `memory` | graph-only | closed-book, same subgraph |
| `selectors` | PageRank top-k, graph-only | connected growth, graph-only |

Other options: `--source {dbpedia,wikidata}`, `--selector`, `--hops` (2), `--budget` (25), `--sentences` (8), `--per-relation` (3), `--force` (recompute existing examples).

Examples are written to `data/gallery/`. Every SPARQL result is cached in `data/sparql-cache/`, so a prepared seed can be rebuilt offline. **Copy the whole `data/` folder to the demo laptop.**

---

## Annotation and agreement

Annotators save labels in the right-hand panel. Every reviewed claim should be saved, including ones where the annotator agrees, because a saved unchanged label counts as agreement. Labels are appended to `data/annotations.jsonl`.

```bash
python3 agreement.py                # automatic vs human: Cohen's kappa, confusion matrix, per-label P/R
python3 agreement.py --coarse       # grounded / glue / unsupported
python3 agreement.py --json report.json
```

When two annotators label the same claims, the script also reports inter-annotator kappa.

---

## Initial annotation study

File: `evaluation/InsideTheGraph_Annotation_Evaluation.xlsx`. Sheets: README, Results, Review, Annotations, Claims, Evidence. All results are formula-linked to the source labels.

**Setup.** 100 claims from 10 entities (Alan Turing, CERN, Chania, ETH Zurich, Knossos, Lise Meitner, Mona Lisa, Nikos Kazantzakis, University of Crete, Venus de Milo), 10 claims each. Each claim comes with its supplied triples (T…) and derived triples (D…). Six models labelled every claim independently with `dir / rule / judg / unsup`, following the annotation instructions in the workbook (600 annotations in total).

**Reference.** One human reviewer labelled 11 claims explicitly. For the remaining 89, the reviewer accepted the model majority label. Reference distribution: 70 `dir`, 1 `rule`, 21 `judg`, 8 `unsup`.

| Model | Label agreement | Macro-F1 | Unsup. recall | Unsup. precision |
|---|---|---|---|---|
| GPT-6.1 Sol | 0.94 | 0.915 | 1.00 (8/8) | 0.615 |
| GPT-6 Astra | 0.95 | 0.923 | 1.00 (8/8) | 0.615 |
| GPT-6.1 Terra | 0.92 | 0.829 | 1.00 (8/8) | 0.667 |
| Gemini Hydrogen | 0.94 | 0.864 | 1.00 (8/8) | 0.889 |
| Claude Haiku 5.5 | 0.93 | 0.848 | 1.00 (8/8) | 0.800 |
| Claude Opus 4.8 | 0.94 | 0.933 | 1.00 (8/8) | 0.800 |

All six models agree on 88% of claims. Pairwise agreement between models ranges from 0.91 to 0.99.

**Limitations of this study:**
- The reference is model-assisted: one reviewer accepted or revised model labels. It is not independent multi-annotator ground truth, and no inter-human agreement is available.
- The class counts are small (8 `unsup`, 1 `rule`), so recall and macro-F1 change noticeably with a single decision.
- The study scores the supplied annotation runs. It does **not** score the predictions of the auditor deployed in the demo (`KGN_VERIFIER_MODEL`).
- The Haiku and Opus outputs are identical on 99 of 100 rows, and it has not been confirmed that they were separate runs.
- Sampling seed, generation settings and prompt condition per claim were not recorded.

---

## Configuration

| Variable | Default | Meaning |
|---|---|---|
| `KGN_MODEL` | `claude-sonnet-5` | Narrative writer |
| `KGN_VERIFIER_MODEL` | same as writer | Claim extraction and alignment; a different model family avoids self-preference |
| `KGN_TEMPERATURE` | unset | Left out by default; if a model rejects it, the call is retried without it |
| `KGN_DBPEDIA_ENDPOINT` / `KGN_WIKIDATA_ENDPOINT` | public endpoints | SPARQL endpoints |
| `KGN_ORACLE_TIMEOUT` / `_WORKERS` / `_BUDGET` | 20 / 6 / 90 | Wider-graph lookup: seconds per query, parallel lookups, total seconds per narrative. Claims still pending when the budget runs out are labelled *unchecked* |
| `KGN_USER_AGENT` | generic | Sent with SPARQL requests; Wikimedia expects contact details, e.g. `inside-the-graph/1.0 (you@hmu.gr)` |
| `KGN_WIKI2VEC` | unset | Wikipedia2Vec model file; enables the wiki2vec selectors (`pip install wikipedia2vec numpy`) |
| `KGN_PER_NODE` / `KGN_EXPAND_CAP` / `KGN_IN_LIMIT` / `KGN_MAX_TRIPLES` | 60 / 40 / 150 / 3000 | Neighbourhood extraction limits |
| `KGN_DATA` | `./data` | Caches, runs, gallery, annotations |
| `KGN_OFFLINE` / `KGN_MOCK_LLM` | off | Fixture data / fake model output |

---

## HTTP API

| Method | Path | Body / result |
|---|---|---|
| GET | `/api/config` | sources, selectors, modes, labels, models |
| POST | `/api/subgraph` | `{source, seed, hops, selector, budget, per_relation}` → selection |
| POST | `/api/narrate` | `{selection_id, mode, sentences}` → narrative + audit |
| POST | `/api/annotate` | `{run_id, claim_index, label, note, annotator}` |
| GET | `/api/gallery`, `/api/gallery/<id>` | prepared examples |

---

## Repository layout

```
app.py              Flask server
precompute.py       builds prepared examples (data/gallery/)
agreement.py        human vs automatic label agreement
seeds.txt           seed entities (English Wikipedia titles)
kgnarr/
  sources/          DBpedia and Wikidata extraction
  selectors.py      subgraph selectors
  generate.py       writer prompts (graph-only, plain, closed-book)
  reasoner.py       rule closure with derivation records
  verify.py         claim extraction, alignment, labels, wider-graph lookup
  pipeline.py       end-to-end runs, gallery, annotations
static/             browser interface (vis-network for the graph)
fixtures/           offline fixture and DBpedia ontology
data/               SPARQL cache, selections, runs, gallery, annotations
evaluation/         annotation study workbook
tests/              smoke tests
```

## Known limitations

- Claim extraction and alignment are done by a model, so `dir`, `rule` and `judg` labels are model judgements. Only the derivations themselves are deterministic.
- Derivations depend on the configured rules and predicate mappings.
- Wikidata qualifiers are omitted, which limits temporal and contextual checks.
- The wider-graph lookup is bounded (direct edge, two-edge path, literal match), so it can miss evidence.
- Wikidata incoming edges are capped without ordering, so candidate sets for hub entities depend on endpoint order (cached after the first run).
- Audit labels describe support by the supplied evidence, not truth.

## Data and ethics

The demo uses public KG data (DBpedia, Wikidata) and keeps their attribution and licence terms. Visitors are asked not to enter sensitive information. Visitor input may be used for research only with informed consent and the applicable review.

## Citation

```bibtex
@inproceedings{vassiliou2027insidethegraph,
  title     = {InsideTheGraph? Inspecting Evidence in Knowledge-Graph-Grounded Narratives},
  author    = {Vassiliou, Giannis and Troullinou, Georgia and Trouli, Georgia Eirini and
               Batsakis, Sotirios and Papadakis, Nikolaos and Kondylakis, Haridimos},
  booktitle = {Companion Proceedings of the ACM Web Conference 2027 (WWW '27)},
  year      = {2027},
  address   = {Dublin, Ireland}
}
```
