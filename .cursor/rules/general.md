# General Guidelines - Alkeme Backend Template

## Security (Critical - Never Break These Rules)

### Secrets Management

**NEVER log secrets, credentials, or sensitive data:**

```python
# ❌ NEVER DO THIS
logger.info(f"Connecting to {settings.MONGODB_URI}")  # Contains password!
logger.debug(f"Auth token: {token}")  # Exposes token!
logger.info(f"API key: {api_key}")  # Leaks credentials!

# ✅ Correct - log actions, not secrets
logger.info("Connecting to database")
logger.debug("Authentication token received")
logger.info("API key validated")
```

**What counts as a secret:**
- Passwords and password hashes
- API keys and tokens
- JWT tokens and claims
- Connection strings (contain passwords)
- OAuth client secrets
- Encryption keys
- Private keys
- Any credentials or authentication data

### Environment Variables
- Store ALL secrets in `.env` file
- **NEVER commit `.env` files** to git
- Provide `.env.example` with placeholder values
- Use Azure Key Vault for production secrets
- Validate required environment variables on startup

```python
# ✅ Good .env.example
MONGODB_URI=mongodb://localhost:27017/mydb
AZURE_CLIENT_SECRET=your-secret-here-replace-in-production

# ❌ Never in .env.example or git
MONGODB_URI=mongodb://user:RealPassword123@prod.mongo.azure.com
AZURE_CLIENT_SECRET=actual-real-secret-ab123cd456
```

### Input Validation
- Validate **ALL** user inputs with Pydantic models
- Use `Query()`, `Path()`, `Field()` for parameter validation
- Sanitize inputs to prevent injection attacks
- Set maximum request body size
- Validate file uploads (type, size, content)

```python
from pydantic import Field, validator

class UserCreate(BaseModel):
    email: EmailStr  # Validates email format
    name: str = Field(..., min_length=1, max_length=100)
    age: int = Field(..., ge=0, le=150)  # Range validation
```

### CORS Configuration

```python
# ❌ NEVER in production
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # WRONG - allows any origin!
)

# ✅ Correct - specify exact origins
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.get_cors_origins(),  # From config
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Authentication
- Use Azure AD (fastapi-azure-auth) for production
- Validate JWT tokens on every protected endpoint
- Use `Security(azure_scheme)` dependency
- Support `DEV_NO_AUTH` flag for local development only
- Never bypass auth in production

### Rate Limiting
- Apply rate limiting on public endpoints
- Use Azure API Management or middleware
- Protect against brute force attacks
- Track by IP address or authenticated user

## Code Quality Standards

### Pre-Commit Checklist
Before every commit, run:

```bash
# 1. Format code
uv run ruff format backend

# 2. Lint code
uv run ruff check backend --fix

# 3. Type check
uv run mypy backend

# 4. Run tests
uv run pytest

# 5. Check coverage
uv run pytest --cov=backend --cov-report=term-missing
```

All must pass before committing.

### Coverage Requirements
- Maintain **>80% test coverage**
- Test both happy paths and error cases
- Mock external dependencies
- Use fixtures for common setup

### Code Review Standards
- No business logic in routers (belongs in services)
- All functions have type hints
- No secrets in logs
- Error messages are actionable
- Code follows naming conventions
- Docstrings on public functions

## Git & Version Control

### Commit Messages
Use conventional commits format:

```bash
# Format: <type>: <description>

feat: add user registration endpoint
fix: resolve ObjectId validation bug
docs: update API documentation
refactor: extract email validation to service
test: add tests for project deletion
chore: update dependencies
```

**Types:**
- `feat:` - New feature
- `fix:` - Bug fix
- `docs:` - Documentation changes
- `refactor:` - Code refactoring (no functional changes)
- `test:` - Adding or updating tests
- `chore:` - Maintenance tasks (deps, config, etc.)
- `perf:` - Performance improvements
- `style:` - Code style changes (formatting, etc.)

### Commit Best Practices
- Keep commits atomic and focused
- Commit related changes together
- Write clear, descriptive messages
- Reference issues when applicable: `fix: resolve login issue (#123)`

### What NOT to Commit
❌ `.env` files (secrets)  
❌ `__pycache__/` directories  
❌ `.venv/` or `venv/` directories  
❌ IDE-specific files (`.idea/`, `.vscode/`)  
❌ Build artifacts (`dist/`, `build/`)  
❌ Database files (`.db`, `.sqlite`)  
❌ Log files (`*.log`)  
❌ Temporary files (`*.tmp`, `*.temp`)  

### .gitignore
Use comprehensive `.gitignore` for Python projects:

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
.venv/
venv/
ENV/

# Environment
.env
.env.local
.env.*.local

# IDE
.vscode/
.idea/
*.swp
*.swo

# Testing
.pytest_cache/
.coverage
htmlcov/

# Build
dist/
build/
*.egg-info/
```

## Development Workflow

### Local Development Setup
```bash
# 1. Clone repository
git clone <repo-url>
cd project

# 2. Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# 3. Create virtual environment and install dependencies
uv sync

# 4. Copy environment template
cp .env.example .env
# Edit .env with your values

# 5. Start MongoDB (Docker)
docker compose up -d mongodb

# 6. Run application
uv run uvicorn backend.main:app --reload

# 7. Run tests
uv run pytest
```

### Docker Development
```bash
# Build and run with Docker Compose
docker compose up --build

# View logs
docker compose logs -f backend

# Run tests in Docker
docker compose exec backend pytest

# Stop services
docker compose down
```

### Azure Deployment
```bash
# 1. Build Docker image
docker build -t backend:latest .

# 2. Login to Azure Container Registry
az acr login --name <acr-name>

# 3. Tag and push image
docker tag backend:latest <acr-name>.azurecr.io/backend:latest
docker push <acr-name>.azurecr.io/backend:latest

# 4. Deploy infrastructure
cd infra
./deploy.sh
```

## Testing Standards

### Test Organization
```
tests/
├── conftest.py              # Pytest fixtures
├── test_main.py             # Test main app
├── test_routers/
│   ├── test_users.py
│   └── test_projects.py
├── test_services/
│   ├── test_user_service.py
│   └── test_project_service.py
└── test_models/
    └── test_user_models.py
```

### Test Naming
```python
# ✅ Good - descriptive, specific
def test_create_user_with_valid_data_returns_201():
    pass

def test_create_user_with_duplicate_email_returns_422():
    pass

def test_get_user_by_invalid_id_returns_404():
    pass

# ❌ Bad - vague
def test_user():
    pass

def test_api():
    pass
```

### Test Structure (AAA Pattern)
```python
@pytest.mark.asyncio
async def test_create_user(test_db):
    # Arrange - Set up test data
    service = UserService(test_db)
    user_data = UserCreate(
        email="test@example.com",
        name="Test User",
        password="secure123",
    )
    
    # Act - Perform the action
    user = await service.create_user(user_data)
    
    # Assert - Verify the result
    assert user.email == "test@example.com"
    assert user.is_active is True
    assert user.id is not None
```

### Mock External Dependencies
```python
import pytest
from unittest.mock import AsyncMock, patch

@pytest.fixture
def mock_mailgun():
    with patch("backend.clients.mailgun.send_email") as mock:
        mock.return_value = AsyncMock(return_value=True)
        yield mock

async def test_user_registration_sends_email(mock_mailgun):
    # Test that email is sent without actually sending it
    await register_user(user_data)
    mock_mailgun.assert_called_once()
```

## Dependencies Management

### Using uv Package Manager
```bash
# Install dependencies
uv sync

# Add new dependency
uv add fastapi

# Add dev dependency
uv add --dev pytest

# Update dependencies
uv lock --upgrade

# Run command in venv
uv run pytest

# Install global tool
uv tool install ruff
```

### Dependency Constraints
- Pin major versions: `fastapi>=0.117.1` (allows minor updates)
- **Critical**: Use PyMongo 3.x (NOT 4.x) for Cosmos DB compatibility
- Document why specific versions are pinned
- Review and update dependencies regularly (monthly)

```toml
# pyproject.toml
dependencies = [
    "fastapi[standard]>=0.117.1",
    "pydantic>=2.11.9",
    "motor>=2.5.0,<3.0.0",  # Motor 2.x for async
    "pymongo>=3.13.0,<4.0.0",  # PyMongo 3.x for Cosmos DB!
]
```

## Documentation Standards

### Code Documentation
- Add docstrings to all public functions, classes, and modules
- Use Google or NumPy docstring format
- Include examples for complex functions
- Document parameters, return values, and exceptions

```python
def get_user_by_id(user_id: str) -> UserPublic | None:
    """Get user by ID from database.
    
    Args:
        user_id: MongoDB ObjectId as string
        
    Returns:
        UserPublic model if found, None otherwise
        
    Raises:
        ValueError: If user_id is not a valid ObjectId format
        
    Example:
        >>> user = await get_user_by_id("507f1f77bcf86cd799439011")
        >>> print(user.email)
        'user@example.com'
    """
    pass
```

### README Requirements
Every generated project must have:
- Project description
- Prerequisites
- Installation instructions
- Configuration guide
- Development workflow
- Testing instructions
- Deployment guide
- API documentation link

### API Documentation
- FastAPI generates OpenAPI docs automatically at `/docs`
- Add clear descriptions to endpoints via docstrings
- Use `response_model` for response schema documentation
- Tag endpoints for organization
- Include examples in docstrings

## Performance Guidelines

### Database Performance
- Create indexes for frequently queried fields
- Use projections to limit returned fields
- Implement pagination for large result sets
- Use aggregation pipelines for complex queries
- Monitor query performance with explain()

### API Performance
- Use `StreamingResponse` for large payloads
- Enable gzip compression middleware
- Set appropriate cache headers
- Monitor response times (aim for <500ms p95)
- Use async operations for parallel tasks

### Profiling
- Profile before optimizing (measure, don't guess)
- Use `cProfile` or `py-spy` for Python profiling
- Monitor with Azure Application Insights
- Set up alerts for performance degradation

## Logging Standards

### Structured Logging
```python
import logging

logger = logging.getLogger(__name__)

# ✅ Good - structured, no secrets
logger.info("User created", extra={
    "user_id": user.id,
    "email_domain": user.email.split("@")[1],  # Domain only, not full email
    "action": "user_registration"
})

# ❌ Bad - unstructured, contains secrets
logger.info(f"User {user.email} created with password {password}")
```

### Log Levels
- **DEBUG**: Detailed information for debugging
- **INFO**: General informational messages
- **WARNING**: Warning messages (deprecated features, etc.)
- **ERROR**: Error messages that need attention
- **CRITICAL**: Critical errors that require immediate action

### What to Log
✅ User actions (login, resource creation)  
✅ API requests/responses (status codes, duration)  
✅ Database operations (query types, duration)  
✅ External API calls  
✅ Errors and exceptions with traceback  

❌ Secrets, passwords, tokens  
❌ Personal identifiable information (PII)  
❌ Full connection strings  

## Monitoring & Observability

### Health Checks
```python
@app.get("/health")
async def health():
    """Health check endpoint for container orchestration."""
    return {"status": "healthy"}
```

### Metrics to Track
- API response times (p50, p95, p99)
- Error rates by endpoint
- Database query duration
- External API latency
- Active connections
- Memory and CPU usage

### Azure Application Insights
- Automatic request tracking
- Exception logging
- Custom metrics
- Distributed tracing
- Real user monitoring

## Common Mistakes to Avoid

### Security
❌ Logging secrets or credentials  
❌ Using `*` in CORS origins (production)  
❌ Not validating user inputs  
❌ Committing `.env` files  
❌ Bypassing authentication in production  

### Code Quality
❌ Missing type hints  
❌ No tests or low coverage  
❌ Business logic in routers  
❌ Using global state instead of DI  
❌ Not handling error cases  

### Database
❌ Not validating ObjectId strings  
❌ Using PyMongo 4.x (breaks Cosmos DB)  
❌ Missing indexes on queried fields  
❌ No pagination for large result sets  
❌ Returning full documents without projections  

### Development
❌ Not using virtual environments  
❌ Committing IDE-specific files  
❌ Skipping code quality checks  
❌ Vague commit messages  
❌ Not documenting complex logic  

## Tools & Resources

### Development Tools
- **uv**: Fast Python package manager
- **ruff**: Linter and formatter (replaces black, isort, flake8)
- **mypy**: Static type checker
- **pytest**: Testing framework
- **Docker**: Containerization
- **Azure CLI**: Cloud deployment

### Pre-commit Hooks
Use detect-secrets or pre-commit to scan for secrets:

```bash
# Install detect-secrets
uv tool install detect-secrets

# Scan for secrets
detect-secrets scan --baseline .secrets.baseline

# Pre-commit hook
detect-secrets-hook --baseline .secrets.baseline
```

### Useful Commands
```bash
# Code quality
uv run ruff format backend
uv run ruff check backend --fix
uv run mypy backend

# Testing
uv run pytest
uv run pytest --cov=backend
uv run pytest -k test_user  # Run specific tests
uv run pytest -x  # Stop on first failure

# Docker
docker compose up -d
docker compose logs -f backend
docker compose down

# Azure
az login
az account set --subscription <id>
az acr login --name <acr-name>
```

## References

- **Security Best Practices**: OWASP Top 10
- **Python Style**: PEP 8
- **Git Commits**: Conventional Commits
- **Testing**: pytest documentation
- **Docker**: Docker best practices
- **Azure**: Azure Container Apps documentation

