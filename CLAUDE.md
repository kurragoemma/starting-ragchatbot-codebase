# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

Always use `uv` to run the server. Do not use `pip` directly.

```bash
# Install dependencies
uv sync

# Start the server (from project root)
./run.sh

# Or manually
cd backend && uv run uvicorn app:app --reload --port 8000
```

The app runs at `http://localhost:8000`. API docs at `http://localhost:8000/docs`.

Requires a `.env` file in the project root:
```
ANTHROPIC_API_KEY=your_key_here
```

## Architecture

This is a full-stack RAG (Retrieval-Augmented Generation) system. The backend is in `backend/`, the web UI in `frontend/`, and source documents in `docs/`.

### Request Flow

1. `frontend/script.js` sends `POST /api/query` with `{query, session_id}`
2. `backend/app.py` (FastAPI) delegates to `rag_system.RAGSystem.query()`
3. `rag_system.py` passes the query + conversation history to `ai_generator.py`
4. Claude decides whether to invoke `CourseSearchTool` (defined in `search_tools.py`)
5. If tool called → `vector_store.py` runs semantic search against ChromaDB
6. Claude generates a final answer using search results as context
7. Response + sources returned to frontend

### Key Design Decisions

- **Tool-based RAG**: Claude autonomously decides when to search — not every query triggers retrieval. This is controlled by the system prompt in `ai_generator.py`.
- **Two ChromaDB collections**: `course_catalog` (course metadata) and `course_content` (text chunks). Both live in `backend/chroma_db/` and persist across restarts.
- **Fuzzy course matching**: `vector_store._resolve_course_name()` uses vector similarity to match course names — not exact string matching.
- **Session memory**: `session_manager.py` keeps the last `MAX_HISTORY=2` exchanges in memory (lost on server restart).
- **Sentence-aware chunking**: `document_processor.py` splits text into ~800-char chunks at sentence boundaries with 100-char overlap.

### Document Format

Files in `docs/` must follow this structure for `document_processor.py` to parse them:

```
Course Title: [name]
Course Link: [url]
Course Instructor: [name]

Lesson 0: [title]
Lesson Link: [url]
[content...]

Lesson 1: [title]
[content...]
```

### Configuration

All tuneable parameters are in `backend/config.py`:
- `CHUNK_SIZE = 800`, `CHUNK_OVERLAP = 100`
- `MAX_RESULTS = 5` (search results per query)
- `MAX_HISTORY = 2` (conversation exchanges remembered)
- `ANTHROPIC_MODEL = "claude-sonnet-4-20250514"`
- `EMBEDDING_MODEL = "all-MiniLM-L6-v2"`

### Module Responsibilities

| File | Responsibility |
|------|---------------|
| `app.py` | FastAPI routes, static file serving, startup doc loading |
| `rag_system.py` | Orchestrates all components; the main entry point for queries |
| `document_processor.py` | Parses course `.txt` files into `CourseChunk` objects |
| `vector_store.py` | ChromaDB wrapper — add/search course content and metadata |
| `ai_generator.py` | Anthropic API calls; handles tool-use loop |
| `search_tools.py` | Defines `CourseSearchTool` and `ToolManager` used by Claude |
| `session_manager.py` | In-memory conversation history per session |
| `models.py` | Pydantic/dataclass models shared across modules |
| `config.py` | Single source of truth for all settings |
