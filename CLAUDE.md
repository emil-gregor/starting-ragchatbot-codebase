# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A RAG chatbot that answers questions about course materials. FastAPI backend (Python 3.13, managed with `uv`) plus a static HTML/JS frontend served by the same app. Uses ChromaDB + `sentence-transformers` for retrieval and the Anthropic API for generation.

## Commands

```bash
uv sync                                   # install dependencies
cp .env.example .env                      # then set ANTHROPIC_API_KEY
./run.sh                                  # start server on :8000
cd backend && uv run uvicorn app:app --reload --port 8000   # manual start
```

- Web UI: http://localhost:8000 — API docs: http://localhost:8000/docs
- The server **must be started from `backend/`**: `app.py` uses relative paths `../docs`, `../frontend`, and `./chroma_db`.
- There is no test suite or linter configured yet. Always use `uv run` / `uv add` rather than pip.

## Architecture

Request flow for `POST /api/query` (`backend/app.py`):

1. `RAGSystem.query` (`rag_system.py`) is the orchestrator. It fetches session history from `SessionManager` (in-memory, last `MAX_HISTORY` exchanges, injected into the **system prompt** as text, not as message turns).
2. `AIGenerator.generate_response` (`ai_generator.py`) calls Claude with the `search_course_content` tool and `tool_choice: auto`. Claude decides whether to search.
3. If `stop_reason == "tool_use"`, `_handle_tool_execution` runs the tool via `ToolManager`, then makes **one** follow-up call **without tools**. Only a single round of tool use is supported.
4. `CourseSearchTool` (`search_tools.py`) calls `VectorStore.search`, formats results with `[Course - Lesson N]` headers, and stashes `last_sources` on itself. `RAGSystem` reads them via `ToolManager.get_last_sources()` and resets them after each query — sources are passed out-of-band, not through the model's response.

Vector store (`vector_store.py`) uses two ChromaDB collections:
- `course_catalog` — one doc per course (title, instructor, link, lessons serialized as JSON in metadata). Used to fuzzy-resolve a `course_name` argument to an exact title via vector search.
- `course_content` — text chunks with `course_title` / `lesson_number` metadata, filtered with `where` clauses.

The course **title is the unique ID** across both collections.

New tools: subclass `Tool` in `search_tools.py` (implement `get_tool_definition` returning an Anthropic tool schema, and `execute`) and register it with `ToolManager` in `RAGSystem.__init__`.

## Document ingestion

On startup, `app.py` loads every `.txt`/`.pdf`/`.docx` in `docs/` (all files are read as UTF-8 text). Courses whose title already exists in ChromaDB are **skipped**, so editing a doc won't re-index it — delete `backend/chroma_db` or call `add_course_folder(..., clear_existing=True)`.

`DocumentProcessor` expects this format:

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 0: <lesson title>
Lesson Link: <url>
<content...>
```

Content is split into sentence-based chunks (`CHUNK_SIZE`=800 chars, `CHUNK_OVERLAP`=100). Tunables, model name, and embedding model live in `backend/config.py`.
