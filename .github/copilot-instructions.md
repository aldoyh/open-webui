# Open WebUI AI Coding Agent Instructions

## Architecture Overview

**Open WebUI** is a full-stack AI chat platform with FastAPI backend and SvelteKit frontend that supports multiple LLM providers (Ollama, OpenAI-compatible APIs) with built-in RAG capabilities.

### Key Components
- **Backend**: FastAPI (`backend/open_webui/main.py`) with modular routers in `backend/open_webui/routers/`
- **Frontend**: SvelteKit SPA (`src/`) with component-based architecture
- **Database**: SQLAlchemy models with Alembic migrations
- **Vector DB**: ChromaDB/Qdrant for RAG with pluggable backends
- **Config System**: `PersistentConfig` pattern for environment + database configuration
- **Authentication**: JWT + OAuth2 with role-based permissions

## Development Workflow

### Local Development Setup
```bash
# Frontend development (port 5173)
npm run dev

# Backend development 
cd backend && python -m uvicorn open_webui.main:app --reload --host 0.0.0.0 --port 8080

# Full stack with Docker
docker compose up --build
```

### Build Process
- Frontend: `npm run build` creates static files in `build/`
- Backend: Multi-stage Dockerfile copies frontend build to serve via FastAPI
- **Critical**: Frontend build hash in `svelte.config.js` enables version polling for auto-reload

## Project-Specific Patterns

### Backend Architecture
- **Router Pattern**: Each domain has dedicated router (e.g., `routers/chats.py`, `routers/ollama.py`)
- **Dependency Injection**: `get_verified_user()`, `get_admin_user()` for auth
- **Config Access**: Use `request.app.state.config.SETTING_NAME` for runtime config
- **Model Loading**: Load balancing across multiple backend URLs with `get_ollama_url()` / `get_openai_url()`

### Frontend Patterns  
- **Stores**: Global state in `src/lib/stores/` (user, models, config, etc.)
- **API Layer**: Consistent fetch wrappers in `src/lib/apis/`
- **Components**: Atomic design with shared components in `src/lib/components/common/`
- **Routes**: SvelteKit file-based routing with `(app)` layout group
- **i18n**: Context-based translations with `getContext('i18n')`

### Configuration System
```python
# Backend: PersistentConfig for env + database persistence
OLLAMA_BASE_URLS = PersistentConfig("OLLAMA_BASE_URLS", "ollama.base_urls", [...])

# Access in routers
request.app.state.config.OLLAMA_BASE_URLS.value
```

### Model Provider Integration
- **Ollama**: Direct API proxy with connection pooling (`routers/ollama.py`)
- **OpenAI**: Compatible API adapter with key rotation (`routers/openai.py`) 
- **Pipelines**: External service integration via MCP protocol

### RAG Implementation
- **Knowledge Bases**: File ingestion → vector embeddings → retrieval
- **Loaders**: Pluggable document processors (`retrieval/loaders/`)
- **Search**: Hybrid vector + keyword search with reranking

## Critical Integration Points

### Authentication Flow
1. JWT tokens stored in `localStorage.token`
2. Socket.io connection with token-based auth
3. Role-based access control via `utils/access_control.py`

### Chat Message Flow
1. Frontend sends to `/api/chat` or `/ollama/api/chat`
2. Router validates user + applies filters
3. Backend selection via URL index
4. Response streaming with WebSocket events

### File Upload Pipeline
1. Upload via `routers/files.py` → Storage provider (local/S3/GCS)
2. Document processing via `routers/retrieval.py`
3. Vector embedding and storage
4. Knowledge base association

## Common Development Tasks

### Adding New Router
```python
# backend/open_webui/routers/new_feature.py
router = APIRouter()

@router.get("/")
async def get_items(user=Depends(get_verified_user)):
    return []

# Register in main.py
app.include_router(new_feature.router, prefix="/api/v1/new-feature", tags=["new-feature"])
```

### Adding Frontend API
```typescript
// src/lib/apis/new-feature/index.ts
export const getItems = async (token: string) => {
    return fetch(`${WEBUI_API_BASE_URL}/new-feature/`, {
        headers: { authorization: `Bearer ${token}` }
    }).then(res => res.json());
};
```

### Environment Configuration
- Backend: Add to `config.py` using `PersistentConfig`
- Frontend: Use `PUBLIC_` prefix for client-accessible vars
- Docker: Set in `docker-compose.yaml` or build args

## Testing Strategy
- **Backend**: pytest with database fixtures
- **Frontend**: Cypress E2E tests in `cypress/e2e/`
- **Integration**: Docker-based testing with test containers

## Key Files Reference
- `backend/open_webui/main.py`: FastAPI app setup and core endpoints
- `src/routes/(app)/+layout.svelte`: Main app layout with auth + state initialization  
- `backend/open_webui/config.py`: Configuration management system
- `src/lib/stores/index.ts`: Global state definitions
- `docker-compose.yaml`: Multi-service development setup