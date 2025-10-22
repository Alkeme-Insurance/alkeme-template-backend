"""Pytest configuration and fixtures."""

from __future__ import annotations

from typing import AsyncGenerator

import pytest
from httpx import AsyncClient
from motor.motor_asyncio import AsyncIOMotorClient

from backend.main import app


@pytest.fixture
async def test_client() -> AsyncGenerator[AsyncClient, None]:
    """Create test client."""
    async with AsyncClient(app=app, base_url="http://testserver") as client:
        yield client


@pytest.fixture
async def test_db() -> AsyncGenerator[AsyncIOMotorClient, None]:
    """Create test database."""
    client = AsyncIOMotorClient("mongodb://localhost:27017")
    db = client["test_{{ package_name }}"]

    yield db

    # Cleanup
    await client.drop_database("test_{{ package_name }}")
    client.close()

