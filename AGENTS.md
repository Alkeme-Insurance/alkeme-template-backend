# AI Agent Rules - Alkeme Backend Template

You are an expert in Python, FastAPI, async programming, MongoDB/Motor, Azure deployment, and modern backend development practices.

## Project Context

This is a **Copier template repository** that generates production-ready FastAPI backend applications with:
- FastAPI + Pydantic v2 + Motor (async MongoDB)
- Azure AD authentication via fastapi-azure-auth
- Docker deployment with uv package manager
- Azure Container Apps + Cosmos DB deployment via Bicep IaC
- Local dev: MongoDB Docker container → Production: Azure Cosmos DB

**Critical Template Context:**
- Files ending in `.jinja` are Jinja2 templates
- Use `{{ variable }}` for substitution, `{% if %}` for conditionals
- All generated code must be production-ready

## Rules Organization

Comprehensive rules are organized in `.cursor/rules/` for maintainability:

### 📋 [Project Rules](.cursor/rules/project.md)
Architecture, FastAPI patterns, deployment strategy
- Layered architecture (Routers → Services → Clients)
- FastAPI application structure and patterns
- Docker & Azure deployment configurations

### 🐍 [Python Rules](.cursor/rules/python.md)
Code style, type hints, async patterns, Pydantic models
- Python 3.10+ with type hints on ALL functions
- Naming conventions (user preferences)
- Async-first programming patterns

### 🛡️ [General Rules](.cursor/rules/general.md)
Security, testing, git workflow, documentation
- Security best practices (never log secrets)
- Code quality standards (pre-commit checklist)
- Development workflow and testing patterns

## Quick Critical Rules

### Architecture (Enforce Strictly)
```
Routers (HTTP) → Services (Business Logic) → Clients (Data/External APIs)
```

**Routers** (`backend/routers/`): HTTP only, validate input, use `Depends()`, minimal logic

**Services** (`backend/services/`): Pure business logic, NO FastAPI imports, return typed objects

**Clients** (`backend/clients/`): Database operations, external APIs, no business logic

### Code Style (Required)

```python
# Type hints on ALL functions
async def get_user(user_id: str) -> UserPublic | None:
    pass

# Naming conventions
user_service.py          # Files: lowercase_with_underscores
def get_user_by_id():   # Functions: lowercase_with_underscores
class UserService:       # Classes: PascalCase
is_active = True        # Booleans: is_, has_, can_ prefix
MAX_RETRIES = 3         # Constants: UPPER_CASE
```

### Async Programming (Critical)

```python
# ✅ Use async def for ALL I/O operations
async def get_user(user_id: str):
    user = await db.users.find_one({"_id": ObjectId(user_id)})
    return user

# ❌ NEVER block the event loop
import requests  # Don't use sync libraries in async code
```

### Pydantic Models (Required Pattern)

Use descriptive suffixes: `UserCreate`, `UserUpdate`, `UserInDB`, `UserPublic`

```python
from pydantic import BaseModel, EmailStr, Field, ConfigDict

class UserCreate(BaseModel):
    """For POST requests."""
    email: EmailStr
    name: str = Field(..., min_length=1, max_length=100)

class UserPublic(BaseModel):
    """API response (no sensitive fields)."""
    id: str = Field(alias="_id")
    email: EmailStr
    name: str
    
    model_config = ConfigDict(from_attributes=True, populate_by_name=True)
```

### MongoDB/Motor (Critical)

**Must use PyMongo 3.x (NOT 4.x)** for Azure Cosmos DB compatibility

```python
# Always validate ObjectId before querying
if not ObjectId.is_valid(user_id):
    raise HTTPException(status_code=400, detail="Invalid ID")

# Async operations with Motor
user = await db.users.find_one({"_id": ObjectId(user_id)})
```

### Security (Never Break)

```python
# ❌ NEVER log secrets
logger.info(f"Connecting to {settings.MONGODB_URI}")  # Contains password!

# ✅ Correct - log actions, not secrets
logger.info("Connecting to database")
```

**Critical Security Rules:**
- Never log secrets, tokens, passwords, connection strings
- Validate ALL inputs with Pydantic models
- No `*` in CORS origins (production)
- Use Azure Key Vault for production secrets

### Error Handling (Guard Clauses)

```python
async def get_user(user_id: str):
    # Validate early, return immediately
    if not ObjectId.is_valid(user_id):
        raise HTTPException(status_code=400, detail="Invalid ID")
    
    user = await service.get_user_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="Not found")
    
    return user  # Happy path unindented
```

### FastAPI Application Setup

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    async with mongo_lifespan():
        if settings.AZURE_TENANT_ID:
            await azure_scheme.openid_config.load_config()
        await ensure_indexes()
        yield

app = FastAPI(lifespan=lifespan)

# Health check (no auth required)
@app.get("/health")
async def health():
    return {"status": "healthy"}
```

### Database Strategy

- **Local dev**: MongoDB Docker container (always in docker-compose.yml)
- **Production**: Azure Cosmos DB (MongoDB API) via Bicep
- **Compatibility**: PyMongo 3.x required for Cosmos DB wire protocol 6

## Code Quality Checklist

Before every commit:
```bash
uv run ruff format backend
uv run ruff check backend --fix
uv run mypy backend
uv run pytest --cov=backend
```

## Common Mistakes to Avoid

❌ Using sync I/O in async functions  
❌ Not validating ObjectId strings  
❌ Logging secrets or connection strings  
❌ Using global state instead of `Depends()`  
❌ Business logic in routers  
❌ PyMongo 4.x (breaks Cosmos DB)  
❌ Committing `.env` files  
❌ `*` in CORS origins (production)  

## Tools

- **uv**: Fast Python package manager (10-100x faster than pip)
- **ruff**: Linter & formatter (replaces black, isort, flake8)
- **mypy**: Static type checker
- **pytest**: Testing framework with pytest-asyncio
- **detect-secrets**: Pre-commit secret scanning

## References

For comprehensive guidance, see:
- **Project Architecture**: [.cursor/rules/project.md](.cursor/rules/project.md)
- **Python Style Guide**: [.cursor/rules/python.md](.cursor/rules/python.md)
- **Security & Workflow**: [.cursor/rules/general.md](.cursor/rules/general.md)
- **Rules Organization**: [.cursor/rules/README.md](.cursor/rules/README.md)

---

**Remember**: This template generates production-ready code. All files should be clean, well-typed, and deployable to Azure Container Apps with minimal configuration.

