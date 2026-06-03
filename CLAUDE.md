# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

AIOps Incident Response Assistant — a RAG chatbot that answers IT on-call
questions from local runbooks. Stack: **Streamlit** UI → **LangChain 1.x** →
**ChromaDB** (local vector store) → **Google Gemini** (LLM + embeddings).

## Architecture (3 files)

- `ingest.py` — one-off pipeline: loads `data/runbooks/*.{txt,pdf}`, chunks them,
  embeds with Gemini, and persists to `./chroma_db`. Run before the app and after
  adding/changing runbooks.
- `rag_chain.py` — builds the `RetrievalQA` chain (Chroma retriever + Gemini LLM).
  Owns the shared constants `CHROMA_DIR` and `EMBEDDING_MODEL`.
- `app.py` — Streamlit chat UI. Imports `get_rag_chain()` and caches it via
  `@st.cache_resource`.

Data flow: `ingest.py` writes `chroma_db/` → `rag_chain.py` reads it → `app.py`
serves queries.

## Run commands (Windows / PowerShell, venv at `.\venv`)

```powershell
.\venv\Scripts\activate
pip install -r requirements.txt
python ingest.py            # build the vector store (run first)
streamlit run app.py        # launch UI at http://localhost:8501
```

## Critical project rules (learned the hard way)

1. **Embedding model must match between ingest and query.** It is defined once as
   `EMBEDDING_MODEL` in `rag_chain.py` and imported by `ingest.py`. If you change
   it, you MUST delete `chroma_db/` and re-run `python ingest.py` — old and new
   embeddings have incompatible dimensions.

2. **Gemini model names get retired.** A `404 NOT_FOUND` means the model is gone;
   a `429 RESOURCE_EXHAUSTED` with `limit: 0` means it's paid-tier only (e.g.
   `gemini-2.5-pro` is not on the free tier — use `gemini-2.5-flash`). To list
   what's currently available on the key:
   ```python
   from google import genai; import os
   c = genai.Client(api_key=os.getenv("GOOGLE_API_KEY"))
   for m in c.models.list(): print(m.name, m.supported_actions)
   ```

3. **Restart Streamlit after editing `rag_chain.py`.** The chain is cached with
   `@st.cache_resource`, and that cache does NOT invalidate when an imported
   module changes — only when `app.py`'s own cached function changes. Ctrl+C and
   re-run, or use the app's ⋮ menu → Clear cache.

4. **LangChain 1.x import paths.** This project is on LangChain 1.x:
   - `RetrievalQA` → `from langchain_classic.chains import RetrievalQA`
   - `PromptTemplate` → `from langchain_core.prompts import PromptTemplate`
   - `Chroma` → `from langchain_chroma import Chroma` (NOT `langchain_community`)
   - text splitters → `from langchain_text_splitters import ...`

## Conventions

- Secrets live in `.env` (`GOOGLE_API_KEY`), never committed. Both entry points
  guard for a missing key with a clear message.
- Runbooks are plain `.txt`/`.pdf` in `data/runbooks/`. Add files there and
  re-ingest — no code changes needed.

## Known tech debt (works, but dated)

- `RetrievalQA` is deprecated (runs via the `langchain-classic` compat layer). The
  modern path is LCEL + `create_retrieval_chain`.
- No conversational memory — each query is independent; follow-up questions don't
  see prior turns.
