# Cursor AI Rules Documentation

This directory contains modular AI agent rules for your generated backend application. These rules help AI coding assistants (like Cursor, GitHub Copilot, etc.) understand your project's architecture, coding standards, and best practices.

## File Organization

### 📋 [`project.md`](./project.md)
**Project Architecture & Patterns**

- Layered architecture (Routers → Services → Clients)
- FastAPI application structure
- Error handling patterns
- Database strategy
- Dependency injection
- Common CRUD patterns

**When to reference**: Architecture decisions, application structure, FastAPI patterns

### 🐍 [`python.md`](./python.md)
**Python Code Style & Implementation**

- Type hints and type annotations
- Naming conventions
- Async programming patterns
- Pydantic v2 models
- MongoDB/Motor operations
- Testing patterns
- Performance optimization

**When to reference**: Code style, Python-specific patterns, async/await, type hints

### 🛡️ [`general.md`](./general.md)
**Security, Quality & Workflow**

- Security best practices
- Code quality checklist
- Git workflow
- Development workflow
- Testing strategy
- Monitoring & logging
- Common mistakes to avoid

**When to reference**: Security concerns, workflow questions, best practices

## Primary Entry Point

### 📖 [`AGENTS.md`](../AGENTS.md) (Root Level)
The main AI agent rules file that provides quick critical rules and references these modular files.

## How This Works

When you use AI coding assistants with this project:

1. **AGENTS.md** provides quick, critical rules that must always be followed
2. The assistant can reference specific rule files for detailed guidance:
   - Architecture questions → `project.md`
   - Code style questions → `python.md`
   - Security/workflow questions → `general.md`

## Maintenance

### Adding New Rules

When adding project-specific rules:
1. Determine the category (project/python/general)
2. Add to the appropriate `.md` file
3. Keep rules concise and actionable
4. Include code examples for clarity

### Updating Rules

- Keep rules up-to-date with project evolution
- Remove outdated patterns
- Update examples to match current codebase
- Ensure consistency across files

## Key Principles

All rules follow these principles:

1. **Actionable**: Clear what to do and what to avoid
2. **Example-driven**: Show correct and incorrect patterns
3. **Enforceable**: Can be checked by linters/type checkers where possible
4. **Critical-first**: Most important rules are prominently featured
5. **Context-aware**: Rules match project's tech stack and patterns

## For AI Assistants

When helping with this codebase:
1. Always follow the layered architecture (Routers → Services → Clients)
2. All functions must have type hints
3. Use async/await for all I/O operations
4. Never log secrets or connection strings
5. Validate MongoDB ObjectIds before use
6. Use Pydantic models for all input validation
7. Follow guard clause pattern for error handling

## Quick Reference

### Architecture
```
Routers (HTTP) → Services (Business Logic) → Clients (Data/External APIs)
```

### Code Quality Commands
```bash
uv run ruff format backend      # Format
uv run ruff check backend --fix # Lint
uv run mypy backend             # Type check
uv run pytest --cov=backend     # Test
```

### Critical Rules
- ✅ Type hints on ALL functions
- ✅ Async for I/O operations
- ✅ Guard clauses for validation
- ✅ Pydantic for input validation
- ❌ Never log secrets
- ❌ No business logic in routers
- ❌ No FastAPI imports in services

## Learn More

- See [`AGENTS.md`](../AGENTS.md) for quick start rules
- See [`README.md`](../README.md) for project documentation
- See [`CONTRIBUTING.md`](../CONTRIBUTING.md) for development workflow

---

**These rules ensure consistent, high-quality code across the entire project.**

