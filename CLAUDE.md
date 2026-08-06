# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single hands-on training lab, not a library or app: `notebooks/rag_confluence_failure_diagnosis.ipynb`.
It builds a RAG pipeline over a dummy Confluence wiki, deliberately reproduces a realistic retrieval
failure (a stale runbook page crowding out its current replacement), then uses Arize Phoenix's tracing
and eval tooling to catch and diagnose it, with results logged back onto the actual trace as annotations
(visible in the Phoenix UI, not just printed). Trainees fix the failure themselves in two guided
exercises. There is no application code outside the notebook — `data/confluence_seed_pages.md` (seed
content for the dummy Confluence space) and `docs/screenshots/*.png` (real Phoenix UI screenshots,
embedded in both the notebook and README) are the only other non-trivial files.

Runs fully locally by default (Ollama on Apple Silicon) so trainees need zero cloud API keys, including
for the eval judge. Cloud (OpenAI) mode is a documented alternate path for Colab / PR-reviewer readers.

## Commands

```bash
uv sync                                                          # install/update deps from pyproject.toml
uv run jupyter notebook --notebook-dir=. notebooks/rag_confluence_failure_diagnosis.ipynb   # open the lab
uv add <package>                                                 # add a new dependency (updates pyproject.toml + uv.lock)

# Non-interactive full-notebook execution (used for verifying edits, not for normal lab use):
uv run jupyter nbconvert --to notebook --execute \
    notebooks/rag_confluence_failure_diagnosis.ipynb \
    --output rag_confluence_failure_diagnosis.executed.ipynb
rm notebooks/rag_confluence_failure_diagnosis.executed.ipynb     # always clean up the executed copy after inspecting it — never commit it
```

**`--notebook-dir=.` is required, not cosmetic.** Passing a notebook file path to `jupyter notebook`
roots its file server on that file's own directory (`notebooks/`), not your shell's cwd. The
notebook's screenshot links (`../docs/screenshots/...`) point one level above that root, so
without this flag Jupyter refuses to serve them and they render as broken images even though the
files are present on disk.

There is no separate test suite, lint config, or build step — the notebook's own end-to-end execution
*is* the test. Requires `.env` in the repo root (gitignored) with `CONFLUENCE_URL`,
`CONFLUENCE_USERNAME`, `CONFLUENCE_API_TOKEN`, `CONFLUENCE_SPACE_KEY`, and (cloud mode only)
`OPENAI_API_KEY`. Requires a running local Ollama daemon with `qwen2.5`, `nomic-embed-text`, and
`gemma2:9b` pulled for local mode (the default).

## Editing the notebook

Prefer editing the `.ipynb` via a small `nbformat` script over the `NotebookEdit` tool for anything
touching more than one cell — bulk/multi-cell edits via `NotebookEdit` have proven unreliable in this
repo. Pattern used throughout: read the notebook with `nbformat.read(...)`, look up cells by their
stable `cell.id` (not index, which shifts), mutate `.source`, `nbformat.write(...)` back. Write these
scripts to the scratchpad, not into the repo.

After any change to code cells, re-run the full notebook via `nbconvert --execute` and check actual
cell *outputs* (dataframe row counts, annotation IDs, printed labels) — **a clean nbconvert exit does
not mean the change worked.** The tracing bug below shipped for a long time with zero errors and only
an empty dataframe as the symptom.

## Architecture / notebook flow

15 numbered sections, config-driven by two toggles in the "Configuration" cell:

- `LOCAL_MODE` (`True`/`False`) — switches the generation LLM, embedding model, *and* eval judge
  between Ollama (`qwen2.5` / `nomic-embed-text` / `gemma2:9b`) and OpenAI
  (`gpt-4o-mini` / `text-embedding-3-small` / `gpt-4o-mini`). The judge is deliberately a **different**
  model from the generator in local mode (`gemma2:9b` vs `qwen2.5`) — using the same model to
  judge its own output is self-preference bias, one of the reliability problems this lab exists to
  demonstrate. **This only holds in `LOCAL_MODE`** — in cloud mode `gpt-4o-mini` plays both roles,
  a documented compromise of the alternate path (see section 2's markdown), not an oversight. Keep
  this distinction accurate in any future edit; don't let "always different" language creep back
  in for cloud mode.
- Confluence credentials, loaded from `.env` via `load_dotenv()`, with interactive prompt fallback.

Pipeline shape: `ConfluenceReader` (Basic Auth: `user_name=`/`password=`, **not** `api_token=` — the
latter is Bearer auth, meant for Server/Data Center PATs and returns 403 against Confluence Cloud) →
title-normalization (strips literal `#`/`##`/emoji artifacts from hand-pasted Markdown) → filter to
`EXPECTED_TITLES` (drops Confluence's auto-generated homepage/template pages) → `VectorStoreIndex` at
`similarity_top_k=1` (deliberately narrow — `top_k=2` lets the model see both the deprecated and
current page and self-correct, hiding the failure).

**Section 3 ("Warm-up: meet the building blocks") is where LlamaIndex, Phoenix tracing, and
Phoenix evals each get demonstrated once on a throwaway 3-document warm-up corpus (`warmup_docs`),
before section 4 touches real Confluence data.** `tracer`, `client`, `relevance_evaluator`, and
`faithfulness_evaluator` are all built exactly once, in section 3 — every later section reuses
these same objects, it never reinstantiates them. Do not add a second `register()`/`.instrument()`
call or a second evaluator instantiation anywhere past section 3; either would be redundant at
best and may error at worst (OpenInference instrumentors are not designed to be attached twice).
LlamaIndex also ships an older one-line Phoenix integration
(`llama_index.core.set_global_handler("arize_phoenix")`, from
`llama-index-callbacks-arize-phoenix`) — deliberately not used here, since it doesn't expose the
manual trace_id/span control `run_traced_query`/`find_retriever_span` depend on; don't "simplify"
tracing back to it.

**Tracing: `phoenix.otel.register()` is load-bearing, not optional.** `px.launch_app()` alone does
*not* wire up a global `TracerProvider` — spans silently never export (confirmed empirically: 0 rows
from `get_spans_dataframe()` without `register()`, real rows with it). The instrumentation cell must
call `register(project_name=..., auto_instrument=False)` and pass the resulting `tracer_provider`
explicitly into `LlamaIndexInstrumentor().instrument(...)`.

**Three layers, easy to conflate: OpenTelemetry, OpenInference, Phoenix.** OpenTelemetry is the
vendor-neutral tracing standard (`TracerProvider`, spans, trace/span IDs) — nothing about it is
LLM-specific, and `run_traced_query`'s `tracer.start_as_current_span()`/`span.get_span_context()`
calls are plain OTel API, portable to any OTel backend. OpenInference is Arize's semantic-
convention layer on top of OTel, specific to LLM/RAG spans — it defines what attributes like
`retrieval.documents.0.document.content` mean, and is what `LlamaIndexInstrumentor` actually emits.
Phoenix is the backend/UI consuming both, wired up via `phoenix.otel.register()`'s OTLP exporter.
Section 3.2 in the notebook teaches this stack explicitly (a trainee-facing addition, not just an
implementation note) — keep it accurate in any future edit to the tracing setup.

**Trace correlation uses manual spans, not "most recent row."** `run_traced_query(query_engine,
query_str, span_name)` wraps each trigger query in a manually-started OTel span and returns
`(response, trace_id, wrapper_span_id)`, so later cells can look up *this query's* spans by
`trace_id` instead of assuming the last row in a dataframe belongs to the query that just ran.
`find_retriever_span(client, trace_id)` then filters Phoenix's `RETRIEVER`-kind spans for the one that
actually carries `retrieval.documents.*` attributes — **a single LlamaIndex retrieval produces two
`RETRIEVER` spans** (`VectorIndexRetriever.retrieve` and child `._retrieve`), and only the outer one has
the attributes. Any span-selection logic must filter on attribute presence, not just `span_kind` or
span count.

**Eval results are logged back onto spans as Phoenix annotations, not just printed** — this is the
part that actually demonstrates Phoenix as an observability tool rather than `phoenix.evals` as a
bare scoring library. `log_retrieval_annotations_to_phoenix(...)` is the one shared helper (called from
both exercises) that writes `document_relevance` and `deprecation_check` annotations per retrieved
document via `client.spans.add_document_annotation(...)`, and a `faithfulness` annotation on the
wrapper span via `client.spans.add_span_annotation(...)`. Both calls use `sync=True` explicitly (the
API defaults to `False`) so a trainee alt-tabbing to the Phoenix UI sees the annotation immediately
instead of hitting a confusing "it's not there yet."

**`deprecation_check` is written successfully but never rendered anywhere in the Phoenix UI —
confirmed empirically (v19.15–19.18), not a guess.** Phoenix's trace-detail Info tab only builds a
dedicated "Retrieval Metrics" widget (ndcg/precision/hit) for the `document_relevance` annotation
name specifically; a custom annotation name is stored (the write succeeds, no error) but has no
generic display anywhere in the trace view, the spans table, or the project-level annotation
summary. `/v1/document_annotations` is POST-only in the REST API, and the Python client's
`get_span_annotations`/`get_span_annotations_dataframe` do not return document-level annotations
either — there is currently no read path for them at all outside GraphQL introspection. The
notebook's section 8 and README's walkthrough step 8 say this explicitly now; don't let a future
edit reintroduce a claim that `deprecation_check` is UI-visible.

**The fixed response (Exercise 1) is now also traced and annotated, not just printed.** It used to
call `fixed_query_engine.query(...)` directly with no `run_traced_query` wrapper, so Phoenix had no
trace_id to look it up by and `log_retrieval_annotations_to_phoenix` was never called for it — the
"Before/after comparison" section's Phoenix-visible contrast (needed for `retriever_fixed.png`,
see below) silently didn't exist. Fixed by wrapping it in `run_traced_query` like the naive query,
and adding a cell right after the comparison table that evaluates and logs its annotations too.
That fix also caught a real bug: `DocumentRelevanceEvaluator.evaluate()` scores exactly one
document per call (see its docstring — "Returns one Score"), but `fixed_query_engine` retrieves 2
documents (`similarity_top_k=2`); the naive path never needed a loop since `top_k=1` always
retrieves exactly one document. The fixed-response cell now calls `.evaluate()` once per retrieved
document and joins their text for the faithfulness context. If `log_retrieval_annotations_to_phoenix`
is ever called with a multi-document response elsewhere, check for this same shape mismatch.

**`docs/screenshots/*.png` are real captures from one live run, not mockups** — generated via a
scratchpad Node script driving headless Chrome (Puppeteer, reused from an existing global install
rather than adding a new project dependency) against a real `px.launch_app()` instance populated by
running the actual pipeline. If seed data, model output, or the Phoenix UI's own layout changes
enough to make these stale, they need regenerating the same way; there is no committed script for
this in the repo (kept as a one-off, like every other notebook-editing script in this project).

The lab's actual conclusion deliberately rests on a **deterministic** check
(`deterministic_deprecation_check`, a plain metadata-set intersection), not the LLM judges. The
judge model itself went through a reliability check before being picked: the first candidate,
`mistral-nemo:12b`, was measurably inconsistent run-to-run on `FaithfulnessEvaluator` specifically
(confirmed empirically over 10 repeated trials per failure case: solid on one case, 8/10 "faithful"
vs. 2/10 "unfaithful" on the other, identical input each time). `gemma2:9b` was tested the same way
and came back fully consistent (10/10) on both cases, so it replaced `mistral-nemo:12b` as the
default judge. That history, not just the general LLM-judge-reliability literature, is why the lab
frames judge output as something to cross-check rather than trust outright — a judge that looks
consistent in one test still isn't ground truth.

Datasets & Experiments (`client.datasets.create_dataset`, `client.experiments.run_experiment`) are
Phoenix's idiomatic side-by-side comparison feature but are **intentionally left as an optional
take-home pointer** in the stretch section, not built into the required lab — the existing pandas
before/after table is the required comparison mechanism.

`DEPRECATED_TITLES` (the hand-set Python set driving `deterministic_deprecation_check` and the
Exercise 1/2 metadata-filter fix) is a deliberate demo simplification, not a claim that real wikis
come with a "deprecated" flag — the comment above it says so directly. Section 13, a stretch
section, tests that simplification against real Confluence signals: `ConfluenceReader` already
puts Confluence's own `status` field into `d.metadata["status"]` (before this notebook overwrites
it), and it will almost always read `"current"` for both the old and new page in a pair, since the
old page was never formally archived — that's the same root cause as the opening story, not a
Confluence limitation. The section then pulls a real signal instead, `version.when` (last-modified
timestamp) and page labels, via `reader.confluence` (the `atlassian-python-api` client
`ConfluenceReader` wraps internally) — pseudocode only, matching the dual-judge stretch section's
style, since it needs a live Confluence space with real version history to run meaningfully.

## Narrative framing

The notebook's intro (title cell, then dedicated "The Story" and "The Tools" cells) and the
README both open with a short, deliberately anonymized story based on a real reported case of a
support chatbot serving a stale policy page that contradicted itself, with the company losing the
resulting dispute — the same failure shape this lab teaches (old page crowds out current page,
judge misses it, deterministic check catches it). No company name, person's name, court name, or
citation appears in either version — keep it that way in any future edit; the point is the failure
mechanism, not the specific case.

The intro used to be one large markdown cell; it's now six separate cells (Title, The Story, The
Tools, Architecture, Prerequisites, Known limitations) so each topic is independently readable and
navigable in Jupyter, before the numbered "1. Setup & install" flow starts. Keep new intro content
in its own cell rather than growing one of these back into a multi-topic wall of text.

## Content-engineering notes (why the seed pages look the way they do)

Reproducing the retrieval failure reliably required tuning `data/confluence_seed_pages.md`, not just
code:
- The deprecated/legacy page must be longer and more keyword-dense than its current replacement, or
  the current page wins retrieval and there's no failure to diagnose.
- Any "⚠️ DEPRECATED" banner text in a deprecated page's body must be removed — the generation LLM
  reads it and self-corrects to the right answer even when only the deprecated page was retrieved,
  which defeats the exercise.

## Writing style for prose

These rules govern prose only: docs, PR text, messages, README, cookbooks. Never touch code or
technical terms — swap in everyday words only where precision survives. Review every prose output
against these rules before delivering.

0. Never touch code or technical terms; swap in everyday words only where precision survives.
1. Never use a metaphor, simile, or other figure of speech you are used to seeing in print.
2. Never use a long word where a short one will do.
3. If it is possible to cut a word out, always cut it out.
4. Never use the passive where you can use the active.
5. Never use a foreign phrase, a scientific word, or a jargon word if you can think of an everyday
   English equivalent.
6. Break any of these rules sooner than say anything outright barbarous.
