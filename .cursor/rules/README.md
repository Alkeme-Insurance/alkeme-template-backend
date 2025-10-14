# Cursor Rules Organization

This directory contains modular rules for the Alkeme Backend Template project. Rules are organized by concern for easier maintenance and reference.

> **Note**: These rules are referenced by `AGENTS.md` in the project root, which is the new standard format for AI agent guidance (replacing legacy `.cursorrules`).

## File Structure

### 📋 project.md (333 lines)
**Project-Specific Rules & Architecture**

Covers:
- Copier template context and Jinja2 syntax
- Layered architecture pattern (Routers → Services → Clients)
- FastAPI application structure and patterns
- File organization and naming conventions
- Configuration patterns (Pydantic Settings)
- Database strategy (MongoDB local → Cosmos DB production)
- Docker & Azure deployment
- Error handling patterns

**Use when:** Working on project structure, architecture, FastAPI patterns, or deployment.

### 🐍 python.md (498 lines)
**Python Code Style & Patterns**

Covers:
- Python version requirements (3.10+)
- Code style (PEP 8, line length, imports)
- Type hints (required on all functions)
- Naming conventions (detailed user preferences)
- Async programming patterns
- Pydantic v2 models and validation
- MongoDB/Motor patterns
- Functions & methods best practices
- Error handling
- Documentation requirements
- Testing patterns
- Performance optimization

**Use when:** Writing Python code, defining models, database operations, or async functions.

### 🛡️ general.md (601 lines)
**Security, Quality & Development Workflow**

Covers:
- Security (secrets management, input validation, CORS, auth, rate limiting)
- Code quality standards (pre-commit checklist, coverage, code review)
- Git & version control (commit messages, what not to commit)
- Development workflow (local setup, Docker, Azure deployment)
- Testing standards (organization, naming, AAA pattern, mocking)
- Dependencies management (uv commands, version constraints)
- Documentation standards
- Logging (structured logging, what to log/not log)
- Monitoring & observability
- Common mistakes to avoid

**Use when:** Setting up development environment, implementing security, writing tests, or establishing workflows.

## How These Rules Work Together

1. **`AGENTS.md`** (root) - Primary AI agent rules file (new standard)
2. **`project.md`** - Architecture and project-specific patterns
3. **`python.md`** - Python code style and implementation details  
4. **`general.md`** - Cross-cutting concerns (security, testing, workflow)
5. **`.cursorrules`** (legacy) - Deprecated, will be removed

## Rules Hierarchy

```
AGENTS.md (Primary - new standard)
    ├─ Quick critical rules with examples
    └─ References detailed rules below
        
.cursor/rules/
    ├─ project.md (333 lines)    - What to build & how to structure
    ├─ python.md (498 lines)     - How to write Python code
    └─ general.md (601 lines)    - How to develop & deploy safely
```

## For AI Assistants

Primary entry point: **`AGENTS.md`** (root)

When helping with:
- **Architecture questions** → Reference `project.md`
- **Code implementation** → Reference `python.md`
- **Security/testing/workflow** → Reference `general.md`
- **Quick patterns** → Use `AGENTS.md`

## For Developers

Quick lookup:
```bash
# Architecture patterns
cat .cursor/rules/project.md | grep -A 10 "Architecture"

# Python naming conventions
cat .cursor/rules/python.md | grep -A 20 "Naming"

# Security rules
cat .cursor/rules/general.md | grep -A 15 "Security"

# Pre-commit checklist
cat .cursor/rules/general.md | grep -A 10 "Pre-Commit"
```

## Maintenance

When updating rules:
1. Keep `.cursorrules` concise (quick reference only)
2. Add detailed explanations to appropriate `rules/*.md` file
3. Update this README if adding new files
4. Ensure no duplication between files
5. Cross-reference related rules

## Total Documentation

- `AGENTS.md`: ~200 lines (primary entry point)
- `project.md`: 333 lines
- `python.md`: 498 lines
- `general.md`: 601 lines
- `README.md`: ~120 lines
- **Total**: ~1,752 lines of comprehensive guidance

---

Last updated: October 2025

