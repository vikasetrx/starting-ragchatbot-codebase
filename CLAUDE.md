# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Course Materials RAG System** - a full-stack web application that uses Retrieval-Augmented Generation to answer questions about course materials through semantic search and AI-powered responses.

**Tech Stack:**
- Backend: FastAPI (Python 3.13+)
- Vector DB: ChromaDB with SentenceTransformer embeddings (`all-MiniLM-L6-v2`)
- AI: Anthropic's Claude (`claude-sonnet-4-20250514`) with tool use
- Frontend: Vanilla JavaScript
- Package Manager: uv

## Development Commands

### Setup
```bash
# Install dependencies
uv sync

# Setup environment (create .env with ANTHROPIC_API_KEY)
cp .env.example .env
# Edit .env and add your Anthropic API key
```

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start (from backend directory)
cd backend
uv run uvicorn app:app --reload --port 8000
```

**Endpoints:**
- Web Interface: `http://localhost:8000`
- API Documentation: `http://localhost:8000/docs`

## Architecture

### RAG Pipeline Flow

The query handling follows this architecture:

```
Frontend → API Endpoint → RAGSystem → Claude AI (with tools) → Vector Search → Response
```

**Key architectural pattern:** This uses Claude's **tool use** capability rather than hardcoded retrieval. Claude autonomously decides when to invoke the `search_course_content` tool.

### Core Components (`backend/`)

1. **`app.py`** - FastAPI application
   - Two endpoints: `/api/query` (POST), `/api/courses` (GET)
   - Serves static frontend files from `../frontend`
   - Auto-loads course documents from `../docs` on startup

2. **`rag_system.py`** - Main orchestrator
   - Coordinates all RAG components
   - Manages document ingestion via `add_course_folder()`
   - Core method: `query(query, session_id)` returns `(answer, sources)`

3. **`ai_generator.py`** - Claude API integration
   - Two-phase tool execution pattern:
     1. Initial request with tool definitions
     2. If `stop_reason == "tool_use"`, execute tools and get final response
   - Method: `generate_response()` handles conversation history + tools

4. **`vector_store.py`** - ChromaDB wrapper
   - **Two collections:**
     - `course_catalog`: Course metadata (titles, instructors, lesson lists)
     - `course_content`: Chunked course material for semantic search
   - Smart course name resolution using semantic search on catalog
   - `search()` method supports filtering by course name and lesson number

5. **`search_tools.py`** - Tool definitions for Claude
   - `CourseSearchTool`: Defines `search_course_content` tool
   - `ToolManager`: Registry pattern for tool execution
   - Tracks sources from last search for UI display

6. **`session_manager.py`** - Conversation context
   - In-memory session storage (not persistent)
   - Maintains last `MAX_HISTORY * 2` messages per session
   - History formatted as string prepended to system prompt

7. **`document_processor.py`** - Course document parser
   - Expects specific format in `docs/*.txt`:
     ```
     Course Title: [title]
     Course Link: [url]
     Course Instructor: [instructor]

     Lesson N: [title]
     Lesson Link: [url]
     [content...]
     ```
   - Chunks text at sentence boundaries (800 chars, 100 char overlap)
   - Creates `CourseChunk` objects with metadata (course, lesson, chunk_index)

8. **`config.py`** - Configuration dataclass
   - Loads from environment variables
   - Key settings: `CHUNK_SIZE=800`, `MAX_RESULTS=5`, `MAX_HISTORY=2`

9. **`models.py`** - Pydantic models
   - `Course`, `Lesson`, `CourseChunk` data structures

### Frontend (`frontend/`)

Simple single-page app:
- `index.html` - Chat interface
- `script.js` - Handles user input, API calls, displays responses with sources
- `styles.css` - UI styling

### Data Flow

**Document Ingestion:**
```
docs/*.txt → DocumentProcessor.process_course_document()
→ VectorStore.add_course_metadata() (catalog collection)
→ VectorStore.add_course_content() (content collection)
```

**Query Processing:**
```
User query → RAGSystem.query()
→ AIGenerator.generate_response(tools=[search_course_content])
→ Claude invokes tool → ToolManager.execute_tool()
→ CourseSearchTool.execute() → VectorStore.search()
→ ChromaDB semantic search with filters
→ Results to Claude → Final response generated
→ Return (answer, sources)
```

## Key Architectural Decisions

### Why Two ChromaDB Collections?
- **Catalog**: Small collection for fuzzy course name matching. User says "anthropic course" → resolves to exact title "Building Towards Computer Use with Anthropic"
- **Content**: Large collection with all chunks. Filtered by exact course title + optional lesson number

### Tool Use Pattern
Claude is given the `search_course_content` tool definition and autonomously decides:
- When to search
- What query to use
- Which filters to apply (course_name, lesson_number)

This is more flexible than hardcoded retrieval - Claude can skip search if it already knows the answer.

### Session Management
Sessions are in-memory only (lost on restart). For production, implement persistent storage in `session_manager.py`.

### Chunking Strategy
Sentence-based chunking preserves semantic units better than fixed-size chunks. Overlap ensures context isn't lost at boundaries.

## Configuration

All settings in `backend/config.py`:
- `ANTHROPIC_MODEL`: Current model (Claude Sonnet 4)
- `CHUNK_SIZE`/`CHUNK_OVERLAP`: Text chunking parameters
- `MAX_RESULTS`: Top K results from vector search
- `MAX_HISTORY`: Conversation turns to include in context
- `CHROMA_PATH`: ChromaDB storage location (default `./chroma_db`)

## Adding Course Materials

Place `.txt` files in `docs/` directory following the expected format (see `document_processor.py` docstring). Files are auto-loaded on server startup.

To manually trigger ingestion:
```python
# In backend/app.py or via Python REPL
courses, chunks = rag_system.add_course_folder("../docs", clear_existing=False)
```

## Common Modifications

**Change AI model:** Update `ANTHROPIC_MODEL` in `config.py`

**Adjust retrieval:** Modify `MAX_RESULTS` in `config.py` or `limit` parameter in `vector_store.search()`

**Change chunking:** Update `CHUNK_SIZE`/`CHUNK_OVERLAP` in `config.py` (requires re-ingesting documents)

**Add new tools:** Create tool class in `search_tools.py`, register in `rag_system.py:__init__`

**Persist sessions:** Modify `session_manager.py` to use database instead of in-memory dict

## File References

When tracing bugs or features, key interaction points:
- Query entry: `app.py:query_documents` (line 57)
- RAG orchestration: `rag_system.py:query` (line 102)
- Tool execution: `ai_generator.py:_handle_tool_execution` (line 89)
- Vector search: `vector_store.py:search` (line 61)
- Document parsing: `document_processor.py:process_course_document` (line 97)
