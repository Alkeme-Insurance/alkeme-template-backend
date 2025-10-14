# Python Code Style & Implementation Rules

## Python Version & Standards

- **Python 3.10+** required
- Follow **PEP 8** style guide
- Use **type hints** on ALL function signatures
- Write **docstrings** for public functions and classes

## Type Hints (REQUIRED)

### All Functions Must Have Type Hints

```python
# ✅ Correct
async def get_user(user_id: str) -> UserPublic | None:
    """Get user by ID."""
    user = await db.users.find_one({"_id": ObjectId(user_id)})
    return UserPublic(**user) if user else None

def calculate_total(items: list[dict]) -> float:
    """Calculate total price."""
    return sum(item["price"] for item in items)

# ❌ Wrong - missing type hints
async def get_user(user_id):
    return await db.users.find_one({"_id": ObjectId(user_id)})
```

### Modern Type Hints

```python
# Use | for unions (Python 3.10+)
def get_value() -> str | None:
    pass

# Use list[], dict[], set[] (Python 3.9+)
def process(items: list[str]) -> dict[str, int]:
    pass

# Use typing for complex types
from typing import Literal, TypedDict

Status = Literal["pending", "active", "inactive"]

class UserDict(TypedDict):
    name: str
    email: str
    age: int
```

## Naming Conventions

### Files and Modules

```python
user_service.py          # Files: lowercase_with_underscores
user_repository.py
email_client.py
```

### Functions

```python
def get_user_by_id():    # Functions: lowercase_with_underscores
def create_new_project():
def validate_email_format():
```

### Classes

```python
class UserService:       # Classes: PascalCase
class EmailClient:
class DatabaseManager:
```

### Variables

```python
# Regular variables: lowercase_with_underscores
user_id = "123"
project_name = "My Project"
total_count = 42

# Booleans: descriptive prefixes
is_active = True
has_permission = False
can_edit = True
should_retry = False

# Constants: UPPER_CASE
MAX_RETRIES = 3
DEFAULT_TIMEOUT = 30
API_VERSION = "v1"

# Private: _leading_underscore
_internal_cache = {}
_helper_function = lambda x: x * 2
```

## Async Programming (CRITICAL)

### Use async def for I/O Operations

```python
# ✅ Correct - async for I/O
async def get_user(user_id: str) -> UserPublic | None:
    user = await db.users.find_one({"_id": ObjectId(user_id)})
    return UserPublic(**user) if user else None

async def fetch_from_api(url: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.json()

# ✅ Correct - regular def for CPU-bound
def calculate_hash(data: str) -> str:
    return hashlib.sha256(data.encode()).hexdigest()

def process_data(items: list[dict]) -> list[dict]:
    return [transform(item) for item in items]
```

### Never Block the Event Loop

```python
# ❌ WRONG - blocks event loop
import requests
async def fetch_data(url: str):
    response = requests.get(url)  # Blocks!
    return response.json()

# ✅ Correct - use async libraries
import httpx
async def fetch_data(url: str):
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.json()
```

## Pydantic v2 Models (REQUIRED)

### Model Naming Conventions

Use descriptive suffixes:
- `Create` - POST requests
- `Update` - PUT/PATCH requests
- `InDB` - Database representation
- `Public` - API responses (no sensitive fields)

```python
from pydantic import BaseModel, EmailStr, Field, ConfigDict
from datetime import datetime

class UserCreate(BaseModel):
    """For POST /users - creating new user."""
    email: EmailStr
    name: str = Field(..., min_length=1, max_length=100)
    password: str = Field(..., min_length=8)

class UserUpdate(BaseModel):
    """For PATCH /users/{id} - partial updates."""
    name: str | None = None
    email: EmailStr | None = None

class UserInDB(BaseModel):
    """Database representation - includes all fields."""
    id: str = Field(alias="_id")
    email: EmailStr
    name: str
    hashed_password: str
    is_active: bool = True
    created_at: datetime
    updated_at: datetime | None = None
    
    model_config = ConfigDict(
        from_attributes=True,
        populate_by_name=True
    )

class UserPublic(BaseModel):
    """API response - excludes sensitive fields."""
    id: str = Field(alias="_id")
    email: EmailStr
    name: str
    is_active: bool
    created_at: datetime
    
    model_config = ConfigDict(
        from_attributes=True,
        populate_by_name=True
    )
```

### Validation

```python
from pydantic import field_validator, model_validator

class UserCreate(BaseModel):
    email: EmailStr
    password: str
    password_confirm: str
    
    @field_validator('password')
    @classmethod
    def validate_password(cls, v: str) -> str:
        if len(v) < 8:
            raise ValueError('Password must be at least 8 characters')
        if not any(c.isupper() for c in v):
            raise ValueError('Password must contain uppercase letter')
        return v
    
    @model_validator(mode='after')
    def check_passwords_match(self) -> 'UserCreate':
        if self.password != self.password_confirm:
            raise ValueError('Passwords do not match')
        return self
```

## MongoDB/Motor Patterns

### Always Validate ObjectId

```python
from bson import ObjectId
from fastapi import HTTPException

# ✅ Always validate before use
if not ObjectId.is_valid(user_id):
    raise HTTPException(status_code=400, detail="Invalid ID format")

user = await db.users.find_one({"_id": ObjectId(user_id)})
```

### Async Motor Operations

```python
from motor.motor_asyncio import AsyncIOMotorDatabase

async def find_user(db: AsyncIOMotorDatabase, user_id: str) -> dict | None:
    """Find user by ID."""
    return await db.users.find_one({"_id": ObjectId(user_id)})

async def find_users(db: AsyncIOMotorDatabase, is_active: bool = True) -> list[dict]:
    """Find multiple users."""
    cursor = db.users.find({"is_active": is_active})
    return await cursor.to_list(length=100)

async def create_user(db: AsyncIOMotorDatabase, user: dict) -> str:
    """Insert user and return ID."""
    result = await db.users.insert_one(user)
    return str(result.inserted_id)

async def update_user(db: AsyncIOMotorDatabase, user_id: str, updates: dict) -> bool:
    """Update user."""
    result = await db.users.update_one(
        {"_id": ObjectId(user_id)},
        {"$set": updates}
    )
    return result.modified_count > 0

async def delete_user(db: AsyncIOMotorDatabase, user_id: str) -> bool:
    """Delete user."""
    result = await db.users.delete_one({"_id": ObjectId(user_id)})
    return result.deleted_count > 0
```

### Use Projections

```python
# ✅ Only fetch needed fields
user = await db.users.find_one(
    {"_id": ObjectId(user_id)},
    {"name": 1, "email": 1, "created_at": 1}  # Only these fields
)

# ✅ Exclude sensitive fields
user = await db.users.find_one(
    {"_id": ObjectId(user_id)},
    {"hashed_password": 0, "reset_token": 0}  # Exclude these
)
```

## Error Handling

### Exceptions

```python
# In routers: Use HTTPException
from fastapi import HTTPException

raise HTTPException(status_code=404, detail="User not found")

# In services: Use standard exceptions
raise ValueError("Invalid email format")
raise KeyError("User ID not found")
raise RuntimeError("Database connection failed")
```

### Try-Except

```python
# ✅ Specific exceptions
try:
    result = await db.users.insert_one(user_data)
except DuplicateKeyError:
    raise HTTPException(status_code=409, detail="User already exists")

# ✅ Log and re-raise
try:
    await external_api_call()
except httpx.RequestError as e:
    logger.error(f"API call failed: {e}")
    raise HTTPException(status_code=503, detail="External service unavailable")
```

## Functions

### Single Responsibility

```python
# ✅ Each function does one thing
async def get_user_by_id(user_id: str) -> UserPublic | None:
    """Get user by ID."""
    return await user_client.find_by_id(user_id)

async def get_user_by_email(email: str) -> UserPublic | None:
    """Get user by email."""
    return await user_client.find_by_email(email)

# ❌ Function does too much
async def get_user(id_or_email: str):
    if "@" in id_or_email:
        return await user_client.find_by_email(id_or_email)
    else:
        return await user_client.find_by_id(id_or_email)
```

### Pure Functions

```python
# ✅ Pure function - no side effects
def calculate_discount(price: float, percent: float) -> float:
    """Calculate discounted price."""
    return price * (1 - percent / 100)

# ✅ Pure function with clear inputs/outputs
def format_user_name(first: str, last: str) -> str:
    """Format user's full name."""
    return f"{first} {last}".strip()
```

## Logging

```python
import logging

logger = logging.getLogger(__name__)

# ✅ Structured logging
logger.info("User created", extra={"user_id": user_id})
logger.warning("Rate limit exceeded", extra={"ip": client_ip})
logger.error("Database error", extra={"error": str(e)}, exc_info=True)

# ❌ NEVER log secrets
logger.info(f"MongoDB URI: {settings.MONGODB_URI}")  # NO!
logger.info(f"Password: {password}")  # NO!
```

## Testing Patterns

### Async Tests

```python
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_create_user(client: AsyncClient):
    """Test user creation."""
    response = await client.post(
        "/users",
        json={
            "name": "Test User",
            "email": "test@example.com",
            "password": "SecurePass123"
        }
    )
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Test User"
    assert data["email"] == "test@example.com"
    assert "password" not in data  # Should not return password
```

### Fixtures

```python
@pytest.fixture
async def test_user(db):
    """Create a test user."""
    user = {
        "name": "Test User",
        "email": "test@example.com",
        "hashed_password": "hashed",
        "is_active": True,
        "created_at": datetime.utcnow()
    }
    result = await db.users.insert_one(user)
    user["_id"] = str(result.inserted_id)
    
    yield user
    
    # Cleanup
    await db.users.delete_one({"_id": ObjectId(user["_id"])})
```

## Performance

### Database Queries

```python
# ✅ Use indexes
await db.users.create_index("email", unique=True)
await db.projects.create_index([("user_id", 1), ("created_at", -1)])

# ✅ Use projections
user = await db.users.find_one({"_id": ObjectId(user_id)}, {"name": 1, "email": 1})

# ✅ Limit results
users = await db.users.find().limit(100).to_list(length=100)

# ✅ Use aggregation for complex queries
pipeline = [
    {"$match": {"is_active": True}},
    {"$group": {"_id": "$role", "count": {"$sum": 1}}}
]
results = await db.users.aggregate(pipeline).to_list(length=None)
```

### Caching

```python
from functools import lru_cache

# Cache for pure functions
@lru_cache(maxsize=128)
def calculate_something_expensive(value: int) -> int:
    # Expensive calculation
    return result
```

## Common Mistakes

❌ No type hints on functions  
❌ Using sync libraries in async code  
❌ Not validating ObjectId before queries  
❌ Logging secrets or connection strings  
❌ Not using projections (fetching all fields)  
❌ Missing docstrings on public functions  
❌ Nested if statements (use guard clauses)  
❌ Functions doing multiple things  
❌ Not handling None returns from database  

## Code Quality Checklist

Before committing:
```bash
✅ uv run ruff format backend     # Format code
✅ uv run ruff check backend --fix # Fix linting issues
✅ uv run mypy backend             # Type check
✅ uv run pytest --cov=backend     # Run tests with coverage
```

## Remember

1. **Type hints**: Required on ALL functions
2. **Async**: Use `async def` for I/O operations
3. **Naming**: Clear, descriptive, follows conventions
4. **Pydantic**: Use for ALL data validation
5. **MongoDB**: Always validate ObjectId
6. **Guard clauses**: Validate early, return early
7. **Logging**: Never log secrets
8. **Testing**: Write tests for all functions

