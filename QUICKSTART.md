# Alkeme Backend Template - Quick Start Guide

## 🚀 Generate Your First Project

### Option 1: Interactive Mode (Recommended)

```bash
uvx copier copy ~/workspace/alkeme-template-backend my-backend
```

You'll be prompted for:
- Project name
- Package name (Python identifier)
- Description
- Author information
- Azure AD authentication (yes/no)
- Azure Cosmos DB deployment (yes/no)
- Python version (3.10-3.13)
- Git hooks for secret scanning

### Option 2: Non-Interactive Mode

```bash
uvx copier copy ~/workspace/alkeme-template-backend my-backend \
  --data project_name="My Awesome Backend" \
  --data package_name="my_backend" \
  --data use_azure_auth=true \
  --data deploy_cosmos_db=true \
  --trust --defaults
```

## 📦 What You Get

```
my-backend/
├── backend/               # FastAPI application
│   ├── main.py           # App entry point with lifespan
│   ├── config.py         # Settings management
│   ├── auth.py           # Azure AD (if enabled)
│   ├── routers/          # API endpoints (users, projects)
│   ├── services/         # Business logic
│   ├── models/           # Pydantic schemas
│   ├── clients/          # MongoDB client
│   └── utils/            # Indexes, seeding
├── tests/                # Pytest suite
├── pyproject.toml        # Dependencies (uv)
├── Dockerfile            # Multi-stage build
├── docker-compose.yml    # Local dev stack
└── README.md             # Project-specific docs
```

## 🏃 Run Your Project

### Development Mode (Python)

```bash
cd my-backend

# Install dependencies
uv sync

# Copy environment template
cp env.example .env

# Start server
uv run uvicorn backend.main:app --reload

# Visit
# http://localhost:8000/docs (Swagger UI)
# http://localhost:8000/redoc (ReDoc)
```

### Docker Mode (Recommended)

```bash
cd my-backend

# Start everything (backend + MongoDB)
docker compose up --build

# In another terminal, check logs
docker compose logs -f backend

# Stop
docker compose down
```

## ✅ Verify Installation

```bash
# Health check
curl http://localhost:8000/health
# Expected: {"status":"healthy"}

# API docs
open http://localhost:8000/docs
```

## 🧪 Run Tests

```bash
cd my-backend

# All tests
uv run pytest

# With coverage
uv run pytest --cov=backend

# Verbose
uv run pytest -v
```

## 🔧 Code Quality

```bash
# Format code
uv run ruff format backend

# Lint code
uv run ruff check backend --fix

# Type check
uv run mypy backend

# All checks
uv run ruff format backend && \
uv run ruff check backend --fix && \
uv run mypy backend
```

## 📝 Environment Variables

### Required (Local Development)

```bash
# .env file
MONGODB_URI=mongodb://mongodb:27017/my_backend
BACKEND_CORS_ORIGINS=http://localhost:3000
DEV_NO_AUTH=true
```

### Required (Azure AD - if enabled)

```bash
AZURE_TENANT_ID=your-tenant-id
AZURE_CLIENT_ID=your-client-id
AZURE_CLIENT_SECRET=your-secret
OPENAPI_CLIENT_ID=your-client-id
```

### Production (Azure Cosmos DB)

```bash
MONGODB_URI=mongodb://your-account.mongo.cosmos.azure.com:10255/?ssl=true&replicaSet=globaldb
DEV_NO_AUTH=false
```

## 🏗️ Project Architecture

```
┌─────────────┐
│   Routers   │  ← FastAPI endpoints (HTTP layer)
└──────┬──────┘
       │
┌──────▼──────┐
│  Services   │  ← Business logic (pure Python)
└──────┬──────┘
       │
┌──────▼──────┐
│   Clients   │  ← MongoDB, external APIs
└─────────────┘
```

### Routers
- Handle HTTP requests/responses
- Validate input with Pydantic
- Use `Depends()` for dependencies
- Optional `Security()` for auth

### Services
- Pure business logic
- No FastAPI imports
- Return Pydantic models
- Validate business rules

### Clients
- Database operations (Motor)
- External API calls
- No business logic

## 🐳 Docker Commands

```bash
# Build only
docker compose build

# Start in background
docker compose up -d

# View logs
docker compose logs -f

# Restart a service
docker compose restart backend

# Stop and remove
docker compose down

# Stop and remove with volumes
docker compose down -v
```

## 🔐 Azure AD Setup (If Enabled)

1. **Register App in Azure AD**
   - Go to Azure Portal → Azure AD → App registrations
   - Click "New registration"
   - Name: "My Backend API"
   - Redirect URI: `http://localhost:8000/oauth2-redirect`

2. **Configure App**
   - Copy Application (client) ID → `AZURE_CLIENT_ID`
   - Copy Directory (tenant) ID → `AZURE_TENANT_ID`
   - Create client secret → `AZURE_CLIENT_SECRET`

3. **Expose API**
   - App registrations → Expose an API
   - Add scope: `user_impersonation`

4. **Update .env**
   ```bash
   AZURE_TENANT_ID=your-tenant-id
   AZURE_CLIENT_ID=your-client-id
   AZURE_CLIENT_SECRET=your-secret
   OPENAPI_CLIENT_ID=your-client-id
   ```

## 📚 Common Tasks

### Add a New Endpoint

1. Create model in `backend/models/`
2. Create service in `backend/services/`
3. Create router in `backend/routers/`
4. Register router in `backend/main.py`

### Add Database Index

Edit `backend/utils/indexes.py`:
```python
await db.my_collection.create_index("field_name")
```

### Add Environment Variable

1. Add to `backend/config.py`:
   ```python
   MY_VAR: str = Field(default="", alias="MY_VAR")
   ```

2. Add to `env.example`:
   ```bash
   MY_VAR=value
   ```

## 🐛 Troubleshooting

### Server won't start

```bash
# Check MongoDB is running
docker compose ps

# View backend logs
docker compose logs backend

# Restart services
docker compose restart
```

### Dependencies won't install

```bash
# Clear cache
rm -rf .venv
uv cache clean

# Reinstall
uv sync
```

### Type checking errors

```bash
# Check specific file
uv run mypy backend/main.py

# Ignore specific error
# type: ignore[error-code]
```

## 📖 Learn More

- **Template Documentation**: `README.md`
- **Contributing Guide**: `CONTRIBUTING.md`
- **Implementation Details**: `IMPLEMENTATION.md`
- **FastAPI Docs**: https://fastapi.tiangolo.com/
- **Pydantic Docs**: https://docs.pydantic.dev/
- **Motor Docs**: https://motor.readthedocs.io/

## 🆘 Get Help

- **Template Issues**: Open issue in template repository
- **Generated Project Issues**: Check generated `README.md`
- **Email**: dev@alkeme.com

---

**Happy coding! 🎉**

