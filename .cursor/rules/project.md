# Project Rules - Alkeme Backend Template

## Project Type

This is a **Copier template repository** for generating production-ready FastAPI backend applications.

### What This Template Generates
- FastAPI + Pydantic v2 + Motor (async MongoDB)
- Azure AD authentication via fastapi-azure-auth
- Docker deployment with uv package manager
- Azure Container Apps + Cosmos DB deployment via Bicep IaC
- GitHub Actions CI/CD pipeline

### Critical Template Context
- **Files ending in `.jinja`** are Jinja2 templates
- Use `{{ variable }}` for variable substitution
- Use `{% if condition %}...{% endif %}` for conditionals
- Use `{% for item in list %}...{% endfor %}` for loops
- All generated code must be production-ready and deployable

## Architecture Pattern (Enforce Strictly)

### Layered Architecture - NOT MVT
```
Routers (HTTP layer) → Services (Business logic) → Clients (Data/External APIs)
```

This is a **hard requirement** for all generated applications.

### Layer Responsibilities

**Routers** (`backend/routers/`)
- Handle HTTP requests and responses
- Validate input with Pydantic models
- Use `Depends()` for dependency injection (DB, auth)
- Minimal logic - delegate all business logic to services
- Return appropriate HTTP status codes

```python
# Example: backend/routers/users.py
from fastapi import APIRouter, Depends, HTTPException, status, Security
from motor.motor_asyncio import AsyncIOMotorDatabase

router = APIRouter(prefix="/users", tags=["users"])

@router.post("/", response_model=UserPublic, status_code=status.HTTP_201_CREATED)
async def create_user(
    user_data: UserCreate,
    db: AsyncIOMotorDatabase = Depends(get_database),
    claims: dict = Security(azure_scheme),
) -> UserPublic:
    """Create a new user."""
    service = UserService(db)
    return await service.create_user(user_data)
```

**Services** (`backend/services/`)
- Pure business logic, NO FastAPI dependencies
- Orchestrate data operations
- Return/receive typed objects (RORO pattern)
- Raise `ValueError` or domain exceptions (routers convert to HTTP errors)

```python
# Example: backend/services/user_service.py
class UserService:
    def __init__(self, db: AsyncIOMotorDatabase):
        self.collection = db["users"]
    
    async def create_user(self, user_data: UserCreate) -> UserPublic:
        # Business validation
        if await self.collection.find_one({"email": user_data.email}):
            raise ValueError("Email already exists")
        
        user_dict = user_data.model_dump()
        user_dict["created_at"] = datetime.utcnow()
        result = await self.collection.insert_one(user_dict)
        user_dict["_id"] = result.inserted_id
        return UserPublic(**user_dict)
```

**Clients** (`backend/clients/`)
- Database operations (MongoDB/Cosmos DB)
- External API calls (Mailgun, etc.)
- No business logic, just data access

```python
# Example: backend/clients/mongo_db.py
from motor.motor_asyncio import AsyncIOMotorClient, AsyncIOMotorDatabase

_client: AsyncIOMotorClient | None = None
_database: AsyncIOMotorDatabase | None = None

def get_database() -> AsyncIOMotorDatabase:
    if _database is None:
        raise RuntimeError("Database not initialized")
    return _database
```

## FastAPI Application Structure

### Main Application Setup
```python
# backend/main.py
from contextlib import asynccontextmanager
from typing import AsyncGenerator
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    # Startup: connect DB, load config, seed data
    async with mongo_lifespan():
        if settings.AZURE_TENANT_ID:
            await azure_scheme.openid_config.load_config()
        await ensure_indexes()
        yield
    # Shutdown: cleanup via context managers

app = FastAPI(
    title="API Title",
    version="0.1.0",
    lifespan=lifespan,
)

# Include routers
app.include_router(users.router)
app.include_router(projects.router)

# CORS middleware (specific origins only)
if settings.BACKEND_CORS_ORIGINS:
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.get_cors_origins(),
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

# Health check (no auth required)
@app.get("/health")
async def health():
    return {"status": "healthy"}
```

### Dependency Injection Pattern
```python
# Use Depends() - NEVER global state
from backend.clients.mongo_db import get_database
from backend.auth import azure_scheme

async def endpoint(
    db: AsyncIOMotorDatabase = Depends(get_database),
    claims: dict = Security(azure_scheme),
):
    pass
```

## File Organization

### Directory Structure
```
backend/
├── __init__.py
├── main.py                    # FastAPI app + lifespan
├── config.py                  # Pydantic Settings
├── auth.py                    # Azure AD setup (if enabled)
├── dependencies.py            # Reusable dependencies
├── routers/                   # API endpoints
│   ├── users.py
│   ├── projects.py
│   └── kanban.py (conditional)
├── services/                  # Business logic
│   ├── user_service.py
│   └── project_service.py
├── models/                    # Pydantic schemas
│   ├── user.py
│   └── project.py
├── clients/                   # Data access
│   ├── mongo_db.py
│   └── mailgun.py (conditional)
└── utils/                     # Utilities
    ├── seed.py
    ├── indexes.py
    └── logging_config.py
```

### Naming Conventions (User Preference)
- **Files/modules**: `lowercase_with_underscores` (user_service.py, mongo_db.py)
- **Functions/variables**: `lowercase_with_underscores` (get_user_by_id, is_active)
- **Classes**: `PascalCase` (UserService, ProjectModel)
- **Constants**: `UPPER_CASE_WITH_UNDERSCORES` (MAX_RETRIES, DEFAULT_TIMEOUT)

## Configuration Pattern

### Pydantic Settings
```python
# backend/config.py
from pydantic import Field, computed_field
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        case_sensitive=True,
    )
    
    MONGODB_URI: str = Field(default="mongodb://localhost:27017")
    BACKEND_CORS_ORIGINS: str = "http://localhost:3000"
    AZURE_TENANT_ID: str | None = None
    AZURE_CLIENT_ID: str | None = None
    DEV_NO_AUTH: bool = False
    
    def get_cors_origins(self) -> list[str]:
        return [o.strip() for o in self.BACKEND_CORS_ORIGINS.split(",")]
    
    @computed_field
    @property
    def SCOPE_NAME(self) -> str:
        return f"api://{self.AZURE_CLIENT_ID}/user_impersonation" if self.AZURE_CLIENT_ID else ""

settings = Settings()  # Singleton
```

## Database Strategy

### Local Development vs Production
- **Local dev**: MongoDB Docker container (always included in docker-compose.yml)
- **Production**: Azure Cosmos DB (MongoDB API) via Bicep deployment
- **Same code works with both**: Only connection string changes
- **Compatibility**: Must use PyMongo 3.x (NOT 4.x) for Cosmos DB

### Connection Pattern
```python
# backend/clients/mongo_db.py
@asynccontextmanager
async def mongo_lifespan() -> AsyncGenerator[None, None]:
    global _client, _database
    _client = AsyncIOMotorClient(settings.MONGODB_URI)
    _database = _client.get_default_database()
    await _client.admin.command("ping")
    yield
    if _client:
        _client.close()
```

## Docker & Deployment

### Multi-Stage Dockerfile with uv
```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
RUN pip install uv
COPY pyproject.toml ./
RUN uv sync --no-dev

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/.venv /app/.venv
COPY backend/ ./backend/
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser
ENV PATH="/app/.venv/bin:$PATH"
CMD ["uvicorn", "backend.main:app", "--host", "0.0.0.0", "--port", "80"]
```

### Docker Compose (Local Development)
```yaml
services:
  backend:
    build: .
    ports:
      - "8000:80"
    environment:
      MONGODB_URI: mongodb://mongodb:27017/mydb
    depends_on:
      - mongodb
  
  mongodb:
    image: mongo:7
    ports:
      - "27017:27017"
```

### Azure Deployment via Bicep
- Container Registry (ACR) for Docker images
- Container Apps for serverless container hosting
- Cosmos DB (MongoDB API) for database
- Log Analytics + Application Insights for monitoring
- Key Vault for secrets (optional)

## Error Handling Pattern

### Guard Clauses (Preferred)
```python
async def get_user(user_id: str):
    # Validate early, return/raise immediately
    if not ObjectId.is_valid(user_id):
        raise HTTPException(status_code=400, detail="Invalid ID format")
    
    user = await service.get_user_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    
    if not user.is_active:
        raise HTTPException(status_code=403, detail="User inactive")
    
    return user  # Happy path unindented
```

## Template Quality Standards

### Production-Ready Code
All generated code must:
- Pass `ruff check`, `ruff format`, `mypy`, `pytest`
- Have >80% test coverage
- Include comprehensive docstrings
- Follow all architecture patterns
- Be deployable to Azure without modification

### Jinja Template Guidelines
- Keep template logic minimal
- Use clear variable names
- Document conditional features
- Provide sensible defaults
- Test all feature flag combinations

## References

- **Project Documentation**: See `project.md` and `project-spec.md`
- **Python Rules**: See `.cursor/rules/python.md`
- **General Guidelines**: See `.cursor/rules/general.md`
- **Detailed Patterns**: See `.cursorrules` at root

