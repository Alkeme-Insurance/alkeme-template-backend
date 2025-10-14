# Alkeme Backend Template

**Production-ready FastAPI backend template with MongoDB, Azure deployment, and modern Python tooling.**

This is a [Copier](https://copier.readthedocs.io/) template for scaffolding FastAPI backend applications with async MongoDB, Docker containerization, Azure Container Apps deployment, and optional Azure AD authentication.

## Features

- ⚡ **Modern Backend Stack**: FastAPI + Pydantic v2 + Motor (async MongoDB)
- 🐍 **Python 3.10-3.13** support
- 📦 **uv Package Manager**: Fast, reliable Python package management (10-100x faster than pip)
- 🐳 **Docker Ready**: Multi-stage builds optimized for production
- 🗄️ **Database Strategy**:
  - Local dev: MongoDB 7.0 Docker container
  - Production: Azure Cosmos DB (MongoDB API) via Bicep IaC
- 🔐 **Authentication**: Optional Azure AD (MSAL) integration
- ☁️ **Azure Deployment**: Bicep infrastructure templates for Container Apps
- 🧪 **Testing**: Pytest with async support and coverage
- 🔧 **Code Quality**: Ruff (linting + formatting) + Mypy (type checking)
- 🛡️ **Security**: Pre-commit hooks with detect-secrets

## Prerequisites

- Python 3.10+
- [Copier](https://copier.readthedocs.io/) - `pip install copier` or `uvx copier`
- [uv](https://github.com/astral-sh/uv) - `curl -LsSf https://astral.sh/uv/install.sh | sh`
- Docker and Docker Compose (for local development)
- Azure CLI (for Azure deployment)

## Quick Start

### Generate a New Project

```bash
# Using uvx (recommended)
uvx copier copy gh:alkeme/alkeme-template-backend my-backend

# Or using copier directly
copier copy https://github.com/alkeme/alkeme-template-backend my-backend
```

Follow the prompts to configure your project:

- Project name and description
- Python version (3.10-3.13)
- Azure AD authentication (yes/no)
- Cosmos DB deployment (yes/no)
- Git hooks for secret scanning

### Run the Generated Project

```bash
cd my-backend

# Local development (Python)
uv sync
uv run uvicorn backend.main:app --reload

# Or with Docker
docker compose up --build
```

Visit:
- API: http://localhost:8000
- Swagger UI: http://localhost:8000/docs
- ReDoc: http://localhost:8000/redoc

## Project Structure

Generated projects follow a layered architecture:

```
my-backend/
├── backend/                 # Main application code
│   ├── main.py             # FastAPI app & lifespan
│   ├── config.py           # Settings management
│   ├── auth.py             # Azure AD auth (optional)
│   ├── routers/            # API endpoints (HTTP layer)
│   │   ├── users.py
│   │   └── projects.py
│   ├── services/           # Business logic (pure Python)
│   │   ├── user_service.py
│   │   └── project_service.py
│   ├── models/             # Pydantic schemas
│   │   ├── user.py
│   │   └── project.py
│   ├── clients/            # External integrations
│   │   └── mongo_db.py
│   └── utils/              # Utilities
│       ├── indexes.py
│       └── seed.py
├── tests/                  # Test suite
│   ├── conftest.py
│   └── test_main.py
├── pyproject.toml          # Python dependencies (uv)
├── Dockerfile              # Multi-stage Docker build
├── docker-compose.yml      # Local development stack
└── env.example             # Environment template
```

## Architecture

Projects follow a strict layered architecture:

```
Routers (HTTP) → Services (Business Logic) → Clients (Data/External APIs)
```

- **Routers**: Handle HTTP requests, validate input, return responses (FastAPI)
- **Services**: Pure business logic, no FastAPI dependencies
- **Clients**: Database operations, external API calls

## Configuration

### Feature Flags

During project generation, you can enable/disable features:

- **Azure AD Authentication**: Adds fastapi-azure-auth for Microsoft SSO
- **Cosmos DB Deployment**: Includes Bicep templates for Azure Cosmos DB
- **Git Hooks**: Sets up detect-secrets or pre-commit for secret scanning

### Database Strategy

- **Local Development**: Always uses MongoDB 7.0 Docker container (included in docker-compose.yml)
- **Production**: Optionally deploys Azure Cosmos DB (MongoDB API) via Bicep
- **BYO MongoDB**: Can connect to any MongoDB-compatible database via `MONGODB_URI`

## Development Workflow

Generated projects include:

```bash
# Code quality
uv run ruff format backend    # Format code
uv run ruff check backend --fix  # Lint code
uv run mypy backend           # Type check

# Testing
uv run pytest                 # Run tests
uv run pytest --cov=backend   # With coverage

# Docker
docker compose up --build     # Run full stack
docker compose logs -f        # View logs
```

## Deployment

### Azure Container Apps

Generated projects include Bicep infrastructure templates in `infra/`:

```bash
cd infra
./deploy.sh
```

This deploys:
- Azure Container Apps (backend)
- Azure Cosmos DB (optional, MongoDB API)
- Azure Container Registry
- Managed identities and networking

### Docker Only

For non-Azure deployments:

```bash
docker build -t my-backend:latest .
docker push my-registry/my-backend:latest
```

## Sister Templates

- **Frontend**: [alkeme-template-frontend](https://github.com/alkeme/alkeme-template-frontend) - React + TypeScript + Vite

## Updating Generated Projects

After generating a project, you can update it when the template changes:

```bash
cd my-backend
copier update
```

Copier will show you what changed and let you resolve conflicts.

## Contributing

Contributions welcome! Please:

1. Fork this repository
2. Create a feature branch
3. Make your changes to the `template/` directory
4. Test with `copier copy . /tmp/test-backend`
5. Submit a pull request

## Template Development

### Testing the Template

```bash
# Generate a test project
copier copy . /tmp/test-backend

# Test the generated project
cd /tmp/test-backend
uv sync
uv run pytest
docker compose up --build
```

### Template Structure

- `copier.yml`: Template configuration and questions
- `template/`: Template files (Jinja2 syntax)
  - Files ending in `.jinja` will have the suffix removed during generation
  - Use `{{ variable }}` for substitution
  - Use `{% if condition %}` for conditional content

## License

Alkeme - Intelligent Solutions

## Support

- Issues: https://github.com/alkeme/alkeme-template-backend/issues
- Email: dev@alkeme.com

---

**Related Projects:**
- Based on [fast_azure](https://github.com/alkeme/fast_azure) backend architecture
- Sister project: [alkeme-template-frontend](https://github.com/alkeme/alkeme-template-frontend)

