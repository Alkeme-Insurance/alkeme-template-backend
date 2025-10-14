# Alkeme Backend Template - Project Specification

## Overview

This specification outlines the complete implementation of a Copier-based FastAPI backend template that generates production-ready Python applications with Docker containerization and Azure deployment infrastructure.

**Template Goals:**
- Generate FastAPI + MongoDB/Cosmos DB applications
- Production-ready Docker builds with uv package manager
- Azure Container Apps deployment via Bicep IaC
- CI/CD with GitHub Actions
- Feature flags for optional scaffolding (Azure Auth, Mailgun, Kanban, etc.)
- Modern Python tooling (uv, ruff, mypy, pytest)

**Based on:** `~/workspace/fast_azure/backend`  
**Sister Project:** `~/workspace/alkeme-template-frontend`  
**Target Deployment:** Azure Container Apps

---

## Phase 1: Template Repository Structure

### Step 1.1: Create Template Root Directory Structure

```
alkeme-template-backend/
├── copier.yml                          # Copier configuration
├── README.md                           # Template documentation
├── project.md                          # User-facing overview
├── project-spec.md                     # This specification
├── .copier-answers.yml.jinja          # Answer tracking
├── CONTRIBUTING.md                     # Contribution guidelines
├── CHANGELOG.md                        # Version history
├── LICENSE                             # MIT License
└── template/                           # Template files (will be rendered)
```

**Action:** Initialize git repository and create base structure

```bash
mkdir -p alkeme-template-backend
cd alkeme-template-backend
git init
gh repo create alkeme-template-backend --private --source=. --remote=origin
```

### Step 1.2: Create copier.yml Configuration

**File:** `copier.yml`

**Key Configuration:**
- Template version: 9.2+
- Questions: project metadata, feature flags, Azure config
- Post-generation tasks: uv sync, git init, pre-commit install
- Subdirectory: template/

**Question Categories:**

1. **Project Identity**
   - project_name (str): Display name
   - package_name (str): Python package name (snake_case)
   - project_description (str): Short description
   - owner_org (str): Organization name
   - author_name (str): Maintainer name
   - author_email (str): Contact email
   - repository_url (str): Git repository URL

2. **Feature Flags**
   - use_azure_auth (bool): Azure AD authentication (default: true)
   - deploy_cosmos_db (bool): Deploy Azure Cosmos DB in production (default: true)
     - *Note: Local development always uses MongoDB Docker container*
     - *This flag only affects Azure deployment (Bicep)*
   - use_mailgun (bool): Email service integration (default: false)
   - use_kanban (bool): Scaffold Kanban endpoints (default: false)
   - include_dashboard (bool): Dashboard/KPI endpoints (default: false)
   - use_verification (bool): Email verification logic (default: false)

3. **Development Tools**
   - use_cursor_rules (bool): Generate .cursorrules (default: true)
   - cursor_rules_style (str): fastapi-comprehensive/minimal (default: fastapi-comprehensive)
   - use_git_hooks (bool): Pre-commit secret scanning (default: true)
   - git_hook_tool (str): detect-secrets/pre-commit/custom (default: detect-secrets)
   - python_version (str): 3.10/3.11/3.12/3.13 (default: 3.12)
   - python_dependency_manager (str): uv/poetry/pipenv (default: uv)

4. **CI/CD Configuration**
   - ci_github_actions (bool): GitHub Actions workflow (default: true)
   - ci_include_azure_deploy (bool): Azure deployment job (default: true)
   - ci_run_on_pr (bool): Run CI on pull requests (default: true)
   - ci_run_on_push (bool): Run CI on push to main (default: true)

5. **Azure Configuration**
   - use_azure_deployment (bool): Generate Bicep IaC (default: true)
   - azure_region (str): Azure region (default: eastus)
   - azure_environment (str): dev/staging/prod (default: dev)
   - azure_subscription_id (str): Subscription ID (optional)
   - azure_resource_group (str): Resource group name (default: rg-{package_name}-{env})
   - azure_container_registry_name (str): ACR name (alphanumeric, default: acr{package_name}{env})
   - use_azure_key_vault (bool): Deploy Key Vault (default: false)
   - use_app_insights (bool): Application Insights (default: true)

6. **Azure AD (MSAL) Configuration**
   - azure_tenant_id (str): Tenant ID (optional)
   - azure_client_id (str): Application ID (optional)
   - azure_api_scope (str): API scope (default: api://{client_id}/user_impersonation)

7. **Development Configuration**
   - dev_port (int): Local dev server port (default: 8000)
   - docker_port (int): Docker host port (default: 8000)
   - mongodb_uri_dev (str): Local MongoDB URI (default: mongodb://localhost:27017)

**Post-Generation Tasks:**
```yaml
_tasks:
  - "[ ! -d .git ] && git init -q || true"
  - "uv sync"
  - "cp .env.example .env || true"
  - "{{ 'uv tool install detect-secrets' if git_hook_tool == 'detect-secrets' else 'uv tool install pre-commit' }}"
  - "git add ."
  - "git commit -m 'chore: scaffold {{ package_name }} via Copier'"
```

### Step 1.3: Create Template Answers File

**File:** `.copier-answers.yml.jinja`

**Purpose:** Track user answers for template updates (allows `copier update`)

---

## Phase 2: Template Files - Core Application

### Step 2.1: Python Project Configuration

**Files to Create:**

1. **`template/pyproject.toml.jinja`**
   - Project metadata (name, version, description)
   - Python version constraint (>={{python_version}})
   - Dependencies: FastAPI, Pydantic v2, Motor, PyMongo 3.x (Cosmos DB compat)
   - Optional dependencies: fastapi-azure-auth, mailgun-py, etc.
   - Dev dependencies: pytest, mypy, ruff, httpx
   - Build system: hatchling or setuptools
   - Tool configurations: ruff, mypy, pytest

**Core Dependencies:**
```toml
dependencies = [
    "fastapi[standard]>=0.117.1",
    "pydantic>=2.11.9",
    "pydantic-settings>=2.11.0",
    "motor>=2.5.0,<3.0.0",
    "pymongo>=3.13.0,<4.0.0",  # Cosmos DB MongoDB wire protocol 6 compatibility
    "dnspython>=2.0.0",
    {% if use_azure_auth %}
    "fastapi-azure-auth>=5.2.0",
    {% endif %}
    {% if use_mailgun %}
    "requests>=2.32.0",
    {% endif %}
]
```

**Dev Dependencies:**
```toml
[tool.uv.dev-dependencies]
pytest = ">=8.3.0"
pytest-asyncio = ">=0.24.0"
pytest-cov = ">=6.0.0"
httpx = ">=0.27.0"
mypy = ">=1.11.0"
ruff = ">=0.7.0"
```

**Tool Configurations:**
```toml
[tool.ruff]
line-length = 100
target-version = "py{{python_version|replace('.', '')}}"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM", "PL"]
ignore = ["E501", "PLR0913"]

[tool.mypy]
python_version = "{{python_version}}"
strict = true
warn_return_any = true
disallow_untyped_defs = true

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
python_files = "test_*.py"
python_functions = "test_*"
addopts = "--strict-markers --tb=short"
```

2. **`template/.python-version`**
   - Pin Python version for pyenv/asdf: `{{python_version}}`

3. **`template/.gitignore`**
   - Standard Python ignores (`.venv/`, `__pycache__/`, `*.pyc`)
   - Environment files (`.env`, `.env.local`)
   - IDE files (`.vscode/`, `.idea/`)
   - Build artifacts (`dist/`, `build/`, `*.egg-info/`)

### Step 2.2: Application Entry Points

**Files to Create:**

1. **`template/backend/__init__.py`**
   - Empty or version export

2. **`template/backend/main.py.jinja`**
   - FastAPI app initialization
   - CORS middleware configuration
   - Lifespan context manager (startup/shutdown)
   - Router registration
   - Health check endpoint
   - Swagger UI OAuth configuration (if Azure Auth)

**Key Patterns:**
```python
from contextlib import asynccontextmanager
from typing import AsyncGenerator
from fastapi import FastAPI, Security
from fastapi.middleware.cors import CORSMiddleware

{% if use_azure_auth %}
from backend.auth import azure_scheme
{% endif %}
from backend.config import settings
from backend.clients.mongo_db import mongo_lifespan
from backend.utils.seed import seed_initial_data
from backend.utils.indexes import ensure_indexes

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    async with mongo_lifespan():
        {% if use_azure_auth %}
        if settings.AZURE_TENANT_ID:
            await azure_scheme.openid_config.load_config()
        {% endif %}
        await seed_initial_data()
        await ensure_indexes()
        yield

app = FastAPI(
    title="{{ project_name }}",
    description="{{ project_description }}",
    version="0.1.0",
    {% if use_azure_auth %}
    swagger_ui_oauth2_redirect_url='/oauth2-redirect',
    swagger_ui_init_oauth={
        'usePkceWithAuthorizationCodeGrant': True,
        'clientId': settings.OPENAPI_CLIENT_ID,
        'scopes': settings.SCOPE_NAME,
    },
    {% endif %}
    lifespan=lifespan,
)

# CORS
if settings.BACKEND_CORS_ORIGINS:
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.get_cors_origins(),
        allow_credentials=True,
        allow_methods=['*'],
        allow_headers=['*'],
    )

# Health check (no auth required)
@app.get("/health")
async def health():
    return {"status": "healthy"}
```

3. **`template/backend/config.py.jinja`**
   - Pydantic Settings model
   - Load from `.env` file
   - Validate configuration
   - Computed fields for derived values

**Configuration Fields:**
```python
from pydantic import Field, computed_field
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file='.env',
        env_file_encoding='utf-8',
        case_sensitive=True,
    )
    
    # MongoDB
    MONGODB_URI: str = Field(default="mongodb://localhost:27017", alias="MONGODB_URI")
    
    # CORS
    BACKEND_CORS_ORIGINS: str = 'http://localhost:8000,http://localhost:5173'
    
    {% if use_azure_auth %}
    # Azure AD
    AZURE_TENANT_ID: str | None = Field(default=None, alias="AZURE_TENANT_ID")
    AZURE_CLIENT_ID: str | None = Field(default=None, alias="AZURE_CLIENT_ID")
    AZURE_CLIENT_SECRET: str | None = Field(default=None, alias="AZURE_CLIENT_SECRET")
    OPENAPI_CLIENT_ID: str = ""
    SCOPE_DESCRIPTION: str = "user_impersonation"
    {% endif %}
    
    {% if use_mailgun %}
    # Mailgun
    MAILGUN_API_KEY: str | None = Field(default=None, alias="MAILGUN_API_KEY")
    MAILGUN_DOMAIN: str | None = Field(default=None, alias="MAILGUN_DOMAIN")
    MAILGUN_FROM_EMAIL: str | None = Field(default=None, alias="MAILGUN_FROM_EMAIL")
    {% endif %}
    
    # Development
    DEV_NO_AUTH: bool = Field(default=False, alias="DEV_NO_AUTH")
    
    def get_cors_origins(self) -> list[str]:
        if isinstance(self.BACKEND_CORS_ORIGINS, str):
            return [origin.strip() for origin in self.BACKEND_CORS_ORIGINS.split(',')]
        return self.BACKEND_CORS_ORIGINS
    
    {% if use_azure_auth %}
    @computed_field
    @property
    def SCOPE_NAME(self) -> str:
        if self.AZURE_CLIENT_ID:
            return f'api://{self.AZURE_CLIENT_ID}/{self.SCOPE_DESCRIPTION}'
        return ''
    {% endif %}

settings = Settings()
```

4. **`template/backend/auth.py.jinja`** (conditional: use_azure_auth)
   - Azure AD authentication scheme
   - JWT token validation
   - User claims extraction

```python
from fastapi_azure_auth import SingleTenantAzureAuthorizationCodeBearer
from backend.config import settings

azure_scheme = SingleTenantAzureAuthorizationCodeBearer(
    app_client_id=settings.AZURE_CLIENT_ID,
    tenant_id=settings.AZURE_TENANT_ID,
    scopes=settings.SCOPES,
    allow_guest_users=True,
)
```

5. **`template/backend/dependencies.py`**
   - Reusable FastAPI dependencies
   - Database connection injection
   - Auth user injection

### Step 2.3: Core Application Code

**Directory Structure:**
```
template/backend/
├── __init__.py
├── main.py.jinja
├── config.py.jinja
├── auth.py.jinja (conditional)
├── dependencies.py.jinja
├── routers/
│   ├── __init__.py
│   ├── users.py.jinja
│   ├── projects.py.jinja
│   {% if use_kanban %}
│   └── kanban.py.jinja
│   {% endif %}
├── services/
│   ├── __init__.py
│   ├── user_service.py.jinja
│   └── project_service.py.jinja
├── models/
│   ├── __init__.py
│   ├── user.py.jinja
│   └── project.py.jinja
├── clients/
│   ├── __init__.py
│   ├── mongo_db.py.jinja
│   {% if use_mailgun %}
│   └── mailgun.py.jinja
│   {% endif %}
├── utils/
│   ├── __init__.py
│   ├── seed.py.jinja
│   ├── indexes.py.jinja
│   └── logging_config.py.jinja
└── internals/
    ├── __init__.py
    └── admin.py.jinja
```

**Key Implementation Files:**

**`template/backend/routers/users.py.jinja`:**
- GET /users - List users (with pagination)
- GET /users/{user_id} - Get user by ID
- POST /users - Create user
- PUT /users/{user_id} - Update user
- DELETE /users/{user_id} - Delete user
- Dependency injection for auth and database

**`template/backend/services/user_service.py.jinja`:**
- Business logic for user operations
- Data validation beyond Pydantic
- Orchestration of database calls
- Pure functions (no FastAPI dependencies)

**`template/backend/models/user.py.jinja`:**
- Pydantic models: UserCreate, UserUpdate, UserInDB, UserPublic
- Field validation
- Computed fields
- Example fields: email, name, is_active, created_at, updated_at

**`template/backend/clients/mongo_db.py.jinja`:**
- Motor client initialization
- Lifespan context manager
- Connection pooling
- Generic CRUD operations
- Error handling

```python
from motor.motor_asyncio import AsyncIOMotorClient, AsyncIOMotorDatabase
from contextlib import asynccontextmanager
from typing import AsyncGenerator
from backend.config import settings

_client: AsyncIOMotorClient | None = None
_database: AsyncIOMotorDatabase | None = None

@asynccontextmanager
async def mongo_lifespan() -> AsyncGenerator[None, None]:
    global _client, _database
    _client = AsyncIOMotorClient(settings.MONGODB_URI)
    _database = _client.get_default_database()
    yield
    if _client:
        _client.close()

def get_database() -> AsyncIOMotorDatabase:
    if _database is None:
        raise RuntimeError("Database not initialized")
    return _database
```

**`template/backend/utils/indexes.py.jinja`:**
- Create database indexes on startup
- Example: email unique index, created_at index

**`template/backend/utils/seed.py.jinja`:**
- Seed initial data (dev/staging only)
- Check if data already exists before seeding

### Step 2.4: Testing Setup

**Files to Create:**

1. **`template/tests/conftest.py.jinja`**
   - Pytest fixtures
   - Test database setup/teardown
   - Mock authenticated user
   - TestClient fixture

```python
import pytest
from fastapi.testclient import TestClient
from backend.main import app

@pytest.fixture
def client():
    return TestClient(app)

@pytest.fixture
def mock_user():
    return {"sub": "test-user-id", "email": "test@example.com"}
```

2. **`template/tests/test_main.py`**
   - Test health endpoint
   - Test CORS headers
   - Test Swagger UI access

3. **`template/tests/test_routers/test_users.py.jinja`**
   - Test user CRUD operations
   - Test authentication (if enabled)
   - Test input validation
   - Test error cases

---

## Phase 3: Docker Configuration

### Step 3.1: Multi-Stage Dockerfile with uv

**File:** `template/Dockerfile`

**Build Strategy:**
- Multi-stage build for minimal image size
- Use uv for fast dependency resolution (5-10x faster than pip)
- Cache dependencies layer separately from code
- Non-root user for security

```dockerfile
# Stage 1: Build dependencies
FROM python:{{python_version}}-slim AS builder

WORKDIR /app

# Install uv
RUN pip install uv

# Copy dependency files
COPY pyproject.toml ./

# Install dependencies (creates .venv)
RUN uv sync --no-dev

# Stage 2: Runtime
FROM python:{{python_version}}-slim

WORKDIR /app

# Install curl for healthcheck
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# Copy virtual environment from builder
COPY --from=builder /app/.venv /app/.venv

# Copy application code
COPY backend/ ./backend/

# Create non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Add .venv to PATH
ENV PATH="/app/.venv/bin:$PATH"

# Health check
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:80/health || exit 1

EXPOSE 80

CMD ["uvicorn", "backend.main:app", "--host", "0.0.0.0", "--port", "80"]
```

### Step 3.2: Docker Compose for Local Development

**File:** `template/docker-compose.yml.jinja`

**Services:**
- **backend**: FastAPI application
- **mongodb**: MongoDB 7.0 Docker image (always included for local development)

**Note:** The MongoDB service is always included in docker-compose.yml for local development, regardless of the `deploy_cosmos_db` flag. The flag only determines whether Azure Cosmos DB is deployed to Azure in production.

```yaml
version: "3.9"

services:
  backend:
    build:
      context: .
      dockerfile: Dockerfile
    image: {{ package_name }}:latest
    container_name: {{ package_name }}
    environment:
      MONGODB_URI: ${MONGODB_URI:-mongodb://mongodb:27017/{{package_name}}}
      {% if use_azure_auth %}
      AZURE_TENANT_ID: ${AZURE_TENANT_ID}
      AZURE_CLIENT_ID: ${AZURE_CLIENT_ID}
      AZURE_CLIENT_SECRET: ${AZURE_CLIENT_SECRET}
      OPENAPI_CLIENT_ID: ${OPENAPI_CLIENT_ID}
      {% endif %}
      BACKEND_CORS_ORIGINS: ${BACKEND_CORS_ORIGINS}
      DEV_NO_AUTH: ${DEV_NO_AUTH:-true}
    ports:
      - "${DOCKER_PORT:-8000}:80"
    depends_on:
      - mongodb
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]
      interval: 30s
      timeout: 3s
      retries: 3
    restart: unless-stopped

  mongodb:
    image: mongo:7
    container_name: {{ package_name }}-mongo
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    restart: unless-stopped

volumes:
  mongodb_data:
```

### Step 3.3: Environment Configuration

**File:** `template/.env.example`

```bash
# MongoDB Connection String
# Local Development (Docker Compose - default):
MONGODB_URI=mongodb://mongodb:27017/{{ package_name }}
# OR for local MongoDB on host machine:
# MONGODB_URI=mongodb://localhost:27017/{{ package_name }}

# Azure Cosmos DB (Production - set automatically by Bicep deployment):
# MONGODB_URI=mongodb://username:password@accountname.mongo.cosmos.azure.com:10255/{{ package_name }}?ssl=true&replicaSet=globaldb&retrywrites=false&maxIdleTimeMS=120000&appName=@accountname@

{% if use_azure_auth %}
# Azure AD Authentication
AZURE_TENANT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_CLIENT_SECRET=your-secret-here
OPENAPI_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
{% endif %}

# CORS Origins (comma-separated)
BACKEND_CORS_ORIGINS=http://localhost:3000,http://localhost:5173

# Development Mode (bypass auth)
DEV_NO_AUTH=true

{% if use_mailgun %}
# Mailgun (Email Service)
MAILGUN_API_KEY=your-key
MAILGUN_DOMAIN=mg.yourdomain.com
MAILGUN_FROM_EMAIL=noreply@yourdomain.com
{% endif %}

# Docker Port Mapping
DOCKER_PORT=8000
```

---

## Phase 4: Azure Deployment Infrastructure

### Step 4.1: Bicep Module Structure

**Directory:** `template/infra/`

```
template/infra/
├── main.bicep                          # Main orchestrator
├── main.bicepparam.jinja               # Parameter file
├── deploy.sh                           # Manual deployment script
├── deploy-ci.sh                        # CI/CD deployment script
├── README.md.jinja                     # Deployment documentation
└── modules/
    ├── container-registry.bicep        # Azure Container Registry
    ├── container-app.bicep             # Azure Container App
    ├── container-app-env.bicep         # Container App Environment
    ├── log-analytics.bicep             # Log Analytics Workspace
    ├── app-insights.bicep              # Application Insights
    ├── managed-identity.bicep          # User-assigned managed identity
    {% if use_cosmos_db %}
    ├── cosmos-db.bicep                 # Cosmos DB MongoDB API
    {% endif %}
    {% if use_azure_key_vault %}
    └── key-vault.bicep                 # Key Vault for secrets
    {% endif %}
```

### Step 4.2: Main Bicep Template

**File:** `template/infra/main.bicep`

**Parameters:**
```bicep
@description('Base name for all resources')
param projectName string

@description('Environment name (dev, staging, prod)')
param environmentName string

@description('Azure region for deployment')
param location string = resourceGroup().location

@description('Container image from ACR')
param containerImage string

{% if use_azure_auth %}
@secure()
@description('Azure AD Client ID')
param azureClientId string

@secure()
@description('Azure AD Tenant ID')
param azureTenantId string

@secure()
@description('Azure AD Client Secret')
param azureClientSecret string
{% endif %}

@description('Deploy Azure Cosmos DB for production')
param deployCosmosDb bool = true

@description('MongoDB connection string (only if deployCosmosDb=false, for BYO MongoDB)')
param mongodbUri string = ''
```

**Resources to Deploy:**
1. Log Analytics Workspace
2. Application Insights (linked to Log Analytics)
3. Container Registry (Basic SKU, upgradable)
4. User-Assigned Managed Identity
5. Container App Environment (with Log Analytics)
6. Container App (with managed identity, env vars, health probes)
7. Cosmos DB MongoDB API (conditional: deployCosmosDb=true, default)
8. Key Vault (conditional: use_azure_key_vault=true)

**Outputs:**
```bicep
output containerAppFQDN string = containerApp.properties.configuration.ingress.fqdn
output containerAppUrl string = 'https://${containerApp.properties.configuration.ingress.fqdn}'
output containerRegistryLoginServer string = containerRegistry.properties.loginServer
output containerRegistryName string = containerRegistry.name
output cosmosDbConnectionString string = deployCosmosDb ? cosmosDb.listConnectionStrings().connectionStrings[0].connectionString : ''
output cosmosDbEndpoint string = deployCosmosDb ? cosmosDb.properties.documentEndpoint : ''
```

### Step 4.3: Container App Module

**File:** `template/infra/modules/container-app.bicep`

**Configuration:**
- Ingress: External, HTTPS only, port 80
- Scaling: 1-10 replicas based on HTTP concurrent requests
- Resources: 0.5 CPU, 1GB memory per replica
- Environment variables from parameters
- Secrets from Key Vault (optional) or parameters
- Health probes: HTTP GET /health
- Managed identity for ACR pull

```bicep
resource containerApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: 'ca-${projectName}-${environmentName}'
  location: location
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${managedIdentityId}': {}
    }
  }
  properties: {
    managedEnvironmentId: containerAppEnvId
    configuration: {
      ingress: {
        external: true
        targetPort: 80
        transport: 'auto'
        allowInsecure: false
      }
      registries: [
        {
          server: acrLoginServer
          identity: managedIdentityId
        }
      ]
      secrets: [
        {% if use_azure_auth %}
        {
          name: 'azure-client-secret'
          value: azureClientSecret
        }
        {% endif %}
      ]
    }
    template: {
      containers: [
        {
          name: projectName
          image: containerImage
          resources: {
            cpu: json('0.5')
            memory: '1Gi'
          }
          env: [
            {
              name: 'MONGODB_URI'
              value: mongodbUri
            }
            {% if use_azure_auth %}
            {
              name: 'AZURE_TENANT_ID'
              value: azureTenantId
            }
            {
              name: 'AZURE_CLIENT_ID'
              value: azureClientId
            }
            {
              name: 'AZURE_CLIENT_SECRET'
              secretRef: 'azure-client-secret'
            }
            {% endif %}
            {
              name: 'DEV_NO_AUTH'
              value: 'false'
            }
          ]
          probes: [
            {
              type: 'Liveness'
              httpGet: {
                path: '/health'
                port: 80
              }
              initialDelaySeconds: 10
              periodSeconds: 30
            }
            {
              type: 'Readiness'
              httpGet: {
                path: '/health'
                port: 80
              }
              initialDelaySeconds: 5
              periodSeconds: 10
            }
          ]
        }
      ]
      scale: {
        minReplicas: 1
        maxReplicas: 10
        rules: [
          {
            name: 'http-scaling'
            http: {
              metadata: {
                concurrentRequests: '50'
              }
            }
          }
        ]
      }
    }
  }
}
```

### Step 4.4: Cosmos DB Module (Conditional)

**File:** `template/infra/modules/cosmos-db.bicep`

**Deployed When:** `deployCosmosDb=true` (default)

**Configuration:**
- API: MongoDB (wire protocol version 6 compatible)
- Consistency level: Session (balanced consistency and performance)
- Automatic failover: Enabled for high availability
- Capacity mode: 
  - Serverless for dev environment (pay-per-request)
  - Provisioned throughput for staging/prod (predictable cost)
- Database name: {{package_name}}
- Collections created by application:
  - users (always)
  - projects (always)
  - kanban_boards (if use_kanban=true)
  - dashboard_data (if include_dashboard=true)

**Connection String Injection:**
- Cosmos DB connection string is output from Bicep deployment
- Automatically injected as `MONGODB_URI` environment variable in Container App
- Application uses same Motor/PyMongo code as local development
- PyMongo 3.x required for Cosmos DB wire protocol compatibility

### Step 4.5: Key Vault Module (Optional)

**File:** `template/infra/modules/key-vault.bicep`

**Purpose:**
- Store Azure AD client secret
- Store MongoDB connection string
- Store Mailgun API key
- Grant Container App managed identity access

### Step 4.6: Bicep Parameters File

**File:** `template/infra/main.bicepparam.jinja`

```bicep
using './main.bicep'

param projectName = '{{ package_name }}'
param environmentName = '{{ azure_environment }}'
param location = '{{ azure_region }}'
param containerImage = '{{ azure_container_registry_name }}.azurecr.io/{{ package_name }}:latest'

{% if use_azure_auth %}
param azureClientId = '{{ azure_client_id or "your-client-id-here" }}'
param azureTenantId = '{{ azure_tenant_id or "your-tenant-id-here" }}'
param azureClientSecret = 'your-client-secret-here'  // Read from Key Vault or GitHub Secret
{% endif %}

{% if use_cosmos_db %}
param deployCosmosDb = true
{% else %}
param mongodbUri = 'your-mongodb-connection-string'
{% endif %}
```

### Step 4.7: Deployment Scripts

**File:** `template/infra/deploy.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Manual deployment script for local/manual deployments

PROJECT_NAME="{{ package_name }}"
ENVIRONMENT="{{ azure_environment }}"
RESOURCE_GROUP="{{ azure_resource_group }}"
LOCATION="{{ azure_region }}"

echo "🚀 Deploying ${PROJECT_NAME} to Azure (${ENVIRONMENT})"

# Login to Azure (if not already logged in)
az account show &>/dev/null || az login

# Set subscription
{% if azure_subscription_id %}
az account set --subscription "{{ azure_subscription_id }}"
{% endif %}

# Create resource group if it doesn't exist
az group create --name "${RESOURCE_GROUP}" --location "${LOCATION}"

# Deploy Bicep template
echo "📦 Deploying infrastructure..."
az deployment group create \
  --resource-group "${RESOURCE_GROUP}" \
  --template-file main.bicep \
  --parameters main.bicepparam \
  --output table

# Get Container App URL
CONTAINER_APP_URL=$(az deployment group show \
  --resource-group "${RESOURCE_GROUP}" \
  --name main \
  --query properties.outputs.containerAppUrl.value \
  --output tsv)

echo "✅ Deployment complete!"
echo "🌐 Container App URL: ${CONTAINER_APP_URL}"
```

**File:** `template/infra/deploy-ci.sh`**

- Non-interactive version for CI/CD
- Use service principal authentication
- Fail fast on errors
- Output JSON for parsing

---

## Phase 5: CI/CD Configuration

### Step 5.1: GitHub Actions Workflow

**File:** `template/.github/workflows/ci-cd.yml.jinja`

**Triggers:**
- Push to main branch
- Pull requests to main
- Manual workflow dispatch

**Jobs:**

**1. lint-type-test:**
```yaml
lint-type-test:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    
    - name: Install uv
      uses: astral-sh/setup-uv@v5
      with:
        version: "latest"
    
    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: "{{ python_version }}"
    
    - name: Install dependencies
      run: uv sync
    
    - name: Lint with ruff
      run: uv run ruff check backend
    
    - name: Type check with mypy
      run: uv run mypy backend
    
    - name: Run tests
      run: uv run pytest --cov=backend --cov-report=xml
    
    - name: Upload coverage
      uses: codecov/codecov-action@v4
      with:
        file: ./coverage.xml
```

**2. docker-build:**
```yaml
docker-build:
  runs-on: ubuntu-latest
  needs: lint-type-test
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  steps:
    - uses: actions/checkout@v4
    
    - name: Build Docker image
      run: docker build -t {{ package_name }}:ci .
```

**3. deploy-azure:**
```yaml
deploy-azure:
  runs-on: ubuntu-latest
  needs: [lint-type-test, docker-build]
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  environment: {{ azure_environment }}
  steps:
    - uses: actions/checkout@v4
    
    - name: Azure Login
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: ACR Login
      run: |
        az acr login --name {{ azure_container_registry_name }}
    
    - name: Build and push Docker image
      run: |
        IMAGE_TAG={{ azure_container_registry_name }}.azurecr.io/{{ package_name }}:${{ github.sha }}
        docker build -t $IMAGE_TAG .
        docker push $IMAGE_TAG
        docker tag $IMAGE_TAG {{ azure_container_registry_name }}.azurecr.io/{{ package_name }}:latest
        docker push {{ azure_container_registry_name }}.azurecr.io/{{ package_name }}:latest
    
    - name: Deploy to Azure Container Apps
      run: |
        cd infra
        az deployment group create \
          --resource-group {{ azure_resource_group }} \
          --template-file main.bicep \
          --parameters main.bicepparam \
          --parameters containerImage={{ azure_container_registry_name }}.azurecr.io/{{ package_name }}:${{ github.sha }} \
          --output table
    
    - name: Get Container App URL
      id: get-url
      run: |
        URL=$(az containerapp show \
          --name ca-{{ package_name }}-{{ azure_environment }} \
          --resource-group {{ azure_resource_group }} \
          --query properties.configuration.ingress.fqdn \
          --output tsv)
        echo "url=https://$URL" >> $GITHUB_OUTPUT
    
    - name: Comment PR with deployment URL
      uses: actions/github-script@v7
      if: github.event_name == 'push'
      with:
        script: |
          github.rest.repos.createCommitComment({
            owner: context.repo.owner,
            repo: context.repo.repo,
            commit_sha: context.sha,
            body: '🚀 Deployed to Azure: ${{ steps.get-url.outputs.url }}'
          })
```

### Step 5.2: Azure Service Principal Setup Documentation

**File:** `template/.github/AZURE_SETUP.md.jinja`

**Instructions:**
1. Create service principal with Contributor role
2. Grant ACR push permissions (AcrPush role)
3. Configure GitHub secrets:
   - `AZURE_CREDENTIALS` (JSON output from `az ad sp create-for-rbac --sdk-auth`)
   - `ACR_LOGIN_SERVER`
   - `AZURE_SUBSCRIPTION_ID`

---

## Phase 6: Developer Experience & Documentation

### Step 6.1: Cursor Rules

**File:** `template/.cursorrules.jinja`

**Content Sections:**
1. **Architecture**: Layered architecture (routers → services → clients)
2. **Code Style**: PEP 8, type hints, descriptive names
3. **FastAPI Specifics**: async def for I/O, def for CPU-bound, dependencies pattern
4. **Error Handling**: HTTPException, guard clauses, centralized logging
5. **Security**: Input validation, secret management, CORS, rate limiting
6. **Testing**: Pytest, async tests, fixtures, mocking
7. **Performance**: Async operations, indexing, caching, pagination
8. **Database**: Motor async operations, projections, transactions

### Step 6.2: Git Hooks Configuration

**Files:**

1. **`template/.pre-commit-config.yaml`** (if using pre-commit)
```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: check-merge-conflict
  
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.7.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

2. **`template/.secrets.baseline`**
   - Baseline file for detect-secrets
   - Pre-populated with known false positives

### Step 6.3: Main README

**File:** `template/README.md.jinja`

**Sections:**
1. Project title and description
2. Prerequisites
3. Quick start (local development)
4. Docker development
5. Azure deployment
6. Environment variables reference
7. Project structure
8. Available commands (uv run, pytest, docker, etc.)
9. Testing strategy
10. API documentation (link to /docs)
11. Contributing guidelines
12. License

### Step 6.4: Infrastructure README

**File:** `template/infra/README.md.jinja`

**Sections:**
1. Azure prerequisites
2. Service principal setup
3. Manual deployment steps
4. CI/CD deployment (automated)
5. Updating configuration
6. Monitoring and logs
7. Scaling configuration
8. Cost estimation
9. Troubleshooting

---

## Phase 7: Testing & Validation

### Step 7.1: Template Testing Strategy

**Create:** `tests/` directory in template repo root (not rendered)

**Test Files:**
- `tests/test_template_generation.py` - Test Copier rendering with various options
- `tests/test_docker_build.sh` - Validate Docker build succeeds
- `tests/test_azure_bicep.sh` - Validate Bicep with `az bicep build`

**Test Process:**
1. Generate project with various option combinations
2. Verify all expected files created
3. Run `uv sync`
4. Run lint, type check, tests
5. Build Docker image
6. Validate Bicep templates
7. Deploy to test Azure environment

### Step 7.2: Example Projects

**Create:** `examples/` directory

**Example Variations:**
- `minimal/` - No Azure auth, no features, local MongoDB
- `full-featured/` - All flags enabled, Cosmos DB, Mailgun, etc.
- `azure-auth-only/` - Azure AD auth but minimal features

### Step 7.3: CI for Template Repository

**File:** `.github/workflows/test-template.yml`

**Jobs:**
- Test template generation with all option combinations
- Validate generated projects
- Run generated project tests
- Validate Docker builds
- Lint Bicep templates

---

## Phase 8: Template Repository Documentation

### Step 8.1: Template README

**File:** `README.md` (in template repo root)

**Content:**
1. What this template generates
2. Features and capabilities
3. Prerequisites for using template
4. Usage: `uvx copier copy gh:alkeme/alkeme-template-backend my-backend`
5. Template options reference
6. Post-generation steps
7. Updating projects from template
8. Contributing to template
9. Support and issues

### Step 8.2: Template Contribution Guide

**File:** `CONTRIBUTING.md`

**Sections:**
- How to modify template
- Testing changes locally
- Adding new optional features
- Updating dependencies
- Release process
- Code of conduct

### Step 8.3: Changelog

**File:** `CHANGELOG.md`

**Format:** Keep a Changelog format
- Unreleased section
- Version history with dates
- Categories: Added, Changed, Fixed, Security

### Step 8.4: License

**File:** `LICENSE`

**Recommendation:** MIT License

---

## Implementation Checklist

### Pre-Flight
- [ ] Install Copier 9.2+: `uv tool install copier`
- [ ] Install uv: `curl -LsSf https://astral.sh/uv/install.sh | sh`
- [ ] Install Azure CLI: `brew install azure-cli` or equivalent
- [ ] Install GitHub CLI: `brew install gh`
- [ ] Set up test Azure subscription
- [ ] Create GitHub organization/repo for template

### Phase 1: Foundation (Week 1)
- [ ] Initialize template repository structure
- [ ] Create `copier.yml` with all questions
- [ ] Define directory structure in `template/`
- [ ] Set up template testing framework
- [ ] Document template usage in `README.md`

### Phase 2: Core Application (Week 1-2)
- [ ] Create `pyproject.toml.jinja` with dependencies
- [ ] Implement `main.py.jinja` (FastAPI app, lifespan)
- [ ] Implement `config.py.jinja` (Pydantic Settings)
- [ ] Implement `auth.py.jinja` (Azure AD, conditional)
- [ ] Create MongoDB client (`mongo_db.py.jinja`)
- [ ] Implement routers: users, projects, kanban (conditional)
- [ ] Implement services: user_service, project_service
- [ ] Implement models: user, project (Pydantic v2)
- [ ] Create utilities: seed.py, indexes.py, logging_config.py
- [ ] Implement Mailgun client (conditional)
- [ ] Set up pytest configuration and fixtures
- [ ] Write example tests for routers
- [ ] Test local development flow (`uv run uvicorn`)

### Phase 3: Docker (Week 2)
- [ ] Create multi-stage `Dockerfile` with uv
- [ ] Create `docker-compose.yml` (backend + MongoDB)
- [ ] Create `.env.example` with all variables
- [ ] Test Docker build locally
- [ ] Test Docker Compose stack
- [ ] Verify health check endpoint works
- [ ] Optimize image size (<200MB)

### Phase 4: Azure Infrastructure (Week 2-3)
- [ ] Create `infra/main.bicep` orchestrator
- [ ] Implement Container Registry module
- [ ] Implement Log Analytics module
- [ ] Implement App Insights module
- [ ] Implement Managed Identity module
- [ ] Implement Container App Environment module
- [ ] Implement Container App module
- [ ] Implement Cosmos DB module (conditional)
- [ ] Implement Key Vault module (conditional)
- [ ] Create `main.bicepparam.jinja` template
- [ ] Write `deploy.sh` script
- [ ] Write `deploy-ci.sh` script
- [ ] Test deployment to Azure dev environment
- [ ] Verify Container App runs correctly
- [ ] Test Azure AD authentication in Azure
- [ ] Test Cosmos DB connection (if enabled)
- [ ] Document Azure setup in `infra/README.md`

### Phase 5: CI/CD (Week 3)
- [ ] Create `.github/workflows/ci-cd.yml.jinja`
- [ ] Configure `lint-type-test` job
- [ ] Configure `docker-build` job
- [ ] Configure `deploy-azure` job
- [ ] Set up Azure service principal
- [ ] Configure GitHub secrets
- [ ] Test full CI/CD pipeline
- [ ] Document service principal setup in `.github/AZURE_SETUP.md`
- [ ] Add deployment status badges to README

### Phase 6: Developer Experience (Week 3-4)
- [ ] Write `.cursorrules` for AI-assisted development
- [ ] Configure pre-commit hooks (detect-secrets or pre-commit)
- [ ] Create `.secrets.baseline`
- [ ] Write comprehensive `README.md.jinja`
- [ ] Write infrastructure `infra/README.md.jinja`
- [ ] Create deployment documentation
- [ ] Add troubleshooting guide
- [ ] Create contribution guidelines

### Phase 7: Testing & Quality (Week 4)
- [ ] Write template generation tests (pytest)
- [ ] Create example projects (minimal, full)
- [ ] Test all feature flag combinations
- [ ] Validate Docker builds for all variations
- [ ] Test Azure deployments (all variations)
- [ ] Performance test Container App (load testing)
- [ ] Security scan Docker images (Trivy, Grype)
- [ ] Validate Bicep templates (`az bicep build`)

### Phase 8: Documentation & Release (Week 4)
- [ ] Write template `README.md`
- [ ] Create `CONTRIBUTING.md`
- [ ] Initialize `CHANGELOG.md`
- [ ] Add `LICENSE` file (MIT)
- [ ] Create template usage examples
- [ ] Record demo video (optional)
- [ ] Publish to GitHub with `copier-template` topic
- [ ] Announce in team/community

### Phase 9: Optional Enhancements (Future)
- [ ] Add Azure Front Door support (global CDN, WAF)
- [ ] Add Azure API Management integration
- [ ] Add Redis caching support
- [ ] Add Celery/background task support
- [ ] Add multi-environment parameter files (dev/staging/prod)
- [ ] Add custom monitoring dashboards (Azure Workbooks)
- [ ] Add cost optimization recommendations
- [ ] Add GraphQL support (Strawberry)
- [ ] Add gRPC support
- [ ] Add WebSocket support

---

## Key Decision Points

### 1. Python Package Manager
**Options:**
- **uv** (recommended) - 10-100x faster than pip, Rust-based, modern
- **poetry** - Popular, feature-rich, slower
- **pipenv** - Older, less active development

**Recommendation:** uv (fastest, most modern, great Docker support)

### 2. Database Strategy

**Local Development:**
- **MongoDB Docker container** - Always included in docker-compose.yml
- Easy setup, no external dependencies
- Perfect for local testing and development

**Production Deployment:**
- **Azure Cosmos DB (MongoDB API)** - Default deployment target
  - Deployed via Bicep when `deploy_cosmos_db=true` (default)
  - Fully managed, globally distributed, auto-scaling
  - MongoDB wire protocol compatible (PyMongo 3.x)
  - Connection string automatically injected into Container App

**Alternative Production Option:**
- **Bring Your Own MongoDB** - Set `deploy_cosmos_db=false`
  - Use existing MongoDB instance (Atlas, self-hosted, etc.)
  - Manually configure `MONGODB_URI` in deployment
  - Cost-effective for smaller deployments

**Recommendation:** Local MongoDB for dev (always), Cosmos DB for production (default template behavior)

### 3. Authentication
**Options:**
- **Azure AD (Entra ID)** - Enterprise SSO, native Azure integration
- **Auth0** - Multi-platform, social logins, higher cost
- **Custom JWT** - Full control, more work

**Recommendation:** Azure AD (native Azure, enterprise-ready)

### 4. Container Deployment Target
**Options:**
- **Azure Container Apps** (recommended) - Serverless, auto-scaling, simple
- **Azure Kubernetes Service (AKS)** - More control, complexity, higher cost
- **Azure App Service** - Traditional PaaS, less flexible

**Recommendation:** Container Apps (best balance of simplicity and power)

### 5. Secrets Management
**Options:**
- **Azure Key Vault** - Native, integrated, audit logs
- **Environment variables** - Simple, less secure
- **HashiCorp Vault** - Advanced, self-hosted

**Recommendation:** Key Vault for production, env vars for dev (feature flag)

---

## Success Criteria

### For Generated Projects:
✅ Application runs locally with `uv run uvicorn`  
✅ All tests pass (lint, type check, pytest)  
✅ Docker image builds successfully (<200MB)  
✅ Docker Compose stack runs locally  
✅ Azure deployment succeeds via Bicep  
✅ Container App serves API with HTTPS  
✅ Health check endpoint responds correctly  
✅ Azure AD authentication works (if enabled)  
✅ MongoDB/Cosmos DB connection succeeds  
✅ Swagger UI accessible at /docs  
✅ CI/CD pipeline completes successfully  
✅ API response time <500ms (p95)  
✅ No security vulnerabilities (Trivy scan)  

### For Template Itself:
✅ Template generates successfully for all option combinations  
✅ Clear, comprehensive documentation  
✅ Working examples for common scenarios  
✅ Automated testing validates template quality  
✅ Version control tracks user answers for updates  
✅ Update workflow preserves user customizations  
✅ Community can contribute improvements  
✅ Listed in Copier template gallery  

---

## Maintenance & Updates

### Regular Tasks:
- **Monthly:** Update Python dependencies (`uv lock --upgrade`)
- **Quarterly:** Review Azure best practices, update Bicep templates
- **As needed:** Security patches, critical bug fixes

### Version Strategy:
- **Major (X.0.0):** Breaking changes, major new features
- **Minor (1.X.0):** New optional features, backward-compatible
- **Patch (1.0.X):** Bug fixes, documentation updates

### Update Workflow for Generated Projects:
```bash
cd my-backend
copier update
# Copier merges changes, preserving customizations
# Resolve conflicts interactively
```

---

## Timeline Summary

**Week 1:** Foundation + Core Application  
**Week 2:** Docker + Azure Infrastructure  
**Week 3:** CI/CD + Developer Experience  
**Week 4:** Testing + Documentation + Release  

**Total:** 4 weeks for production-ready template  
**Optional Enhancements:** Ongoing  

---

## Next Steps

1. **Review this specification** with team for feedback
2. **Set up template repository** on GitHub
3. **Create initial `copier.yml`** with basic structure
4. **Implement Phase 1 & 2** (Foundation + Core App)
5. **Test locally** before proceeding to Docker/Azure phases
6. **Iterate based on real usage** and team feedback

---

## Resources & References

### Copier Documentation
- [Copier Official Docs](https://copier.readthedocs.io/)
- [Creating Templates](https://copier.readthedocs.io/en/stable/creating/)
- [Configuring Templates](https://copier.readthedocs.io/en/stable/configuring/)

### FastAPI & Python
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Pydantic v2 Documentation](https://docs.pydantic.dev/latest/)
- [Motor (Async MongoDB)](https://motor.readthedocs.io/)
- [uv Documentation](https://github.com/astral-sh/uv)

### Azure Documentation
- [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/)
- [Azure Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Azure Container Registry](https://learn.microsoft.com/en-us/azure/container-registry/)
- [Azure Cosmos DB MongoDB API](https://learn.microsoft.com/en-us/azure/cosmos-db/mongodb/)
- [fastapi-azure-auth](https://github.com/Intility/fastapi-azure-auth)

### Tools
- [Ruff (Linter)](https://docs.astral.sh/ruff/)
- [Mypy (Type Checker)](https://mypy.readthedocs.io/)
- [Pytest](https://docs.pytest.org/)
- [detect-secrets](https://github.com/Yelp/detect-secrets)

---

*This specification is a living document. Update as you learn from implementation and usage.*

---

**Version:** 1.0.0  
**Last Updated:** 2025-10-14  
**Maintainer:** Alkeme Engineering Team

