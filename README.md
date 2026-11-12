# RAG Eval Harness

A small RAG (retrieval-augmented generation) system plus an evaluation harness that measures how well it works.
**Status: Step 1 (the system under test) is built.**

## Run it

You need Docker Desktop running, Python 3.11+, and an Anthropic API key (https://console.anthropic.com → API Keys).

```bash
make setup                          # create .venv and install packages (once)
cp .env.example .env                # then paste your key into .env
make ingest                         # start Postgres, chunk + embed + store the docs
make ask Q="Can I return a bike I have ridden 300 km?"
```

`python -m app.ask "..." --json` prints the full result. Every run is appended to `runs/ask_log.jsonl`.

---

## Step 1 explained for beginners

### What is RAG?

An LLM doesn't know your company's documents. RAG fixes that in two moves:

1. **Retrieve**: find the few passages of your documents most related to the question.
2. **Generate**: paste those passages into the prompt and tell the LLM to answer *only* from them.

The project's goal is to **measure** how well this works. So Step 1 builds a RAG where every setting can be changed, and every answer is recorded with what it used and what it cost.

### The pipeline, piece by piece

```
data/docs/*.md ──► chunking.py ──► embed.py ──► Postgres + pgvector      (ingest, run once per config)
                                                      │
question ──► embed.py ──► nearest chunks (top_k) ─────┘──► prompt ──► Claude ──► answer + log
                                                                                   (rag.py / ask.py)
```

| Concept | What it means | Where in the code |
|---|---|---|
| **Document** | A source file. Here: 6 fictional "Helio Bikes" policy docs | `data/docs/` |
| **Chunk** | A slice of a document (here 256 tokens) small enough to retrieve precisely. Chunks overlap by 32 tokens so a sentence cut at the border isn't lost | `app/chunking.py` |
| **Token** | The unit LLMs read, about ¾ of a word | |
| **Embedding** | A list of numbers (384 here) that represents the *meaning* of a text. Texts with similar meanings get similar numbers | `app/embed.py` (a local model, free) |
| **Vector database** | Stores the embeddings and finds the closest ones to the question's embedding. pgvector adds this to Postgres | `app/db.py`, `docker-compose.yml` |
| **Cosine similarity** | The closeness score from 0 to 1 printed next to each source | the `<=>` operator in `rag.py` |
| **top_k** | How many chunks are given to the LLM | `configs/base.yaml` |
| **Prompt template** | The instructions plus the slots for sources and question, kept in a versioned file | `prompts/v1.txt` |

### Why the docs are about a fictional company

If you test RAG on Wikipedia, the LLM already knows the answers, so you can't tell whether it used your sources or its memory. With invented facts ("the 625 Wh battery costs 720 EUR"), a correct answer must come from retrieval, and an invented one is clearly a hallucination. You can replace these docs with your own corpus later; aim for 20 to 50 pages before you build the dataset in Step 2.

### Why everything is in a config file

Step 6 compares settings (chunk size 256 vs 512, top-k 3 vs 8, different models, prompt v1 vs v2). To run an experiment, copy `configs/base.yaml`, change **one** value, and run with `--config configs/yours.yaml`. If you change chunking or the embedding model, run `ingest` again with that config. Each chunking setting gets its own "index", so several can live side by side.

### What gets logged per run

`runs/ask_log.jsonl` has one JSON line per question:
- the question and the answer
- `chunks`: the IDs, scores and text of the sources used (Steps 2 and 4 need the IDs for recall@k)
- `input_tokens`, `output_tokens`, `cost_usd`
- `latency_ms`: retrieval, generation and total
- `config_fingerprint`: a hash of the full config, so every result can be traced to the exact settings

### Files

```
app/config.py     load configs/*.yaml
app/chunking.py   split text into overlapping token windows
app/embed.py      text → vector (fastembed, local)
app/db.py         Postgres connection + table
app/ingest.py     python -m app.ingest   (docs → chunks → vectors → DB)
app/rag.py        answer(question, config) → answer, chunks, tokens, cost, latency
app/ask.py        python -m app.ask "..." (CLI + logging)
prompts/v1.txt    prompt template
configs/base.yaml all settings
```

### Things to try to build intuition

1. `make ask Q="How much is a replacement 625 Wh battery?"`: a fact in a single place.
2. `make ask Q="If I buy a Helio Cargo with a second battery, is delivery free?"`: needs two documents (multi-hop).
3. `make ask Q="Do you sell kids' bikes?"`: the docs don't say, so it should refuse.
4. Set `top_k: 1` in a copied config and ask question 2 again. It will probably fail, and that kind of difference is what Step 6 measures.
# rag-eval
