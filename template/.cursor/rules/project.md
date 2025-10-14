# Project Rules - Generated Backend Application

## Architecture (CRITICAL - ALWAYS FOLLOW)

**Strict layered architecture:**

```
Routers (HTTP) → Services (Business Logic) → Clients (Data/External APIs)
```

### Layer Responsibilities

**Routers** (`backend/routers/`):
- Handle HTTP requests/responses ONLY
- Validate input with Pydantic models
- Use FastAPI `Depends()` for dependencies
- Delegate business logic to services
- Return responses or raise HTTPException
- **NO business logic** in routers

**Services** (`backend/services/`):
- Pure business logic (NO FastAPI imports)
- Orchestrate operations, apply business rules
- Return typed Python objects (Pydantic models)
- Raise standard Python exceptions (not HTTPException)
- Independent of HTTP layer

**Clients** (`backend/clients/`):
- Database operations (Motor)
- External API calls
- No business logic
- Return raw data or domain objects

### Example Flow

```python
# ✅ Correct layered architecture

# Router (backend/routers/users.py)
@router.get("/users/{user_id}", response_model=UserPublic)
async def get_user(user_id: str):
    # Validate ID format
    if not ObjectId.is_valid(user_id):
        raise HTTPException(status_code=400, detail="Invalid user ID")
    
    # Delegate to service
    user = await user_service.get_user_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    
    return user

# Service (backend/services/user_service.py)
async def get_user_by_id(user_id: str) -> UserPublic | None:
    """Get user by ID - business logic layer."""
    # Use client to fetch data
    user_data = await user_client.find_by_id(user_id)
    if not user_data:
        return None
    
    # Apply business logic, transform data
    return UserPublic(**user_data)

# Client (backend/clients/user_client.py)
async def find_by_id(user_id: str) -> dict | None:
    """Find user in database - data access layer."""
    return await db.users.find_one({"_id": ObjectId(user_id)})
```

## FastAPI Application Structure

### Main Application

```python
# backend/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await database.connect()
    await ensure_indexes()
    yield
    # Shutdown
    await database.disconnect()

app = FastAPI(
    title="API Name",
    lifespan=lifespan
)

# Include routers
app.include_router(users_router, prefix="/users", tags=["users"])
```

### Configuration

```python
# backend/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    # Application
    APP_NAME: str = "Backend API"
    DEBUG: bool = False
    
    # Database
    MONGODB_URI: str
    MONGODB_DB_NAME: str
    
    # CORS
    BACKEND_CORS_ORIGINS: list[str] = ["http://localhost:3000"]
    
    model_config = SettingsConfigDict(
        env_file=".env",
        case_sensitive=True
    )

settings = Settings()
```

## Error Handling

### Guard Clauses Pattern

```python
# ✅ Always use guard clauses - validate early, return early
async def get_user(user_id: str) -> UserPublic:
    # Guard 1: Validate input
    if not ObjectId.is_valid(user_id):
        raise HTTPException(status_code=400, detail="Invalid user ID")
    
    # Guard 2: Check existence
    user = await service.get_user_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    
    # Guard 3: Check permissions
    if not user.is_active:
        raise HTTPException(status_code=403, detail="User is inactive")
    
    # Happy path - unindented
    return user
```

### HTTP Exceptions (Routers Only)

```python
from fastapi import HTTPException, status

# Use proper status codes
raise HTTPException(status_code=400, detail="Invalid input")
raise HTTPException(status_code=401, detail="Not authenticated")
raise HTTPException(status_code=403, detail="Not authorized")
raise HTTPException(status_code=404, detail="Not found")
raise HTTPException(status_code=409, detail="Already exists")
raise HTTPException(status_code=422, detail="Validation error")
```

## Database Strategy

### MongoDB Connection

```python
# backend/clients/mongodb.py
from motor.motor_asyncio import AsyncIOMotorClient

mongodb_client: AsyncIOMotorClient | None = None

async def connect_to_mongo():
    global mongodb_client
    mongodb_client = AsyncIOMotorClient(settings.MONGODB_URI)
    logger.info("Connected to MongoDB")

async def close_mongo_connection():
    global mongodb_client
    if mongodb_client:
        mongodb_client.close()
        logger.info("Closed MongoDB connection")

def get_database():
    if not mongodb_client:
        raise RuntimeError("MongoDB client not initialized")
    return mongodb_client[settings.MONGODB_DB_NAME]
```

### Indexes

```python
# backend/utils/indexes.py
async def ensure_indexes():
    db = get_database()
    
    # Users collection
    await db.users.create_index("email", unique=True)
    await db.users.create_index("created_at")
    
    # Projects collection
    await db.projects.create_index([("user_id", 1), ("created_at", -1)])
    await db.projects.create_index("name", unique=True)
```

## Dependency Injection

```python
# Use FastAPI Depends() for shared resources

from fastapi import Depends
from backend.clients.mongodb import get_database

# Database dependency
def get_db():
    return get_database()

# Current user dependency
async def get_current_user(user_id: str = Depends(get_user_id_from_token)):
    user = await user_service.get_user_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

# Use in routes
@router.get("/me")
async def get_me(current_user: UserPublic = Depends(get_current_user)):
    return current_user
```

## Common Patterns

### CRUD Operations

```python
# Create
@router.post("/users", response_model=UserPublic, status_code=201)
async def create_user(user: UserCreate):
    return await service.create_user(user)

# Read
@router.get("/users/{user_id}", response_model=UserPublic)
async def get_user(user_id: str):
    return await service.get_user_by_id(user_id)

# Update
@router.patch("/users/{user_id}", response_model=UserPublic)
async def update_user(user_id: str, user: UserUpdate):
    return await service.update_user(user_id, user)

# Delete
@router.delete("/users/{user_id}", status_code=204)
async def delete_user(user_id: str):
    await service.delete_user(user_id)
    return None
```

### Pagination

```python
from pydantic import BaseModel

class PaginatedResponse(BaseModel):
    items: list
    total: int
    page: int
    page_size: int

@router.get("/users", response_model=PaginatedResponse)
async def list_users(page: int = 1, page_size: int = 10):
    skip = (page - 1) * page_size
    items = await service.list_users(skip=skip, limit=page_size)
    total = await service.count_users()
    return PaginatedResponse(items=items, total=total, page=page, page_size=page_size)
```

## Development Commands

```bash
# Run development server
uv run uvicorn backend.main:app --reload

# Run tests
uv run pytest

# Run with coverage
uv run pytest --cov=backend

# Type check
uv run mypy backend

# Lint and format
uv run ruff format backend
uv run ruff check backend --fix

# Docker development
docker compose up --build
```

## Common Mistakes to Avoid

❌ Business logic in routers  
❌ FastAPI imports in services  
❌ Database operations in routers  
❌ Not using dependency injection  
❌ Not validating ObjectId before queries  
❌ Missing type hints on functions  
❌ Not using guard clauses  
❌ Nested if statements instead of early returns  

## Remember

1. **Architecture**: Routers → Services → Clients (ALWAYS)
2. **Routers**: HTTP only, no business logic
3. **Services**: Business logic only, no FastAPI
4. **Clients**: Data access only
5. **Guard clauses**: Validate early, return early
6. **Dependencies**: Use `Depends()` for shared resources
7. **Type hints**: Required on ALL functions

