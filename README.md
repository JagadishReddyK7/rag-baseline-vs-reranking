# RAG: Baseline Retrieval vs. Cross-Encoder Re-Ranking

Assignment for the *Language Models and Agents* course. Two retrieval pipelines are built over the same chunked
document and compared on failure cases, retrieval quality, latency and scalability.

**Everything is in one notebook:** [`RAG_Baseline_vs_ReRanking.ipynb`](RAG_Baseline_vs_ReRanking.ipynb). It is committed
with all outputs, so it can be read without running it.

## Setup

| Component | Choice |
|---|---|
| Document | EU Artificial Intelligence Act, Regulation (EU) 2024/1689, Official Journal version (144 pages) → 715 chunks |
| Bi-encoder (Stage 1) | `sentence-transformers/all-MiniLM-L6-v2` + FAISS `IndexFlatIP` (exact cosine) |
| Cross-encoder (Stage 2) | `cross-encoder/ms-marco-MiniLM-L-6-v2` |
| LLM | Google Gemini `gemini-3.7-flash`, temperature 0, strictly grounded prompt |

| | Pipeline 1: Baseline | Pipeline 2: Re-ranking |
|---|---|---|
| Retrieval | bi-encoder → Top 10 | same Top 10 |
| Context selection | Top 3 as-is | cross-encoder re-scores 10 → Top 3 |
| Generation | identical prompt → Answer 1 | identical prompt → Answer 2 |

## Key results

**Failure cases.** In every case the gold chunk (the one holding the decisive clause) moves into the Top 3 only after re-ranking:

| Case | Failure type | Gold rank: baseline → re-ranked |
|---|---|---|
| F1 | Distractor (boilerplate) + exception | 7 → 1 |
| F2 | Keyword trap | 10 → 3 |
| F3 | Exception missed | 9 → 2 |
| F4 | Exception / corner case | 7 → 2 |
| F5 | Negation | 9 → 2 |
| F6 | Role constraint | 7 → 1 |
| F7 | Buried chunk | 7 → 1 |
| C1 | Control: re-ranking hurts | 2 → 4 |
| C2 | Limitation: gold not in Top 10 | – |

**Aggregate evaluation** (65 labelled queries):

| | Baseline | Re-ranked |
|---|---|---|
| Hit@1 | 14/65 | 23/65 |
| Hit@3 | 34/65 | 41/65 |
| MRR@10 | 0.366 | 0.496 |

**Latency** (median, CPU only):

| Stage | Median |
|---|---|
| Baseline retrieval | 6.8 ms |
| Cross-encoder re-ranking (10 pairs) | 155 ms |
| Re-ranking pipeline retrieval | 162 ms |

Cross-encoding the whole corpus takes 11.4 s per query (extrapolated: about 4.4 h per query for 1 million chunks).

## Repository contents

```
RAG_Baseline_vs_ReRanking.ipynb   main notebook (code, explanations, outputs)
data/EU_AI_Act_2024_1689.pdf      source document (EUR-Lex, CELEX 32024R1689)
data/eval_queries.json            73 labelled queries used for the aggregate evaluation
llm_cache.json                    cached Gemini answers, so the notebook re-runs without API calls
requirements.txt                  pinned package versions
RAG_LMA_Assignment_v1.md          assignment specification
```

## How to run

1. Python 3.11+ (developed on 3.13, CPU only):
   ```bash
   python -m venv .venv
   .venv\Scripts\activate          # Windows  (Linux/macOS: source .venv/bin/activate)
   pip install -r requirements.txt
   ```
2. Create a file named `.env` in the project folder containing `GEMINI_API_KEY=<your key>` (free key from
   [Google AI Studio](https://aistudio.google.com)). Optional: `GEMINI_MODEL=<model>` overrides the default model.
3. Open the notebook and run all cells (about 3–4 minutes on CPU).

Answers are read from `llm_cache.json` when the prompt and model match, so a re-run makes **no API calls**. Delete the
file to generate fresh answers. The Gemini free tier allows only 20 requests per model per day, and a full run needs 18.

## Notes

- The source document is reused under the European Commission's document reuse policy (Decision 2011/833/EU).
- The 2024 Official Journal text is used as a fixed snapshot. Later amendments to the Act do not affect this experiment.
