# Python Rules - Alkeme Backend Template

## Python Version & Style

### Version Requirements
- **Python 3.10+** (use modern syntax)
- Use `from __future__ import annotations` for forward references

### Code Style (PEP 8 Strict)
- Line length: **100 characters** (configured in ruff)
- Use **double quotes** for strings
- Use trailing commas in multi-line structures
- Import order: stdlib → third-party → local (ruff auto-sorts)

```python
# Good
from __future__ import annotations

from datetime import datetime
from typing import AsyncGenerator

from fastapi import APIRouter, Depends
from motor.motor_asyncio import AsyncIOMotorDatabase

from backend.config import settings
from backend.models.user import UserCreate, UserPublic
```

## Type Hints (Required on ALL Functions)

### Function Signatures
Every function must have type hints for parameters AND return types:

```python
# ✅ Correct
async def get_user(user_id: str) -> UserPublic | None:
    pass

def calculate_total(items: list[dict]) -> float:
    pass

async def process_batch(items: list[str]) -> list[dict[str, Any]]:
    pass

# ❌ Wrong - missing type hints
async def get_user(user_id):  # Missing types
    pass
```

### Modern Type Syntax (Python 3.10+)
```python
# ✅ Prefer modern syntax
list[str]           # instead of List[str]
dict[str, int]      # instead of Dict[str, int]
str | None          # instead of Optional[str]
int | str           # instead of Union[int, str]

# Import from collections.abc for protocols
from collections.abc import Sequence, Mapping, Iterable
```

## Naming Conventions (Strict User Preference)

### Files & Modules
- Use `lowercase_with_underscores`
- Be descriptive and explicit
- Examples: `user_service.py`, `mongo_db.py`, `mailgun_client.py`

### Functions & Variables
- Use `lowercase_with_underscores`
- Verb phrases for functions: `get_user_by_id`, `validate_email`, `send_notification`
- Nouns for variables: `user_data`, `email_address`, `retry_count`

### Booleans (Special Prefix Convention)
- Always prefix with `is_`, `has_`, `can_`, `should_`
- Examples: `is_active`, `has_permission`, `can_edit`, `should_retry`

```python
# ✅ Good
is_authenticated = True
has_valid_email = check_email(email)
can_delete_project = user.role == "admin"

# ❌ Wrong
authenticated = True  # Missing is_ prefix
valid_email = check_email(email)  # Missing has_ prefix
```

### Classes
- Use `PascalCase`
- Descriptive names: `UserService`, `ProjectModel`, `EmailValidator`

### Constants
- Use `UPPER_CASE_WITH_UNDERSCORES`
- Examples: `MAX_RETRIES = 3`, `DEFAULT_TIMEOUT = 30`, `API_VERSION = "v1"`

### Private/Internal
- Prefix with single underscore `_`
- Examples: `_validate_input`, `_client`, `_database`

## Async Programming (Critical)

### When to Use async def
Use `async def` for **ALL I/O operations**:
- Database queries (MongoDB, Cosmos DB)
- HTTP requests (external APIs)
- File I/O operations
- Any operation that waits for external resources

```python
# ✅ Correct - async for I/O
async def get_user(user_id: str) -> UserPublic | None:
    user = await db.users.find_one({"_id": ObjectId(user_id)})
    return UserPublic(**user) if user else None

async def send_email(email: str, content: str) -> bool:
    async with httpx.AsyncClient() as client:
        response = await client.post(MAILGUN_URL, data={"to": email, "html": content})
        return response.status_code == 200

# ✅ Use def for CPU-bound operations
def calculate_hash(data: str) -> str:
    return hashlib.sha256(data.encode()).hexdigest()

def format_date(dt: datetime) -> str:
    return dt.strftime("%Y-%m-%d")
```

### Never Block the Event Loop
```python
# ❌ WRONG - blocking I/O in async code
async def bad_example():
    import requests  # DON'T use sync libraries
    response = requests.get(url)  # Blocks event loop!

# ✅ Correct - use async libraries
async def good_example():
    import httpx
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
```

### Parallel Operations
```python
# ✅ Use asyncio.gather() for parallel operations
results = await asyncio.gather(
    db.users.find_one({"_id": user_id}),
    db.projects.find({"owner_id": user_id}).to_list(10),
    external_api.get_user_stats(user_id),
)
user, projects, stats = results
```

## Pydantic v2 Models

### Model Suffixes (Required Pattern)
Use descriptive suffixes to indicate model purpose:

```python
from pydantic import BaseModel, EmailStr, Field, ConfigDict

# For POST requests - creating new resources
class UserCreate(BaseModel):
    email: EmailStr
    name: str = Field(..., min_length=1, max_length=100)
    password: str = Field(..., min_length=8)

# For PUT/PATCH requests - updating resources
class UserUpdate(BaseModel):
    email: EmailStr | None = None
    name: str | None = None
    is_active: bool | None = None

# Database representation (includes _id, timestamps, etc.)
class UserInDB(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    
    id: str = Field(alias="_id")
    email: EmailStr
    name: str
    password_hash: str
    is_active: bool
    created_at: datetime
    updated_at: datetime | None = None

# Public API response (excludes sensitive fields)
class UserPublic(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,
        populate_by_name=True,
    )
    
    id: str = Field(alias="_id")
    email: EmailStr
    name: str
    is_active: bool
    created_at: datetime
```

### Pydantic v2 Syntax
```python
# ✅ Use v2 methods
user_dict = user.model_dump()  # NOT dict()
user_json = user.model_dump_json()  # NOT json()

# ✅ Use model_config instead of Config class
class User(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,
        populate_by_name=True,
        str_strip_whitespace=True,
    )
```

### Field Validation
```python
from pydantic import Field, field_validator

class User(BaseModel):
    email: EmailStr
    age: int = Field(..., ge=0, le=150)
    name: str = Field(..., min_length=1, max_length=100)
    
    @field_validator("email")
    @classmethod
    def email_must_be_lowercase(cls, v: str) -> str:
        return v.lower()
```

## MongoDB / Motor Patterns

### Motor (Async MongoDB Driver)
**CRITICAL**: Use PyMongo 3.x (NOT 4.x) for Azure Cosmos DB compatibility

```python
from motor.motor_asyncio import AsyncIOMotorClient, AsyncIOMotorDatabase
from bson import ObjectId

# Always validate ObjectId before querying
if not ObjectId.is_valid(user_id):
    raise HTTPException(status_code=400, detail="Invalid ID format")

# Async operations
user = await db.users.find_one({"_id": ObjectId(user_id)})
users = await db.users.find({"is_active": True}).skip(skip).limit(limit).to_list(limit)

# Updates
await db.users.update_one(
    {"_id": ObjectId(user_id)},
    {"$set": update_dict}
)

# Use projections to limit fields
user = await db.users.find_one(
    {"_id": ObjectId(user_id)},
    {"password_hash": 0}  # Exclude password
)
```

### Indexes
```python
# backend/utils/indexes.py
async def ensure_indexes(db: AsyncIOMotorDatabase) -> None:
    """Create database indexes on startup."""
    # Unique indexes
    await db.users.create_index("email", unique=True)
    
    # Compound indexes
    await db.projects.create_index([("owner_id", 1), ("created_at", -1)])
    
    # Text search indexes
    await db.projects.create_index([("name", "text"), ("description", "text")])
```

## Functions & Methods

### Keep Functions Small
- Ideal: < 50 lines
- Maximum: < 100 lines
- If longer, break into smaller functions

### Early Returns & Guard Clauses
```python
# ✅ Good - guard clauses, early returns
async def get_user(user_id: str) -> UserPublic:
    if not ObjectId.is_valid(user_id):
        raise HTTPException(status_code=400, detail="Invalid ID")
    
    user = await service.get_user_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="Not found")
    
    if not user.is_active:
        raise HTTPException(status_code=403, detail="Inactive")
    
    return user  # Happy path unindented

# ❌ Bad - deeply nested
async def get_user(user_id: str) -> UserPublic | None:
    if ObjectId.is_valid(user_id):
        user = await service.get_user_by_id(user_id)
        if user:
            if user.is_active:
                return user
            else:
                raise HTTPException(status_code=403)
        else:
            raise HTTPException(status_code=404)
    else:
        raise HTTPException(status_code=400)
```

### RORO Pattern (Receive Object, Return Object)
```python
# ✅ Good - typed objects in/out
async def create_user(user_data: UserCreate) -> UserPublic:
    pass

# ❌ Bad - primitives
async def create_user(email: str, name: str, password: str) -> dict:
    pass
```

## Error Handling

### Use Specific Exceptions
```python
# ✅ Good - specific exceptions
try:
    result = await operation()
except ValueError as e:
    logger.error(f"Invalid value: {e}")
    raise HTTPException(status_code=400, detail=str(e))
except ConnectionError as e:
    logger.error(f"DB connection failed: {e}")
    raise HTTPException(status_code=503, detail="Service unavailable")

# ❌ Bad - bare except
try:
    result = await operation()
except:  # Too broad!
    pass
```

### FastAPI HTTP Exceptions
```python
from fastapi import HTTPException, status

# Use appropriate status codes
raise HTTPException(status_code=400, detail="Invalid input")
raise HTTPException(status_code=401, detail="Unauthorized")
raise HTTPException(status_code=403, detail="Forbidden")
raise HTTPException(status_code=404, detail="Not found")
raise HTTPException(status_code=422, detail="Validation error")
raise HTTPException(status_code=500, detail="Internal server error")
```

## Documentation

### Docstrings (Required for Public Functions)
```python
def get_user_by_id(user_id: str) -> UserPublic | None:
    """Get user by ID.
    
    Args:
        user_id: MongoDB ObjectId as string
        
    Returns:
        UserPublic model if found, None otherwise
        
    Raises:
        ValueError: If user_id is not a valid ObjectId
    """
    pass
```

### Inline Comments
- Explain **why**, not **what** (code should be self-explanatory)
- Document complex algorithms
- Clarify non-obvious business rules

```python
# ✅ Good - explains why
# We use PyMongo 3.x because Cosmos DB only supports wire protocol 6
client = AsyncIOMotorClient(uri)

# ❌ Bad - states the obvious
# Create a user
user = User(...)
```

## Testing Patterns

### Test Structure
```python
import pytest
from fastapi.testclient import TestClient

# Arrange, Act, Assert pattern
@pytest.mark.asyncio
async def test_create_user(test_db):
    # Arrange
    service = UserService(test_db)
    user_data = UserCreate(
        email="test@example.com",
        name="Test User",
        password="secure123",
    )
    
    # Act
    user = await service.create_user(user_data)
    
    # Assert
    assert user.email == "test@example.com"
    assert user.is_active is True
```

### Use Descriptive Test Names
```python
# ✅ Good - describes what is tested and expected outcome
async def test_create_user_with_duplicate_email_raises_value_error():
    pass

async def test_get_user_by_invalid_id_returns_none():
    pass

# ❌ Bad - vague
async def test_user():
    pass
```

## Performance

### Database Optimization
```python
# ✅ Use projections to limit returned fields
user = await db.users.find_one(
    {"_id": ObjectId(user_id)},
    {"password_hash": 0, "internal_notes": 0}
)

# ✅ Use pagination
async def list_users(skip: int = 0, limit: int = 10) -> list[UserPublic]:
    cursor = db.users.find().skip(skip).limit(limit)
    users = await cursor.to_list(length=limit)
    return [UserPublic(**u) for u in users]
```

### Caching
```python
from functools import lru_cache

# ✅ Cache expensive computations
@lru_cache(maxsize=128)
def calculate_hash(data: str) -> str:
    return hashlib.sha256(data.encode()).hexdigest()
```

## Common Python Mistakes to Avoid

❌ Missing type hints on functions  
❌ Using sync I/O in async functions  
❌ Not validating ObjectId strings  
❌ Bare `except:` clauses  
❌ Deeply nested if statements (use guard clauses)  
❌ Mutable default arguments: `def func(items=[]):`  
❌ Not using `async with` for async context managers  
❌ Forgetting `await` on async calls  
❌ Using PyMongo 4.x (breaks Cosmos DB)  

## Tools & Commands

### Code Quality
```bash
# Format code
uv run ruff format backend

# Lint code
uv run ruff check backend --fix

# Type check
uv run mypy backend

# Run tests
uv run pytest

# Coverage
uv run pytest --cov=backend --cov-report=term-missing
```

## References

- **PEP 8**: https://peps.python.org/pep-0008/
- **Type Hints (PEP 484)**: https://peps.python.org/pep-0484/
- **Pydantic v2**: https://docs.pydantic.dev/latest/
- **Motor**: https://motor.readthedocs.io/
- **FastAPI**: https://fastapi.tiangolo.com/

