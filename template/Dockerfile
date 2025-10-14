# Multi-stage Dockerfile with uv for {{ project_name }}

# Stage 1: Builder - Install dependencies
FROM python:{{ python_version }}-slim AS builder

WORKDIR /app

# Install uv
RUN pip install --no-cache-dir uv

# Copy dependency files
COPY pyproject.toml ./

# Install dependencies (without dev dependencies)
RUN uv sync --no-dev

# Stage 2: Runtime - Create production image
FROM python:{{ python_version }}-slim

WORKDIR /app

# Install curl for healthchecks
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# Copy virtual environment from builder
COPY --from=builder /app/.venv /app/.venv

# Copy application code
COPY backend/ ./backend/

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

USER appuser

# Add virtual environment to PATH
ENV PATH="/app/.venv/bin:$PATH"

# Expose port
EXPOSE 80

# Run application
CMD ["uvicorn", "backend.main:app", "--host", "0.0.0.0", "--port", "80"]

