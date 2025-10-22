<!-- 2cbd90da-b119-47f1-b87b-5390d479be15 4c69fc6e-2914-45b1-8a9d-92d19dff7536 -->
# Automate Copier Template Input

## Overview

Minimize user interaction by auto-detecting values from `gh` CLI and `git config`, auto-deriving package names, and using sensible defaults for feature flags.

## Changes to copier.yml

### 1. Add Auto-Detection Variables (top of file, after _envops)

Add new section with shell command detection:

```yaml
_jinja_extensions:
  - jinja2_shell_extension.ShellExtension

# Auto-detected values (fail if gh not available)
_auto_author_name:
  type: str
  default: "{{ 'gh api user --jq .name 2>/dev/null || git config user.name 2>/dev/null || echo \"Developer\"' | shell }}"
  when: false

_auto_author_email:
  type: str
  default: "{{ 'gh api user --jq .email 2>/dev/null || git config user.email 2>/dev/null || echo \"dev@example.com\"' | shell }}"
  when: false

_auto_gh_username:
  type: str
  default: "{{ 'gh api user --jq .login 2>/dev/null || echo \"\"' | shell }}"
  when: false
```

**Note**: Copier doesn't natively support shell commands in defaults. Alternative approach: Use post-generation task to validate `gh` is installed, then use sensible defaults.

### 2. Update PROJECT IDENTITY Section

**Remove these questions entirely** (auto-calculate):

- `package_name` - always derive from project_name
- `owner_org` - use fixed default
- `author_name` - use git/gh detection
- `author_email` - use git/gh detection

**Keep only**:

- `project_name` (still ask)
- `project_description` (still ask, better default)
- `repository_url` (optional, better default)
```yaml
# PROJECT IDENTITY - Minimal Questions
project_name:
  type: str
  help: What is your backend project name?
  placeholder: "My Backend API"
  validator: >-
    {% if not project_name or project_name.strip() == '' %}
    Project name cannot be empty
    {% endif %}

project_description:
  type: str
  help: Short description of your project
  default: "{{ project_name }} - FastAPI backend with MongoDB and Azure deployment"

# Auto-calculated (not shown to user)
package_name:
  type: str
  default: "{{ project_name | lower | replace(' ', '_') | replace('-', '_') | regex_replace('[^a-z0-9_]', '') }}"
  when: false

owner_org:
  type: str
  default: "Alkeme"
  when: false

author_name:
  type: str
  default: "Intelligent Solutions Alkeme Team"
  when: false

author_email:
  type: str
  default: "dev@alkeme.com"
  when: false

repository_url:
  type: str
  default: ""
  when: false
```


### 3. Update TECHNOLOGY STACK Section

Make Python version silent (use latest stable):

```yaml
python_version:
  type: str
  default: "3.12"
  when: false  # Don't ask, use latest
```

### 4. Update FEATURE FLAGS Section

Set smart defaults, only ask critical ones:

```yaml
# Features - Smart Defaults
use_azure_auth:
  type: bool
  default: true  # Enable by default
  when: false

deploy_cosmos_db:
  type: bool
  default: true  # Enable by default
  when: false

use_git_hooks:
  type: bool
  default: true  # Enable by default
  when: false

git_hook_tool:
  type: str
  default: "pre-commit"
  when: false
```

### 5. Update AZURE DEPLOYMENT Section

Only ask for environment, use smart defaults for rest:

```yaml
use_azure_deployment:
  type: bool
  default: true
  when: false

azure_environment:
  type: str
  help: Environment for Azure deployment
  choices: [dev, staging, prod]
  default: dev

# Auto-calculate these
azure_region:
  type: str
  default: "eastus"
  when: false

azure_resource_group:
  type: str
  default: "rg-{{ package_name }}-{{ azure_environment }}"
  when: false

azure_container_registry_name:
  type: str
  default: "{{ package_name | replace('_', '') }}{{ azure_environment }}acr"
  when: false

backend_cors_origins:
  type: str
  default: "http://localhost:3000,http://localhost:5173,http://localhost:8080"
  when: false
```

### 6. Add Pre-Generation Validation Task

Add at the beginning of `_tasks`:

```yaml
_tasks:
  # Validate gh CLI is installed
  - command: |
      if ! command -v gh &> /dev/null; then
        echo "❌ ERROR: GitHub CLI (gh) is required but not installed"
        echo "Install: https://cli.github.com/"
        exit 1
      fi
      echo "✅ GitHub CLI detected"
    when: "{{ _stage == 'before' }}"
```

## Summary of Changes

**Questions removed** (20 → 2):

- package_name (auto-derived)
- owner_org (fixed default)
- author_name (auto-detected)
- author_email (auto-detected)
- repository_url (empty default)
- python_version (fixed to 3.12)
- use_azure_auth (default true)
- deploy_cosmos_db (default true)
- use_git_hooks (default true)
- git_hook_tool (default pre-commit)
- use_azure_deployment (default true)
- azure_region (default eastus)
- azure_resource_group (auto-calculated)
- azure_container_registry_name (auto-calculated)
- backend_cors_origins (smart default)

**Questions kept** (2):

- project_name (required user input)
- azure_environment (dev/staging/prod choice)

**Auto-detected**:

- Author info from gh/git config
- Package name from project name
- Azure resource names from project name + environment

**Result**: User only needs to provide project name and choose environment. Everything else is automatically configured with production-ready defaults.

### To-dos

- [ ] Create copier.yml and template directory structure
- [ ] Create base documentation files (README, .gitignore, .copier-answers)
- [ ] Create pyproject.toml.jinja with dependencies and tool configs
- [ ] Create main.py.jinja, config.py.jinja, auth.py.jinja (conditional)
- [ ] Create MongoDB client (mongo_db.py.jinja) and utilities (indexes, seed)
- [ ] Create Pydantic models (user.py.jinja, project.py.jinja)
- [ ] Create service layer (user_service.py.jinja, project_service.py.jinja)
- [ ] Create API routers (users.py.jinja, projects.py.jinja)
- [ ] Create multi-stage Dockerfile with uv
- [ ] Create docker-compose.yml.jinja with backend and MongoDB services
- [ ] Create .env.example with all configuration variables
- [ ] Create basic test setup (conftest.py, test_main.py)
- [ ] Test template generation and verify Docker setup works