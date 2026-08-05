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

### Prerequisites

**For local mode (the default, no API keys):**
- [Ollama](https://ollama.com), installed and running
- Pull these models: `ollama pull qwen2.5`, `ollama pull nomic-embed-text`, and
  `ollama pull mistral-nemo:12b`. Use `mistral-nemo:12b` as the judge — a different model from the
  one that writes the answer, so it never judges its own work. It is a big pull, about 7GB, so
  fetch it before a live session.
- 16GB or more of unified memory (M2/M3 class Apple Silicon)
- A free Atlassian Cloud site (it bundles Jira and Confluence, free for up to 10 users, no card
  needed — sign up at [atlassian.com](https://www.atlassian.com)). Fill a Confluence space with the
  dummy "Meridian Analytics" wiki pages the notebook expects, and make a Confluence API token at
  `id.atlassian.com/manage-profile/security/api-tokens`.

**For cloud mode (an alternate path, for example Colab):**
- An `OPENAI_API_KEY`. It covers both the writer and the judge in this mode.

**Known limit — we do not hide it:** the local judge (`mistral-nemo:12b`) is less steady than a
cloud judge. In tests, `FaithfulnessEvaluator` gave different verdicts on the same input, run after
run: about 2 in 3 "faithful," about 1 in 3 "unfaithful." We leave this in view on purpose. The
notebook shows it live as proof that LLM judges can be unreliable — a March 2026 RAND study found
the same thing — not as a bug to hide. It is also why the lab's real verdict rests on the
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

### Status

This is first a personal and workshop project. Whether it becomes a pull request to the Phoenix
repo is a later, separate step, and it depends on the maintainers saying yes — Phoenix's
CONTRIBUTING.md says they are not accepting unsolicited contributions right now.
