# Contributing to Alkeme Backend Template

Thank you for your interest in improving the Alkeme Backend Template! This guide will help you develop and test template changes.

## Development Setup

### Prerequisites

- Python 3.10+
- [Copier](https://copier.readthedocs.io/) - `pip install copier` or `uvx copier`
- [uv](https://github.com/astral-sh/uv) - Fast Python package manager
- Docker Desktop (for testing generated projects)
- Git

### Clone the Repository

```bash
git clone https://github.com/alkeme/alkeme-template-backend.git
cd alkeme-template-backend
```

## Template Structure

```
alkeme-template-backend/
├── copier.yml              # Template configuration & questions
├── template/               # Template files (Jinja2 syntax)
│   ├── backend/           # Application code templates
│   │   ├── *.py           # Regular Python files (copied as-is)
│   │   └── *.jinja        # Jinja templates (suffix removed on generation)
│   ├── tests/             # Test templates
│   ├── pyproject.toml.jinja
│   ├── Dockerfile
│   └── docker-compose.yml.jinja
├── README.md              # Template documentation
├── CONTRIBUTING.md        # This file
└── .cursor/rules/         # AI agent rules

```

## Testing Your Changes

### 1. Generate a Test Project

```bash
# Basic test (no auth)
uvx copier copy . /tmp/test-backend \
  --data project_name="Test Backend" \
  --data package_name="test_backend" \
  --data use_azure_auth=false \
  --trust --defaults

# With Azure auth
uvx copier copy . /tmp/test-backend-auth \
  --data use_azure_auth=true \
  --trust --defaults
```

### 2. Test the Generated Project

```bash
cd /tmp/test-backend

# Install dependencies
uv sync

# Run tests
uv run pytest

# Start server
uv run uvicorn backend.main:app --reload

# Test with Docker
docker compose up --build
```

### 3. Verify Key Functionality

- [ ] FastAPI server starts without errors
- [ ] API docs accessible at http://localhost:8000/docs
- [ ] Health check endpoint returns 200
- [ ] MongoDB connection works (via Docker Compose)
- [ ] Code quality checks pass:
  ```bash
  uv run ruff format backend
  uv run ruff check backend
  uv run mypy backend
  ```

## Template Syntax

### Jinja Variables

```jinja
# In copier.yml
package_name:
  type: str
  default: "my_backend"

# In templates (*.jinja files)
name = "{{ package_name }}"
```

### Conditional Content

```jinja
{% if use_azure_auth -%}
from backend.auth import azure_scheme
{% endif -%}
```

### Important Notes

- Files ending in `.jinja` will have the suffix removed during generation
- Use `{%- ... -%}` to trim whitespace
- Variables from `copier.yml` are available in all templates
- Use `{{ _copier_conf.dst_path }}` for the destination path

## Common Tasks

### Add a New Configuration Option

1. **Add to `copier.yml`:**
   ```yaml
   use_new_feature:
     type: bool
     help: Enable the new feature?
     default: false
   ```

2. **Update templates** with conditional logic:
   ```jinja
   {% if use_new_feature -%}
   # New feature code here
   {% endif -%}
   ```

3. **Update `.copier-answers.yml.jinja`** to track the answer:
   ```yaml
   use_new_feature: {{ use_new_feature }}
   ```

4. **Test both configurations** (feature enabled and disabled)

### Add a New Template File

1. Create file in `template/` directory
2. Use `.jinja` suffix if it needs variable substitution
3. Add Jinja variables: `{{ variable_name }}`
4. Test generation to verify

### Modify Existing Templates

1. Edit the `.jinja` file in `template/`
2. Test with `copier copy . /tmp/test-backend --trust`
3. Verify the generated file is correct
4. Run code quality checks on generated code

## Code Quality Standards

Generated projects must follow these standards:

### Python Code

- **Type hints**: Required on all functions
- **Docstrings**: Public functions and classes
- **Formatting**: Ruff (line length 100)
- **Linting**: Ruff with default rules
- **Type checking**: Mypy strict mode

### Architecture

```
Routers (HTTP) → Services (Business Logic) → Clients (Data/External APIs)
```

- **Routers**: FastAPI endpoints, validation, HTTP concerns
- **Services**: Pure business logic, no FastAPI imports
- **Clients**: Database, external APIs, I/O operations

### Testing

- Use pytest with pytest-asyncio
- Test coverage for core functionality
- Mock external dependencies (Azure AD, MongoDB)

## Submitting Changes

### Pull Request Checklist

- [ ] Tested template generation (both with and without optional features)
- [ ] Generated projects pass code quality checks
- [ ] FastAPI server starts successfully
- [ ] Docker Compose stack works
- [ ] Updated documentation if needed
- [ ] No sensitive information in templates or code

### PR Description Template

```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Documentation update
- [ ] Template improvement

## Testing
- [ ] Generated test project
- [ ] Ran FastAPI server
- [ ] Tested Docker Compose
- [ ] Verified both auth/no-auth configurations

## Screenshots (if applicable)
```

## Getting Help

- **Issues**: https://github.com/alkeme/alkeme-template-backend/issues
- **Discussions**: https://github.com/alkeme/alkeme-template-backend/discussions
- **Email**: dev@alkeme.com

## Resources

- [Copier Documentation](https://copier.readthedocs.io/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [Motor Documentation](https://motor.readthedocs.io/)
- [uv Documentation](https://github.com/astral-sh/uv)

## License

By contributing, you agree that your contributions will be licensed under the same license as the project.

---

Thank you for contributing! 🚀

