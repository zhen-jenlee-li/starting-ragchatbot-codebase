# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Retrieval-Augmented Generation (RAG) chatbot system for answering questions about course materials. The system uses ChromaDB for vector storage, Anthropic's Claude for AI generation, and provides a web interface through FastAPI.

## Prerequisites

- Python 3.13 or higher
- uv (Python package manager)
- Anthropic API key
- Windows users: Use Git Bash for running commands

## Setup and Running

### Installation
```bash
# Install Python dependencies
uv sync

# Create .env file with API key
ANTHROPIC_API_KEY=your_api_key_here
```

### Starting the Application
```bash
# Quick start with shell script
./run.sh

# Manual start (from project root)
cd backend
uv run uvicorn app:app --reload --port 8000
```

The application serves at:
- Web Interface: http://localhost:8000
- API Documentation: http://localhost:8000/docs

## Architecture

### Core Components

The RAG system follows a modular architecture with distinct responsibilities:

**RAGSystem** (`backend/rag_system.py`) - Main orchestrator that coordinates all components. Handles document ingestion workflow (DocumentProcessor → VectorStore) and query workflow (user query → AI with tools → VectorStore search → formatted response).

**VectorStore** (`backend/vector_store.py`) - Manages ChromaDB with two collections:
- `course_catalog`: Course metadata (titles, instructors, links) for semantic course name matching
- `course_content`: Chunked lesson content for retrieval

Provides unified `search()` interface with fuzzy course name matching and optional lesson filtering.

**AIGenerator** (`backend/ai_generator.py`) - Handles Claude API interactions with tool calling. Uses agentic workflow where Claude decides when to search using the `search_course_content` tool. Manages conversation history and tool execution loop.

**DocumentProcessor** (`backend/document_processor.py`) - Parses course documents with expected format:
```
Line 1: Course Title: [title]
Line 2: Course Link: [url]
Line 3: Course Instructor: [name]
Following lines: Lesson N: [title] with Lesson Link: [url] and content
```
Chunks text at sentence boundaries with configurable size/overlap.

**SessionManager** (`backend/session_manager.py`) - Maintains conversation context per session with configurable message history limits.

**ToolManager & CourseSearchTool** (`backend/search_tools.py`) - Implements tool calling pattern. CourseSearchTool wraps VectorStore search and formats results for Claude. Tracks sources for UI display.

### Data Models

All models defined in `backend/models.py` using Pydantic:
- **Course**: title (unique ID), course_link, instructor, lessons[]
- **Lesson**: lesson_number, title, lesson_link
- **CourseChunk**: content, course_title, lesson_number, chunk_index

### Configuration

`backend/config.py` centralizes all settings loaded from environment:
- `ANTHROPIC_MODEL`: "claude-sonnet-4-20250514"
- `EMBEDDING_MODEL`: "all-MiniLM-L6-v2" (sentence-transformers)
- `CHUNK_SIZE`: 800 characters
- `CHUNK_OVERLAP`: 100 characters
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation exchanges
- `CHROMA_PATH`: "./chroma_db" (persistent storage)

### Document Processing Flow

1. Course documents placed in `docs/` folder
2. On startup, `app.py` calls `rag_system.add_course_folder("../docs")`
3. DocumentProcessor parses each file into Course + CourseChunk[]
4. VectorStore stores:
   - Course metadata in `course_catalog` (title as ID)
   - Content chunks in `course_content` (with course_title + lesson_number metadata)
5. Existing courses (by title) are skipped to avoid duplicates

### Query Processing Flow

1. User submits query via `/api/query` endpoint
2. RAGSystem creates session if needed
3. AIGenerator calls Claude with:
   - System prompt defining search tool behavior
   - User query
   - Conversation history (if exists)
   - Tool definitions from ToolManager
4. Claude autonomously decides whether to use `search_course_content` tool
5. If tool used: ToolManager executes CourseSearchTool → VectorStore.search()
6. AIGenerator sends tool results back to Claude for final response
7. SessionManager records exchange for context
8. Response + sources returned to frontend

### Frontend Structure

Simple static HTML/CSS/JavaScript in `frontend/`:
- `index.html`: Main chat interface
- `script.js`: API communication and UI updates
- `style.css`: Styling

FastAPI serves frontend files via StaticFiles mount with no-cache headers for development.

## Key Design Patterns

**Two-Collection Vector Store**: Separates course metadata (for fuzzy course name resolution) from course content (for semantic retrieval). This allows "search in Introduction to Python course" to work even if user says "intro to python".

**Tool-Based RAG**: Instead of always searching, Claude decides when search is needed. This prevents unnecessary searches for general questions and enables one-shot responses.

**Chunk Contextualization**: First chunk of each lesson prefixed with "Lesson N content:" and last lesson chunks include "Course {title} Lesson N content:" to preserve context during chunking.

**Session-Based Conversation**: Each user session maintains conversation history allowing follow-up questions without losing context.

## Common Operations

### Adding New Tools
1. Create tool class inheriting from `Tool` in `search_tools.py`
2. Implement `get_tool_definition()` and `execute(**kwargs)`
3. Register with ToolManager in `RAGSystem.__init__`

### Modifying AI Behavior
Edit `AIGenerator.SYSTEM_PROMPT` in `ai_generator.py`. Current prompt emphasizes:
- One search per query maximum
- No meta-commentary about search process
- Direct, concise, educational responses

### Adjusting Chunk Size/Overlap
Modify `CHUNK_SIZE` and `CHUNK_OVERLAP` in `config.py`. Note: Changes only affect newly processed documents.

### Rebuilding Vector Store
```python
# In RAGSystem
rag_system.add_course_folder(folder_path, clear_existing=True)
```

### Testing Document Format
Place test `.txt` file in `docs/` following the expected format. Restart server to see parsing results in console.

## API Endpoints

- `POST /api/query`: Submit question with optional session_id
  - Request: `{"query": "string", "session_id": "optional"}`
  - Response: `{"answer": "string", "sources": ["string"], "session_id": "string"}`

- `GET /api/courses`: Get course statistics
  - Response: `{"total_courses": int, "course_titles": ["string"]}`

## Dependencies

Key libraries and their roles:
- `chromadb==1.0.15`: Vector database for embeddings
- `anthropic==0.58.2`: Claude API client
- `sentence-transformers==5.0.0`: Embedding generation
- `fastapi==0.116.1`: Web framework
- `uvicorn==0.35.0`: ASGI server
- `python-dotenv==1.1.1`: Environment variable loading
