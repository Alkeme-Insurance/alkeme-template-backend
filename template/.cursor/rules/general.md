# General Guidelines - Security, Quality, Workflow

## Security (NEVER BREAK THESE RULES)

### Never Log Secrets

```python
# ❌ NEVER DO THIS
logger.info(f"Connecting to {settings.MONGODB_URI}")  # Contains password!
logger.info(f"API key: {settings.API_KEY}")
logger.info(f"Token: {user_token}")

# ✅ Correct - log actions, not secrets
logger.info("Connecting to database")
logger.info("Authenticating with external API")
logger.info("Token validated successfully")
```

### Environment Variables

```python
# ✅ Use environment variables for all secrets
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    MONGODB_URI: str  # Never hardcode
    API_KEY: str
    JWT_SECRET: str
    
    model_config = SettingsConfigDict(env_file=".env")

# ❌ Never commit secrets
API_KEY = "sk-1234567890"  # NO!
```

### Input Validation

```python
# ✅ Always validate with Pydantic
from pydantic import BaseModel, EmailStr, Field

class UserCreate(BaseModel):
    email: EmailStr  # Validates email format
    name: str = Field(..., min_length=1, max_length=100)
    age: int = Field(..., ge=0, le=150)

# ❌ Don't accept raw dicts
@router.post("/users")
async def create_user(data: dict):  # No validation!
    pass
```

### CORS Configuration

```python
# ✅ Specific origins in production
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://yourdomain.com",
        "https://app.yourdomain.com"
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"],
)

# ❌ Never use wildcard in production
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Security risk!
)
```

### Password Handling

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# ✅ Always hash passwords
def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

# ❌ Never store plain text passwords
user_data["password"] = password  # NO!
```

## Code Quality

### Pre-Commit Checklist

Before every commit:
```bash
✅ uv run ruff format backend        # Format code
✅ uv run ruff check backend --fix   # Fix linting
✅ uv run mypy backend               # Type check
✅ uv run pytest --cov=backend       # Run tests
```

### Code Review Checklist

- [ ] All functions have type hints
- [ ] All public functions have docstrings
- [ ] No secrets in code or logs
- [ ] Input validation with Pydantic
- [ ] Guard clauses used for error handling
- [ ] Tests written for new functionality
- [ ] No commented-out code
- [ ] No debug print statements

### Documentation

```python
# ✅ Document public functions and classes
async def get_user_by_id(user_id: str) -> UserPublic | None:
    """
    Get user by ID.
    
    Args:
        user_id: MongoDB ObjectId as string
        
    Returns:
        UserPublic model if found, None otherwise
        
    Raises:
        HTTPException: If user_id is invalid format
    """
    if not ObjectId.is_valid(user_id):
        raise HTTPException(status_code=400, detail="Invalid ID")
    return await user_client.find_by_id(user_id)
```

## Git Workflow

### Commit Messages

```bash
# ✅ Good commit messages (conventional commits)
feat: add user profile endpoint
fix: correct email validation logic
docs: update API documentation
refactor: simplify user service
test: add tests for authentication
chore: update dependencies

# ❌ Bad commit messages
git commit -m "changes"
git commit -m "fix"
git commit -m "wip"
```

### Branch Naming

```bash
feature/user-authentication
fix/email-validation-bug
refactor/database-client
docs/api-documentation
```

### .gitignore

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
.Python
venv/
.venv/

# Environment
.env
.env.local
.env.*

# IDE
.vscode/
.idea/
*.swp
*.swo

# Testing
.coverage
htmlcov/
.pytest_cache/

# Build
dist/
build/
*.egg-info/
```

## Development Workflow

### Local Development

```bash
# 1. Install dependencies
uv sync

# 2. Set up environment
cp env.example .env
# Edit .env with your values

# 3. Run database
docker compose up -d mongodb

# 4. Run development server
uv run uvicorn backend.main:app --reload

# 5. Access API docs
open http://localhost:8000/docs
```

### Docker Development

```bash
# Run entire stack
docker compose up --build

# Run in background
docker compose up -d

# View logs
docker compose logs -f backend

# Stop services
docker compose down

# Clean up volumes
docker compose down -v
```

### Testing

```bash
# Run all tests
uv run pytest

# Run with coverage
uv run pytest --cov=backend --cov-report=html

# Run specific test file
uv run pytest tests/test_users.py

# Run specific test
uv run pytest tests/test_users.py::test_create_user

# Run with verbose output
uv run pytest -v

# Run failed tests only
uv run pytest --lf
```

## Monitoring & Logging

### Structured Logging

```python
import logging

logger = logging.getLogger(__name__)

# ✅ Include context in logs
logger.info(
    "User created successfully",
    extra={
        "user_id": user_id,
        "email": user.email,
        "source": "api"
    }
)

logger.error(
    "Database operation failed",
    extra={
        "operation": "insert",
        "collection": "users",
        "error": str(e)
    },
    exc_info=True
)
```

### Health Checks

```python
@app.get("/health")
async def health_check():
    """Health check endpoint for monitoring."""
    return {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "version": "1.0.0"
    }

@app.get("/health/db")
async def database_health():
    """Database health check."""
    try:
        await db.command("ping")
        return {"status": "healthy", "database": "connected"}
    except Exception as e:
        logger.error(f"Database health check failed: {e}")
        raise HTTPException(status_code=503, detail="Database unavailable")
```

## Performance

### Database Optimization

```python
# ✅ Use indexes
await db.users.create_index("email", unique=True)
await db.projects.create_index([("user_id", 1), ("created_at", -1)])

# ✅ Use projections
user = await db.users.find_one(
    {"_id": ObjectId(user_id)},
    {"name": 1, "email": 1}
)

# ✅ Limit results
users = await db.users.find().limit(100).to_list(length=100)

# ✅ Use aggregation for complex queries
pipeline = [
    {"$match": {"is_active": True}},
    {"$group": {"_id": "$role", "count": {"$sum": 1}}}
]
```

### API Performance

```python
# ✅ Pagination
@router.get("/users")
async def list_users(skip: int = 0, limit: int = 10):
    users = await service.list_users(skip=skip, limit=limit)
    return users

# ✅ Response models (automatic filtering)
@router.get("/users/{user_id}", response_model=UserPublic)
async def get_user(user_id: str):
    return await service.get_user_by_id(user_id)

# ✅ Background tasks for long operations
from fastapi import BackgroundTasks

@router.post("/users")
async def create_user(user: UserCreate, background_tasks: BackgroundTasks):
    new_user = await service.create_user(user)
    background_tasks.add_task(send_welcome_email, new_user.email)
    return new_user
```

## Dependencies

### Updating Dependencies

```bash
# Update all dependencies
uv lock --upgrade

# Update specific package
uv add package@latest

# Check for security vulnerabilities
uv audit
```

### Dependency Management

```toml
# pyproject.toml
[project]
dependencies = [
    "fastapi>=0.104.0",
    "motor>=3.3.0",
    "pydantic>=2.5.0",
    "pydantic-settings>=2.1.0",
]

[dependency-groups]
dev = [
    "pytest>=7.4.0",
    "pytest-asyncio>=0.21.0",
    "httpx>=0.25.0",
    "ruff>=0.1.0",
    "mypy>=1.7.0",
]
```

## Common Mistakes to Avoid

### Security

❌ Logging secrets or connection strings  
❌ Hardcoding API keys or passwords  
❌ Using `*` in CORS origins (production)  
❌ Not validating user input  
❌ Storing plain text passwords  
❌ Committing `.env` files  

### Code Quality

❌ Missing type hints  
❌ No tests for new features  
❌ Commented-out code in commits  
❌ Debug print statements  
❌ Long functions (>50 lines)  
❌ Deeply nested code (>3 levels)  

### Architecture

❌ Business logic in routers  
❌ FastAPI imports in services  
❌ Database operations in routers  
❌ Not using dependency injection  
❌ Global state instead of dependencies  

### Database

❌ Not validating ObjectId  
❌ Fetching all fields when only some needed  
❌ No indexes on queried fields  
❌ Not handling None from queries  
❌ Using sync operations in async code  

## Best Practices Summary

### Security
✅ Never log secrets  
✅ Use environment variables  
✅ Validate all inputs  
✅ Hash passwords  
✅ Specific CORS origins  

### Code Quality
✅ Type hints on all functions  
✅ Guard clauses for validation  
✅ Write tests  
✅ Document public APIs  
✅ Run pre-commit checks  

### Architecture
✅ Routers → Services → Clients  
✅ Single responsibility  
✅ Dependency injection  
✅ Error handling at correct layer  

### Performance
✅ Use database indexes  
✅ Use projections  
✅ Implement pagination  
✅ Async operations  
✅ Background tasks for long operations  

### Development
✅ Use Docker for consistency  
✅ Write meaningful commit messages  
✅ Keep dependencies updated  
✅ Monitor application health  
✅ Structure logs properly  

## Resources

- **FastAPI**: https://fastapi.tiangolo.com/
- **Pydantic**: https://docs.pydantic.dev/
- **Motor**: https://motor.readthedocs.io/
- **MongoDB**: https://www.mongodb.com/docs/
- **Python Type Hints**: https://docs.python.org/3/library/typing.html
- **PEP 8**: https://pep8.org/

## Remember

Write **clean**, **secure**, **tested** code that follows **best practices** and is **easy to maintain**.

