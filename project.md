# Alkeme Template Backend — Deployment-Ready Copier Template

## Overview

This Copier template generates production-ready FastAPI backend applications with:

- **FastAPI** with async support and Pydantic v2
- **MongoDB/Azure Cosmos DB** integration via Motor (async)
- **Azure AD Authentication** via fastapi-azure-auth
- **Docker** multi-stage builds (optimized with uv)
- **Azure Container Apps** deployment via Bicep IaC
- **CI/CD** with GitHub Actions
- **Modern Python tooling**: uv, ruff, mypy
- **Structured architecture**: routers, services, models, clients pattern

## What This Template Generates

### Development vs Production Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    LOCAL DEVELOPMENT                             │
├─────────────────────────────────────────────────────────────────┤
│  Docker Compose Stack:                                           │
│  ┌──────────────────┐          ┌──────────────────┐            │
│  │  FastAPI Backend │  ◄────►  │  MongoDB 7.0     │            │
│  │  (Port 8000)     │          │  (Docker Image)  │            │
│  └──────────────────┘          └──────────────────┘            │
│         ▲                                                        │
│         │ HTTP                                                   │
│         ▼                                                        │
│  ┌──────────────────┐                                           │
│  │  React Frontend  │                                           │
│  │  (Port 3000)     │                                           │
│  └──────────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                 AZURE PRODUCTION DEPLOYMENT                      │
├─────────────────────────────────────────────────────────────────┤
│  Container Apps + Cosmos DB:                                     │
│  ┌──────────────────┐          ┌──────────────────┐            │
│  │  FastAPI Backend │  ◄────►  │  Azure Cosmos DB │            │
│  │  Container App   │          │  (MongoDB API)   │            │
│  │  (Auto-scaling)  │          │  Fully Managed   │            │
│  └──────────────────┘          └──────────────────┘            │
│         ▲                                                        │
│         │ HTTPS                                                  │
│         ▼                                                        │
│  ┌──────────────────┐                                           │
│  │  React Frontend  │                                           │
│  │  Container App   │                                           │
│  └──────────────────┘                                           │
│                                                                  │
│  Deployed via: Bicep IaC + GitHub Actions                       │
└─────────────────────────────────────────────────────────────────┘
```

**Key Point:** Local development uses a MongoDB Docker container for simplicity. Production automatically uses Azure Cosmos DB (MongoDB API) for a fully managed, globally distributed database.

### Core Features

✅ **Production-Ready FastAPI App**
- Async-first architecture with Motor (MongoDB driver)
- Pydantic v2 models and schemas
- Health check endpoints
- CORS middleware configuration
- Lifespan context managers for startup/shutdown

✅ **Authentication & Security**
- Azure AD (Entra ID) integration with JWT validation
- Optional dev mode bypass (DEV_NO_AUTH flag)
- Secure secrets management via environment variables
- CORS origins configuration
- Security headers and input validation

✅ **Data Persistence**
- MongoDB/Azure Cosmos DB client with connection pooling
- Async CRUD operations pattern
- Index management and migrations
- Seed data utilities
- Consistent error handling

✅ **Docker & Deployment**
- Multi-stage Dockerfile with uv (fast dependency resolution)
- docker-compose.yml for local development
- Azure Container Registry (ACR) integration
- Azure Container Apps deployment
- Health checks and logging

✅ **Developer Experience**
- `.cursorrules` for AI-assisted development
- Pre-commit hooks with secret scanning (detect-secrets)
- Structured logging with correlation IDs
- Hot reload for local development
- Comprehensive environment configuration

✅ **CI/CD Pipeline**
- GitHub Actions workflows
- Automated testing (lint, type check, unit tests)
- Docker image building and pushing to ACR
- Bicep deployment to Azure Container Apps
- Multi-environment support (dev/staging/prod)

---

## Project Structure (Generated)

```
{{package_name}}/
├── backend/                            # Main application code
│   ├── __init__.py
│   ├── main.py                         # FastAPI app & lifespan
│   ├── config.py                       # Settings (Pydantic BaseSettings)
│   ├── auth.py                         # Azure AD auth setup
│   ├── dependencies.py                 # FastAPI dependencies
│   ├── routers/                        # API endpoints
│   │   ├── users.py
│   │   ├── projects.py
│   │   └── kanban.py (optional)
│   ├── services/                       # Business logic layer
│   │   ├── user_service.py
│   │   └── project_service.py
│   ├── models/                         # Pydantic models/schemas
│   │   ├── user.py
│   │   └── project.py
│   ├── clients/                        # External integrations
│   │   ├── mongo_db.py                 # MongoDB client
│   │   └── mailgun.py (optional)       # Email service
│   ├── utils/                          # Utilities
│   │   ├── seed.py                     # Data seeding
│   │   ├── indexes.py                  # Index management
│   │   └── logging_config.py           # Structured logging
│   └── internals/                      # Internal admin routes
│       └── admin.py
├── tests/                              # Test suite
│   ├── conftest.py                     # Pytest fixtures
│   ├── test_main.py
│   └── test_routers/
├── infra/                              # Azure infrastructure
│   ├── main.bicep                      # Main orchestrator
│   ├── main.bicepparam.jinja           # Parameters (per environment)
│   ├── deploy.sh                       # Deployment script
│   └── modules/
│       ├── container-registry.bicep    # ACR
│       ├── container-app.bicep         # Container Apps
│       ├── cosmos-db.bicep (optional)  # Cosmos DB
│       ├── log-analytics.bicep         # Monitoring
│       └── key-vault.bicep (optional)  # Secrets
├── Dockerfile                          # Multi-stage build with uv
├── docker-compose.yml                  # Local dev stack
├── pyproject.toml                      # Python dependencies (uv)
├── .env.example                        # Environment template
├── .cursorrules                        # Cursor AI rules
├── .pre-commit-config.yaml             # Pre-commit hooks
├── .github/
│   └── workflows/
│       └── ci-cd.yml                   # GitHub Actions
└── README.md                           # Project documentation
```

---

## Quick Start

### Prerequisites

- **Python 3.10-3.13**
- **uv** (Python package manager): `curl -LsSf https://astral.sh/uv/install.sh | sh`
- **Docker** and **Docker Compose**
- **Azure CLI** (`az`): For Azure deployments
- **GitHub CLI** (`gh`): For GitHub integrations (optional)

### Generate a New Project

```bash
# Install Copier (if not already installed)
uvx copier --version || uv tool install copier

# Generate project from template
uvx copier copy gh:alkeme/alkeme-template-backend my-backend

# Answer prompts interactively:
#   - Project name: "My Backend API"
#   - Package name: "my-backend"
#   - Azure features: Yes/No
#   - Feature flags: Kanban, Dashboard, etc.
```

### Local Development

```bash
cd my-backend

# Create virtual environment and install dependencies
uv sync

# Copy environment template
cp .env.example .env
# Edit .env with your values (MongoDB URI, Azure AD config, etc.)

# Run locally (hot reload)
uv run uvicorn backend.main:app --reload --host 0.0.0.0 --port 8000

# Or use Docker Compose
docker compose up --build
```

### Run Tests

```bash
# Run all tests
uv run pytest

# With coverage
uv run pytest --cov=backend --cov-report=term-missing

# Type checking
uv run mypy backend

# Linting
uv run ruff check backend
```

---

## Docker Development

### Build Image

```bash
docker build -t my-backend:latest .
```

### Run with Docker Compose

```bash
# Start all services (app + MongoDB)
docker compose up -d

# View logs
docker compose logs -f backend

# Stop services
docker compose down
```

**Compose Stack (Local Development):**
- `backend`: FastAPI application on port 8000
- `mongodb`: MongoDB 7.0 Docker image on port 27017

**Note:** In local development, the Docker Compose stack includes a MongoDB container for easy setup. At deployment time to Azure, this is automatically swapped for Azure Cosmos DB (MongoDB API) for a fully managed, production-ready database solution.

---

## Azure Deployment

### One-Time Setup

```bash
# Login to Azure
az login

# Set subscription
az account set --subscription "Your Subscription Name"

# Create service principal for CI/CD
az ad sp create-for-rbac --name "sp-my-backend-cicd" \
  --role Contributor \
  --scopes /subscriptions/{subscription-id}/resourceGroups/{resource-group} \
  --sdk-auth

# Save output for GitHub Secrets (see infra/README.md)
```

### Manual Deployment

```bash
cd infra

# Review/edit parameters
nano main.bicepparam

# Deploy to Azure (Cosmos DB will be created automatically)
./deploy.sh

# Output will show:
# - Container App URL
# - Cosmos DB connection string (automatically configured)
# - Container Registry details
```

**What Happens During Deployment:**
1. ✅ Bicep creates Azure Cosmos DB (MongoDB API)
2. ✅ Connection string is automatically injected into Container App
3. ✅ Your backend connects to Cosmos DB instead of local MongoDB
4. ✅ No code changes needed - same MongoDB driver works with both

### GitHub Actions (Automated)

1. Configure GitHub Secrets (see `.github/AZURE_SETUP.md`):
   - `AZURE_CLIENT_ID`
   - `AZURE_CLIENT_SECRET`
   - `AZURE_TENANT_ID`
   - `AZURE_SUBSCRIPTION_ID`
   - `ACR_LOGIN_SERVER`

2. Push to `main` branch → triggers deployment

---

## Environment Variables

### Core Configuration

```bash
# MongoDB Connection String
# Local Development (Docker Compose):
MONGODB_URI=mongodb://mongodb:27017/mybackend
# OR for local MongoDB on host:
MONGODB_URI=mongodb://localhost:27017/mybackend

# Azure Cosmos DB (Production - MongoDB API):
# MONGODB_URI=mongodb://username:password@accountname.mongo.cosmos.azure.com:10255/mybackend?ssl=true&replicaSet=globaldb&retrywrites=false&maxIdleTimeMS=120000&appName=@accountname@

# Azure AD Authentication
AZURE_TENANT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_CLIENT_SECRET=your-secret-here  # (backend only, never expose to frontend)
OPENAPI_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  # For Swagger UI

# CORS Origins (comma-separated)
BACKEND_CORS_ORIGINS=http://localhost:3000,http://localhost:5173

# Development Mode (bypass auth)
DEV_NO_AUTH=true  # Set to false in production
```

### Optional Services

```bash
# Mailgun (if use_mailgun=true)
MAILGUN_API_KEY=your-key
MAILGUN_DOMAIN=mg.yourdomain.com
MAILGUN_FROM_EMAIL=noreply@yourdomain.com

# Application Insights (Azure monitoring)
APPLICATIONINSIGHTS_CONNECTION_STRING=InstrumentationKey=...
```

See `.env.example` for complete reference.

---

## Copier Template Options

When generating a project, Copier will prompt for:

### Project Identity
- **project_name**: Display name (e.g., "My Backend API")
- **package_name**: Python package name (e.g., "my-backend")
- **project_description**: Short description
- **owner_org**: Organization name (e.g., "Alkeme")
- **author_name** / **author_email**: Maintainer info

### Feature Flags
- **use_azure_auth**: Include Azure AD authentication (default: true)
- **deploy_cosmos_db**: Deploy Azure Cosmos DB with Bicep infrastructure (default: true)
  - *Note: Local development always uses MongoDB Docker container regardless of this flag*
  - *This flag only affects whether Cosmos DB is deployed to Azure*
- **use_mailgun**: Include email service integration (default: false)
- **use_kanban**: Scaffold Kanban/board feature (default: false)
- **include_dashboard**: Add KPI dashboard endpoints (default: false)

### Development Tools
- **use_cursor_rules**: Generate `.cursorrules` for AI editors (default: true)
- **use_git_hooks**: Pre-commit hooks with secret scanning (default: true)
- **git_hook_tool**: `detect-secrets` or `pre-commit` (default: detect-secrets)
- **python_version**: 3.10, 3.11, 3.12, or 3.13 (default: 3.12)

### CI/CD
- **ci_github_actions**: Generate GitHub Actions workflow (default: true)
- **ci_include_azure_deploy**: Include Azure deployment job (default: true)

### Azure Configuration
- **use_azure_deployment**: Generate Bicep infrastructure (default: true)
- **azure_region**: Azure region (default: eastus)
- **azure_environment**: dev/staging/prod (default: dev)
- **use_azure_key_vault**: Secret management (default: false)
- **use_app_insights**: Application monitoring (default: true)

---

## Architecture Principles

This template follows the Alkeme engineering practices:

### Layered Architecture
```
Routers (HTTP layer)
   ↓
Services (Business logic)
   ↓
Clients (Data access & external APIs)
```

- **Routers**: Handle HTTP requests, validate input, return responses
- **Services**: Pure business logic, orchestrate data operations
- **Clients**: Database operations, external API calls, side effects

### Key Patterns
- **Dependency Injection**: Use `Depends()` for shared resources (DB, auth, config)
- **RORO (Receive Object, Return Object)**: Functions accept and return typed objects
- **Async-first**: All I/O operations are `async def`
- **Guard Clauses**: Early returns for validation and error cases
- **Single Responsibility**: Small functions with clear purpose

### Error Handling
- Use `HTTPException` for expected API errors
- Centralized exception middleware for unexpected errors
- Structured logging with correlation IDs
- Consistent error response format

### Security
- Validate all inputs with Pydantic models
- Never log secrets (credentials, tokens, API keys)
- Use Azure Key Vault for production secrets
- Rate limiting on sensitive endpoints (via API Gateway or middleware)
- Apply CORS restrictions

---

## Tools & Technologies

### Core Stack
- **FastAPI**: Modern async web framework
- **Pydantic v2**: Data validation and serialization
- **Motor**: Async MongoDB driver (wraps PyMongo)
- **uvicorn**: ASGI server

### Development Tools
- **uv**: Fast Python package manager (replaces pip/poetry/pipenv)
- **ruff**: Lightning-fast linter (replaces flake8, black, isort)
- **mypy**: Static type checker
- **pytest**: Testing framework
- **detect-secrets**: Pre-commit secret scanning

### Azure Services
- **Container Apps**: Serverless container hosting with auto-scaling
- **Container Registry (ACR)**: Private Docker image registry
- **Cosmos DB**: Globally distributed MongoDB-compatible database
- **Key Vault**: Secret and certificate management
- **Application Insights**: APM and monitoring
- **Log Analytics**: Centralized logging

### CI/CD
- **GitHub Actions**: Automated workflows
- **Bicep**: Infrastructure as Code for Azure
- **Azure CLI**: Deployment automation

---

## CI/CD Pipeline

### On Pull Request
1. ✅ Lint (ruff)
2. ✅ Type check (mypy)
3. ✅ Run tests (pytest)
4. ✅ Security scan (detect-secrets, Trivy)
5. ✅ Build Docker image (validation)

### On Push to `main`
1. All PR checks
2. 🐳 Build and tag Docker image
3. 📦 Push to Azure Container Registry
4. 🚀 Deploy to Azure Container Apps (dev environment)
5. 💬 Comment deployment URL on commit

### On Git Tag (`v*`)
1. All PR checks
2. 🐳 Build and tag with version
3. 📦 Push to ACR
4. 🚀 Deploy to production environment
5. 📝 Create GitHub Release

---

## Monitoring & Observability

### Built-in Monitoring
- **Health Check**: `/health` endpoint (no auth required)
- **Structured Logging**: JSON logs with request IDs
- **Azure Application Insights**: Performance tracking, error monitoring
- **Log Analytics**: Query and analyze logs

### Custom Metrics
Add custom metrics to track:
- API response times
- Business KPIs (users created, projects completed, etc.)
- Database query performance
- External API latency

### Alerts (via Azure Monitor)
- Error rate thresholds
- Latency degradation
- Container restart loops
- Resource utilization

---

## Updating Projects from Template

Copier supports updating generated projects when the template changes:

```bash
# Update to latest template version
cd my-backend
copier update

# Copier will merge changes, preserving your customizations
# Resolve any conflicts interactively
```

**Best Practices:**
- Keep customizations in separate files when possible
- Use `_skip_if_exists` in `copier.yml` for files users will customize
- Document any manual migration steps in `CHANGELOG.md`

---

## Security Best Practices

### Secrets Management
✅ **Use Azure Key Vault** for production secrets  
✅ **Never commit** `.env` files (add to `.gitignore`)  
✅ **Rotate secrets** regularly (Azure AD client secrets, DB passwords)  
✅ **Use managed identities** when possible (Container Apps → Cosmos DB)  

### Pre-commit Hooks
```bash
# Installed automatically during project generation
# Scans for secrets before commit
pre-commit run --all-files
```

### Input Validation
- All request bodies validated via Pydantic models
- Query parameters validated with `Query()` dependencies
- Path parameters validated with type annotations
- File uploads validated for type and size

### Rate Limiting
- Implement rate limiting middleware for public endpoints
- Use Azure API Management or NGINX for advanced rate limiting
- Track by IP address or authenticated user

---

## Performance Optimization

### Database
- Create indexes for frequently queried fields (see `utils/indexes.py`)
- Use projections to limit returned fields
- Implement pagination for large result sets
- Cache hot reads in Redis (optional)

### API
- Use `async def` for all I/O operations
- Stream large responses with `StreamingResponse`
- Enable gzip compression middleware
- Set appropriate cache headers

### Container
- Multi-stage Docker builds minimize image size
- uv speeds up dependency resolution (5-10x faster than pip)
- Health checks prevent traffic to unhealthy containers
- Auto-scaling based on HTTP traffic

---

## Troubleshooting

### Common Issues

#### "Module not found" errors
```bash
# Ensure virtual environment is activated
uv sync
source .venv/bin/activate  # or use `uv run`
```

#### Docker build fails on dependencies
```bash
# Clear Docker cache and rebuild
docker compose build --no-cache
```

#### Authentication errors
```bash
# Verify Azure AD configuration
az ad app show --id $AZURE_CLIENT_ID

# Check token validation
curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/
```

#### Cosmos DB connection errors
```bash
# Verify connection string format
# For MongoDB API: mongodb://...
# Check firewall rules in Azure Portal

# Test connection
uv run python -c "from motor.motor_asyncio import AsyncIOMotorClient; print(AsyncIOMotorClient('$MONGODB_URI').admin.command('ping'))"
```

### Logs

```bash
# Local development
uv run uvicorn backend.main:app --log-level debug

# Docker
docker compose logs -f backend

# Azure Container Apps
az containerapp logs show \
  --name my-backend \
  --resource-group rg-my-backend-dev \
  --follow
```

---

## Contributing to the Template

See `CONTRIBUTING.md` for guidelines on:
- Adding new features
- Testing template changes
- Submitting pull requests
- Release process

---

## Resources

### Documentation
- [FastAPI](https://fastapi.tiangolo.com/)
- [Pydantic](https://docs.pydantic.dev/latest/)
- [Motor (Async MongoDB)](https://motor.readthedocs.io/)
- [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/)
- [uv Package Manager](https://github.com/astral-sh/uv)
- [Copier](https://copier.readthedocs.io/)

### Azure Resources
- [Bicep Reference](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Azure AD Authentication](https://learn.microsoft.com/en-us/azure/active-directory/develop/)
- [Cosmos DB MongoDB API](https://learn.microsoft.com/en-us/azure/cosmos-db/mongodb/)

### Template Repository
- GitHub: `https://github.com/alkeme/alkeme-template-backend`
- Issues: Report bugs and request features
- Discussions: Ask questions and share feedback

---

## License

MIT License - see `LICENSE` file in generated projects.

---

## Support

- **Template Issues**: [GitHub Issues](https://github.com/alkeme/alkeme-template-backend/issues)
- **Generated Project Issues**: Create issues in your own repository
- **Alkeme Team**: Contact via email or internal channels

---

**Built with ❤️ by the Alkeme Engineering Team**

