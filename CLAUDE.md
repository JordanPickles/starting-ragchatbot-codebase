# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Dependencies are managed with `uv` (Python 3.13+). Always use `uv` to run commands and manage packages — never `pip` directly.

```bash
uv sync                                   # install dependencies
uv add <package>                          # add a dependency
./run.sh                                  # start the app (runs uvicorn from backend/)
cd backend && uv run uvicorn app:app --reload --port 8000   # manual start
```

- Web UI: http://localhost:8000 — API docs: http://localhost:8000/docs
- Requires `ANTHROPIC_API_KEY` in a root `.env` (see `.env.example`).
- The server **must be started from `backend/`**: paths like `../docs`, `../frontend` and `./chroma_db` are relative to that directory.

There is no test suite, linter or formatter configured yet.

## Architecture

A FastAPI app that answers questions about course transcripts using tool-based RAG: Claude decides whether to search, rather than retrieval happening on every query.

**Request flow** (`/api/query`): `app.py` → `RAGSystem.query()` (`rag_system.py`) → `AIGenerator.generate_response()` (`ai_generator.py`) calls Claude with the `search_course_content` tool → if Claude returns `tool_use`, `ToolManager` runs `CourseSearchTool` (`search_tools.py`) → `VectorStore.search()` → the results go back to Claude in a **second call made without tools**, which writes the final answer. So only one round of tool use is possible per query; adding multi-step search means changing `_handle_tool_execution`.

**Sources** for the UI travel separately from the answer: `CourseSearchTool._format_results` saves them on `self.last_sources`, `RAGSystem.query` reads them via `ToolManager.get_last_sources()` and then resets them. New tools that should show sources need a `last_sources` attribute.

**Two ChromaDB collections** (`vector_store.py`, persisted in `backend/chroma_db/`):
- `course_catalog`: one doc per course, ID = course title. Lesson list stored as a JSON string in `lessons_json` metadata (Chroma metadata can't hold nested structures).
- `course_content`: chunks with `course_title`, `lesson_number`, `chunk_index` metadata; IDs are `{title_with_underscores}_{chunk_index}`.

Search takes two steps: a fuzzy `course_name` is first matched to an exact title by vector-searching `course_catalog` (top 1), then `course_content` is queried with a `where` filter on title and/or lesson number.

**Ingestion** happens on FastAPI startup from `../docs`. Courses whose title is already in Chroma are skipped, so after changing chunking or parsing logic, delete `backend/chroma_db/` (or call `add_course_folder(..., clear_existing=True)`) to re-index. `DocumentProcessor` expects this file format:

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 0: <lesson title>
Lesson Link: <url>
<transcript text...>
```

Text is split into sentence-based chunks (`CHUNK_SIZE`/`CHUNK_OVERLAP` in `config.py`), and each chunk gets a lesson/course context prefix before it is embedded. The `.pdf`/`.docx` extensions are accepted, but every file is read as plain UTF-8 text.

**Conversation history** (`session_manager.py`) is kept in memory only (lost on restart). It is passed to Claude as text inside the system prompt, not as message turns, and trimmed to `MAX_HISTORY` exchanges.

All tunables (model, embedding model, chunk size, max results, history length) live in `backend/config.py`.

**Frontend** (`frontend/`) is plain HTML/JS/CSS served by FastAPI's static mount at `/`; `script.js` calls `/api/query` and `/api/courses`.
