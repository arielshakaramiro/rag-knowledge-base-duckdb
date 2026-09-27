# RAG Knowledge Base in DuckDB

[Bahasa Indonesia](README.id.md)

A single notebook that prepares a small knowledge base for Retrieval-Augmented Generation (RAG): raw text and a PDF catalog are chunked, embedded, and stored in DuckDB with an HNSW index, then queried with semantic search and a small local LLM.

The focus is the ingestion side, which is easy to get subtly wrong: re-running a cell should not duplicate data, chunk parameters should be validated, and similarity scores should be on a scale you can actually put a threshold on.

## Pipeline

```
FAQ text + PDF catalog
        │
        ▼
  chunk_text()  ── word-based chunks with overlap, rejects overlap >= size
        │
        ▼
  embed()       ── paraphrase-multilingual-MiniLM-L12-v2, 384-dim, L2-normalized
        │
        ▼
  DuckDB        ── id = SHA-256 of chunk text, INSERT ... ON CONFLICT DO NOTHING
  + vss HNSW    ── metric = cosine
        │
        ▼
  search()      ── array_cosine_distance, score shown as similarity (1 - distance)
        │
        ▼
  rag_answer()  ── drop chunks under SIM_THRESHOLD, then Qwen2.5-3B-Instruct
                   via create_chat_completion (ChatML template), temperature 0.2
```

## Results

All numbers below come from the executed notebook in this repository.

### Ingestion

| Check | Result |
|---|---|
| Three FAQ texts ingested | 3 rows |
| Same cell run again | still **3 rows** (duplicates skipped by content hash) |
| PDF catalog ingested (3 pages) | 13 new chunks, 16 rows total |
| `chunk_text(size=5, overlap=15)` | rejected with a clear `ValueError` |
| `chunk_text(size=6, overlap=8)` | rejected with a clear `ValueError` |

### Retrieval

![Retrieval scores](images/retrieval-scores.png)

Scores are taken from the search outputs in the notebook (the FAQ questions were run before the catalog was added; the ATK, ink and Tesla questions after).

### Answers

| Question | Retrieved context | Answer |
|---|---|---|
| Berapa lama proses pengiriman reguler | 3 chunks above threshold (0.58 to 0.63) | "Pengiriman reguler memakan waktu 2-4 hari kerja." |
| Berapa harga saham Tesla hari ini? | no chunk above 0.35 | refusal, **LLM not called** |
| ada produk atk apa aja | only an unrelated FAQ chunk passed (0.3524) | refusal |

## What I learned

- **Random IDs make ingestion non-idempotent.** With `uuid4()` as the primary key, every re-run of the ingest cell silently inserts the same chunks again, and the duplicates then take up retrieval slots. A content hash as the ID fixes this at the database level.
- **Overlap larger than chunk size does not fail on its own.** The step size collapses to one word and the function returns a dozen near-identical chunks. Validating the parameters turns that into an immediate error.
- **Normalized embeddings make scores usable.** With L2-normalized vectors and cosine distance, scores sit between 0 and 1, so a threshold is something you can reason about.
- **A threshold also removes good answers.** The ATK question is a real catalog question, but its best catalog chunk scored 0.3255, just under 0.35 and very close to the Tesla question (0.3183). The abbreviation "atk" is probably not well represented by this embedding model.

## Limitations

- The threshold of 0.35 was calibrated by eye on a handful of questions. It cuts at least one relevant question (ATK).
- The knowledge base is tiny (3 FAQ texts and a 3-page catalog), so the scores say little about behavior on a larger corpus.
- `llama-cpp-python` is installed without CUDA in this notebook, so Qwen2.5-3B runs on CPU even though `n_gpu_layers=-1` is set.
- Persisting an HNSW index to a DuckDB file relies on `hnsw_enable_experimental_persistence`, which DuckDB marks as experimental.

## Run it

1. Open `rag_knowledge_base_duckdb.ipynb` in Google Colab.
2. Run the cells in order. The PDF upload step expects `data/katalog_produk_emerald.pdf` (or any text-based PDF).
3. The last step mounts Google Drive and saves `knowledge.duckdb`. Change `DRIVE_DIR` to your own folder.

## Data

- Three short FAQ texts (warranty, returns, shipping), written inline in the notebook.
- `data/katalog_produk_emerald.pdf`: a synthetic product catalog for a fictional company, created for this exercise. Products and prices are not real.

## Tech stack

DuckDB + `vss` · sentence-transformers · pypdf · llama-cpp-python · Qwen2.5-3B-Instruct (GGUF, Q4_K_M) · Google Colab

## License

[MIT](LICENSE)
