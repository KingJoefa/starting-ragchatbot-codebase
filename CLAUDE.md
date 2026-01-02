# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Course Materials RAG System** - a full-stack application that uses Retrieval-Augmented Generation to answer questions about course materials. It combines semantic search (ChromaDB + Sentence Transformers) with AI-powered responses (Claude API) in a tool-based architecture.

## Development Commands

### Running the Application
```bash
# Quick start (recommended)
chmod +x run.sh
./run.sh

# Manual start
cd backend
uv run uvicorn app:app --reload --port 8000
```

### Dependency Management
```bash
uv sync              # Install/update all dependencies
```

### Access Points
- Web Interface: http://localhost:8000
- API Documentation: http://localhost:8000/docs (FastAPI auto-generated)

### Initial Setup
```bash
# 1. Create .env file with API key
echo "ANTHROPIC_API_KEY=your_key_here" > .env

# 2. Install dependencies
uv sync

# 3. Start the application
./run.sh
```

## Architecture

### High-Level Design

The system follows a **modular RAG architecture** with clear separation of concerns:

```
User Query → FastAPI → RAG System → AI Generator (with Tools) → Vector Store
                                        ↓
                                   Tool Execution
                                        ↓
                                  Search Results → Context → Claude Response
```

**Key Architectural Principles:**
1. **Tool-Based AI Interaction**: Claude dynamically decides when to search course content using defined tools, rather than always retrieving context upfront
2. **Session Management**: Maintains conversation history (configurable via `MAX_HISTORY` in config.py)
3. **Semantic Chunking**: Documents split by sentences with overlap for better context preservation
4. **Two-Collection Strategy**: Separate ChromaDB collections for course catalog (metadata) and course content (searchable chunks)

### Component Responsibilities

| Component | File | Purpose |
|-----------|------|---------|
| **RAG Orchestrator** | `backend/rag_system.py` | Coordinates all components, public API for frontend |
| **Vector Store** | `backend/vector_store.py` | ChromaDB interface, semantic search with filtering |
| **AI Generator** | `backend/ai_generator.py` | Claude API integration, tool detection & execution |
| **Document Processor** | `backend/document_processor.py` | Text parsing, chunking, metadata extraction |
| **Search Tools** | `backend/search_tools.py` | Tool definitions (JSON schema) and execution logic |
| **Session Manager** | `backend/session_manager.py` | Conversation history tracking |
| **API Layer** | `backend/app.py` | FastAPI endpoints, CORS, static file serving |
| **Data Models** | `backend/models.py` | Pydantic schemas for type safety |
| **Configuration** | `backend/config.py` | Single source of truth for all settings |

### Data Flow

**1. Document Processing (Initialization):**
```
Text File → Parse metadata → Extract lessons → Create semantic chunks → Generate embeddings → Store in ChromaDB
```

**2. Query Processing (Runtime):**
```
User Query → RAG System → AI Generator → Claude decides: Use tool?
                                           ↓ YES          ↓ NO
                                      Execute search   Generate direct answer
                                           ↓
                                    Vector Store retrieval
                                           ↓
                                    Tool results → Claude synthesis → Response
```

**3. Session Continuity:**
```
Query → Check session_id → Load history → Append to messages → Generate response → Update session
```

## Key Algorithms and Patterns

### Document Processing Algorithm
1. Read text file with UTF-8 encoding (fallback: ignore errors)
2. Extract metadata from first 4 lines:
   - Line 1: Course Title
   - Line 2: Course Link
   - Line 3: Instructor
   - Line 4: (blank separator)
3. Parse lessons using regex: `Lesson \d+: (.+)`
4. Create semantic chunks:
   - Default size: 800 characters
   - Overlap: 100 characters
   - Split on sentence boundaries (`.`, `!`, `?`)
5. Add lesson context to first chunk of each lesson
6. Generate embeddings and store in both collections

### Vector Search Strategy
- **Fuzzy Course Matching**: Uses semantic similarity on course titles (not exact match)
- **Lesson Filtering**: When lesson number provided, filters chunks to that lesson
- **Configurable Results**: Returns top-k results (default: 5) based on similarity
- **Dual Collections**:
  - `course_catalog`: Course metadata for browsing
  - `course_content`: Chunked content for semantic search

### Tool-Based Response Pattern
The system uses Claude's tool use capability with specific constraints:

1. **Tool Definition**: Search tool defined with JSON schema in `search_tools.py`
2. **System Prompt Guidance**: "One search per query maximum" to prevent excessive tool calls
3. **Tool Detection**: AI generator checks if Claude's response includes tool use
4. **Tool Execution**: If tool detected, execute search and feed results back to Claude
5. **Response Synthesis**: Claude generates final answer with tool results as context
6. **No Meta-Commentary**: System prompt instructs Claude to avoid phrases like "based on search results"

## Configuration Tuning

Key parameters in `backend/config.py`:

```python
CHUNK_SIZE = 800          # Text chunk size (characters)
CHUNK_OVERLAP = 100       # Overlap between chunks
MAX_RESULTS = 5           # Search results limit
MAX_HISTORY = 2           # Conversation memory (messages)
ANTHROPIC_MODEL = "claude-sonnet-4-20250514"
EMBEDDING_MODEL = "all-MiniLM-L6-v2"
CHROMA_PATH = "./chroma_db"
```

**Tuning Guidelines:**
- Increase `CHUNK_SIZE` for longer context per chunk (but larger embeddings)
- Increase `CHUNK_OVERLAP` to preserve more context across boundaries
- Increase `MAX_RESULTS` for more comprehensive answers (but longer prompts)
- Increase `MAX_HISTORY` for longer conversation memory (but higher token usage)

## API Endpoints

### POST `/api/query`
Main query endpoint for conversational RAG.

**Request:**
```json
{
  "query": "What is Python?",
  "session_id": "optional-session-id"
}
```

**Response:**
```json
{
  "answer": "Python is a high-level programming language...",
  "sources": ["Course: Intro to Python, Lesson 1", ...],
  "session_id": "abc123"
}
```

### GET `/api/courses`
Returns course statistics and titles.

**Response:**
```json
{
  "total_courses": 4,
  "course_titles": ["Introduction to Python", "Advanced JavaScript", ...]
}
```

## Frontend Integration

The frontend uses **vanilla JavaScript** with relative URLs for API calls:

```javascript
// All API calls use relative paths for proxy compatibility
fetch('/api/query', { method: 'POST', ... })
```

**Key Features:**
- Session persistence via localStorage
- Markdown rendering with marked.js (CDN)
- Message streaming UI (loading indicators)
- Suggested questions for discoverability

**CORS Configuration:**
The backend allows all origins for development. For production deployment, update CORS settings in `backend/app.py`.

## Common Development Tasks

### Adding a New Tool
1. Create tool class in `backend/search_tools.py` inheriting from `Tool`
2. Implement `get_definition()` with JSON schema
3. Implement `execute()` with tool logic
4. Register tool in `AIGenerator.__init__()` in `backend/ai_generator.py`

### Modifying Chunking Strategy
1. Edit `_create_chunks()` in `backend/document_processor.py`
2. Adjust `CHUNK_SIZE` and `CHUNK_OVERLAP` in `backend/config.py`
3. Re-process documents (delete `chroma_db/` folder and restart)

### Adding New Document Types
1. Update `process_course_document()` in `backend/document_processor.py`
2. Modify metadata extraction logic for new format
3. Update Pydantic models in `backend/models.py` if schema changes

### Changing AI Model
1. Update `ANTHROPIC_MODEL` in `backend/config.py`
2. Verify model supports tool use (all Claude 3+ models do)
3. Adjust `max_tokens` in `backend/ai_generator.py` if needed

## Important Conventions

### Code Organization
- **One module per responsibility**: Each file has a single, clear purpose
- **Config as single source of truth**: All tunable parameters in `config.py`
- **Pydantic for validation**: Type-safe data models with automatic validation
- **Abstract base classes**: `Tool` base class for extensibility

### Naming Conventions
- Classes: `PascalCase` (e.g., `CourseSearchTool`, `VectorStore`)
- Functions/methods: `snake_case` (e.g., `process_course_document`)
- Constants: `UPPER_CASE` (e.g., `SYSTEM_PROMPT`, `MAX_RESULTS`)
- Private methods: `_leading_underscore` (e.g., `_create_chunks`)

### Error Handling
- Try-catch blocks with informative error messages
- Graceful fallbacks (e.g., UTF-8 errors → ignore mode)
- FastAPI HTTPException with 500 status for server errors
- Frontend displays error messages in chat UI

## Technology Stack

**Backend:**
- FastAPI 0.116.1 (async web framework)
- Anthropic SDK 0.58.2 (Claude API)
- ChromaDB 1.0.15 (vector database)
- Sentence Transformers 5.0.0 (embeddings)
- Python 3.13

**Frontend:**
- Vanilla JavaScript (no framework)
- HTML5 + CSS3
- Marked.js (markdown rendering)

**Tooling:**
- uv (Python package manager - replaces pip/poetry)
- Uvicorn (ASGI server)

## File Structure Reference

```
backend/
├── app.py              # FastAPI app, endpoints, static serving
├── rag_system.py       # Main orchestrator (RAGSystem class)
├── vector_store.py     # ChromaDB interface (VectorStore class)
├── ai_generator.py     # Claude API integration (AIGenerator class)
├── document_processor.py  # Text parsing/chunking (DocumentProcessor class)
├── search_tools.py     # Tool definitions (CourseSearchTool class)
├── session_manager.py  # History tracking (SessionManager class)
├── models.py           # Pydantic data models
└── config.py           # Configuration dataclass

frontend/
├── index.html          # Main UI
├── script.js           # Client logic (session, API calls, rendering)
└── style.css           # Styling (CSS variables for theming)

docs/                   # Sample course documents
└── course{1-4}_script.txt
```

## Database Schema

**ChromaDB Collections:**

1. **course_catalog** (Course metadata)
   - Documents: Course titles
   - Metadata: `{course_link, instructor, lessons: [...]}`
   - Used for: Course browsing/listing

2. **course_content** (Searchable chunks)
   - Documents: Text chunks
   - Metadata: `{course_title, lesson_number, lesson_title, chunk_index}`
   - Used for: Semantic search with lesson filtering

## System Prompt Strategy

The system prompt in `backend/ai_generator.py` is carefully designed to:
1. Guide tool usage ("maximum one search per query")
2. Prevent meta-commentary ("never mention that you used search")
3. Encourage direct answers for general questions
4. Handle missing information gracefully ("I don't have information about that")

When modifying the system prompt, maintain these constraints to ensure optimal user experience.
