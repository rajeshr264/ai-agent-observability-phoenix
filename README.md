# ai-agent-observability-phoenix

Sample cookbooks built with Arize Phoenix, an open source tool for watching LLM apps.

## RAG Debugging Lab: Confluence Retrieval-Failure Diagnosis

`notebooks/rag_confluence_failure_diagnosis.ipynb` is a hands-on lab. You do the work; you don't
just read it. It builds a RAG pipeline on a dummy Confluence wiki, then causes a real retrieval
failure on purpose: an old runbook page, never archived, crowds out its current replacement.
Phoenix's tracing and eval tools then catch and diagnose the failure. The results go back onto the
trace as annotations, so you see them in the Phoenix UI, not just as printed text. Trainees fix the
failure in two guided exercises: one fully worked, one done alone on a second case.

By default the lab runs fully on your own machine, using Ollama on Apple Silicon. You need no
cloud API keys, not even for the eval judge. This makes it good workshop material: trainees can run
the whole lab on their own MacBooks. A cloud mode using OpenAI is also here, written up for Colab
and for PR reviewers, since Colab cannot keep an Ollama daemon running.

### Why this lab exists

A real customer once asked a real company's support chatbot about a bereavement discount. The
bot said he could apply for it after he flew, within 90 days of the fare. He booked, he flew, he
filed the claim. The company said no.

Here is the part that matters: the bot's own answer linked to the policy page it was quoting.
That page said the opposite — apply before you travel, not after. The bot's answer and its own
source contradicted each other, in the same reply, on the same site.

The company argued the bot was a separate legal entity, answerable for its own words. A tribunal
disagreed and ruled for the customer.

This was not a hallucination. The bot did not invent a policy. It retrieved real text and served
the wrong version of it, with full confidence. An old policy page and its current replacement sat
side by side in the same knowledge base, and the bot picked the old one.

This lab reproduces that exact failure — an old page that crowds out its current replacement — on
a dummy Confluence wiki, so you can catch it, trace it, and fix it yourself.

### Architecture

Here is the flow, from wiki page to judged answer:

```
 CONFLUENCE (dummy "Meridian Analytics" wiki, ENG space, 19 pages)
      │
      │  ConfluenceReader (Basic Auth: user_name + password/API token)
      ▼
 CLEAN + FILTER  (strip stray "#"/emoji from titles, keep only EXPECTED_TITLES)
      │
      │  embed  (Ollama nomic-embed-text  or  OpenAI text-embedding-3-small)
      ▼
 VECTORSTOREINDEX  (similarity_top_k=1 — narrow on purpose, so a stale
                     page can beat its replacement and the failure shows up)
      │
      │
      ▼                                    ┌─ run_traced_query() wraps every
 TRIGGER QUERY  ──────────────────────────► │  step below in one OTel span,
 e.g. "What is the current procedure          so all spans share one trace_id
  for failing over our primary database?"   └─
      │
      ▼
 RETRIEVER  → picks the top-scoring page (may be the stale one)
      │
      ▼
 GENERATION LLM  (Ollama qwen2.5  or  OpenAI gpt-4o-mini)  → writes the answer
      │
      ▼
 RESPONSE
      │
      ├──────────────► DocumentRelevanceEvaluator  (LLM judge: gemma2:9b
      │                 scores the retrieved page                or gpt-4o-mini)
      │
      ├──────────────► FaithfulnessEvaluator       (same LLM judge)
      │                 scores the answer against the retrieved page
      │
      └──────────────► deterministic_deprecation_check  (plain code, no LLM —
                        checks the retrieved page's title against a known list
                        of stale pages; this is the check the lab trusts most)
      │
      ▼
 add_document_annotation() / add_span_annotation()  (sync=True)
      writes every judge score and the code check back onto the trace
      │
      ▼
 PHOENIX UI  — open the trace and see the query, the retrieved page, the
               answer, and every score, all in one place
```

The generation model and the judge model are never the same model in local
mode (`qwen2.5` writes, `gemma2:9b` judges) — a model should not grade
its own work.

### Prerequisites

**For local mode (the default, no API keys):**
- [Ollama](https://ollama.com), installed and running
- Pull these models: `ollama pull qwen2.5`, `ollama pull nomic-embed-text`, and
  `ollama pull gemma2:9b`. Use `gemma2:9b` as the judge — a different model from the one that
  writes the answer, so it never judges its own work. It replaced an earlier judge,
  `mistral-nemo:12b`, after a side-by-side reliability test found it steadier (see "Known limit"
  below).
- 16GB or more of unified memory (M2/M3 class Apple Silicon)
- A free Atlassian Cloud site (it bundles Jira and Confluence, free for up to 10 users, no card
  needed — sign up at [atlassian.com](https://www.atlassian.com)). Fill a Confluence space with the
  dummy "Meridian Analytics" wiki pages the notebook expects, and make a Confluence API token at
  `id.atlassian.com/manage-profile/security/api-tokens`.

**For cloud mode (an alternate path, for example Colab):**
- An `OPENAI_API_KEY`. It covers both the writer and the judge in this mode.

**Known limit — we do not hide it:** a local judge is less steady than a cloud judge, and we
tested that before picking one. The first local judge we tried, `mistral-nemo:12b`, gave different
`FaithfulnessEvaluator` verdicts on the same input, run after run: 10 for 10 "faithful" on one
failure case, but 8 "faithful" and 2 "unfaithful" on the other. We ran the same test on `gemma2:9b`,
the judge this lab uses now, and it came back fully steady — 10 for 10 — on both cases. That is not
proof `gemma2:9b` is always right: a March 2026 RAND study found no LLM judge holds up across the
board, so we still do not treat its score as truth. It is why the lab's real verdict rests on the
deterministic check, not on the judge.

### Setup

Make a `.env` file in the repo root (it is already in `.gitignore`). Put your Confluence keys in
it — `CONFLUENCE_URL`, `CONFLUENCE_USERNAME`, `CONFLUENCE_API_TOKEN`, `CONFLUENCE_SPACE_KEY` —
and, for cloud mode, `OPENAI_API_KEY`. Then run:

```bash
uv sync
uv run jupyter notebook notebooks/rag_confluence_failure_diagnosis.ipynb
```

In the notebook's config cell, set `LOCAL_MODE = True` (the default) or `False`. This switches
both the model that writes answers and the model that judges them, between Ollama and OpenAI.

### Walkthrough: from the notebook to the trap in Phoenix

Follow these steps in order. Each one names the notebook section to run and what to check
for in the Phoenix UI.

1. **Run "Setup & install" and "Configuration"** (sections 1-2). This installs packages and
   sets `LOCAL_MODE`.
2. **Run "Phoenix instrumentation"** (section 3). This starts the Phoenix server and prints a
   link. Open that link now and keep the tab open for the rest of the lab.
3. **Run "Confluence ingestion"** (section 4). This pulls the 19 dummy wiki pages and marks two
   of them, on purpose, as deprecated.
4. **Run "Naive index construction"** (section 5). This builds the index at
   `similarity_top_k=1` — narrow on purpose, so a stale page can beat its replacement.
5. **Run "Trigger the failure"** (section 6). This asks a real question and sets the trap: the
   index may hand back the old, deprecated runbook instead of the current one, and the model
   writes a sure, wrong answer.
6. **Switch to Phoenix and open the trace.** Find the trace named `exercise1_naive_trigger`,
   open it, and click the `VectorIndexRetriever.retrieve` span. You will see the retrieved
   page's title and text right there on the span.
7. **Run "Evals"** (section 8). This scores the retrieved page and the answer with an LLM
   judge, and also runs a plain code check against a known list of stale pages. Watch the
   printed output: the judge often calls the answer relevant and faithful, while the code
   check fails it.
8. **Refresh the Phoenix trace.** The same span you opened in step 6 now carries
   `document_relevance`, `deprecation_check`, and `faithfulness` annotations. This is the trap,
   made visible: two judges disagree, and the plain check is the one telling the truth.
9. **Read "Diagnosis"** (section 9) for a plain account of what went wrong and why the judge
   missed it.
10. **Do Exercise 1** (section 10). Add a metadata filter that drops deprecated pages, then run
    the same query again. Check Phoenix for the new trace: the retriever span now points at the
    current page, and its `deprecation_check` annotation should read "current."
11. **Read "Before/after comparison"** (section 11) for both runs side by side in one table.
12. **Do Exercise 2 on your own** (section 12) — the same steps above, but for the
    password-reset trap, with no worked example to lean on.

### Status

This is first a personal and workshop project. Whether it becomes a pull request to the Phoenix
repo is a later, separate step, and it depends on the maintainers saying yes — Phoenix's
CONTRIBUTING.md says they are not accepting unsolicited contributions right now.
