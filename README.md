# Semiconductor Datasheet RAG - Notebooks (Qdrant)

Notebook rework of `../semiconductor_datasheet_RAG`, with **Qdrant** as the vector store instead of SQLite.

| Notebook | What it does |
|---|---|
| [01_ingestion.ipynb](01_ingestion.ipynb) | PDF path -> extract text / headings / tables / captions -> chunk -> dense + sparse embeddings -> Qdrant |
| [02_retrieval.ipynb](02_retrieval.ipynb) | Question -> dense, keyword and hybrid (RRF) search in Qdrant -> cited context (optional LLM answer) |

## Setup (once)

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\python -m pip install -r requirements.txt
```

Behind the corporate proxy, pip needs `truststore` (pip 24.2+ bundles it). If `pip install` fails with
`CERTIFICATE_VERIFY_FAILED` on the old bundled pip, upgrade pip first from a wheel downloaded by another
interpreter that already trusts the system certificates.

In VS Code: open this folder, open a notebook, **Select Kernel -> Python Environments -> .venv**
(`.vscode/settings.json` already points VS Code at it).

## Qdrant

Start the server (Rancher Desktop / Docker):

```powershell
docker run -d --name qdrant -p 6333:6333 -p 6334:6334 -v qdrant_storage:/qdrant/storage qdrant/qdrant
```

Dashboard: http://localhost:6333/dashboard. Step 5a of notebook 1 waits for the server (up to `QDRANT_WAIT_TIMEOUT_S`).

Before the server is installed, set `QDRANT_MODE = "local"` in step 1 of **both** notebooks: Qdrant runs embedded
from `data/qdrant_local`. Only one process can open the local store at a time, so run each notebook's last cell
(`client.close()`) before switching notebooks.

## Collection layout

Collection `semiconductor_datasheets`, one point per chunk:

- vector `dense`: `BAAI/bge-small-en-v1.5`, 384-dim, cosine
- vector `sparse`: keyword term weights with Qdrant's IDF modifier (BM25-style)
- payload: `text`, `type` (text/table/caption), `page`, `section_path`, `parts`, `table_id`, `file_name`,
  `doc_sha`, `title`, `manufacturer`, `revision`, `embedding_model`, `ingested_at`

Re-running notebook 1 on the same PDF replaces that PDF's points.

Outputs per PDF: `data/processed/<sha16>/parsed.json` (extraction) and `chunks.jsonl` (chunks).
