# FastAPI Testing

## Setup

```bash
pip install pytest pytest-asyncio httpx --break-system-packages
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

---

## conftest.py

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from app.main import app
from app.db.base import Base
from app.db.session import get_async_session
from app.core.security import create_access_token

TEST_DATABASE_URL = "sqlite+aiosqlite:///./test.db"

test_engine = create_async_engine(TEST_DATABASE_URL, echo=False)
TestSessionLocal = async_sessionmaker(test_engine, class_=AsyncSession, expire_on_commit=False)


@pytest.fixture(scope="session", autouse=True)
async def create_tables():
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


@pytest.fixture
async def db():
    async with TestSessionLocal() as session:
        yield session
        await session.rollback()  # rollback after each test


@pytest.fixture
async def client(db):
    async def override_get_session():
        yield db

    app.dependency_overrides[get_async_session] = override_get_session
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()


@pytest.fixture
def auth_headers():
    token = create_access_token(subject=1)
    return {"Authorization": f"Bearer {token}"}
```

---

## Writing Tests

```python
# tests/test_users.py
import pytest
from httpx import AsyncClient


async def test_create_user(client: AsyncClient):
    response = await client.post("/api/v1/users/", json={
        "name": "Alice",
        "email": "alice@example.com",
        "password": "secret123",
    })
    assert response.status_code == 201
    data = response.json()
    assert data["email"] == "alice@example.com"
    assert "id" in data
    assert "password" not in data  # never leak password


async def test_create_duplicate_user(client: AsyncClient):
    payload = {"name": "Bob", "email": "bob@example.com", "password": "secret"}
    await client.post("/api/v1/users/", json=payload)
    response = await client.post("/api/v1/users/", json=payload)
    assert response.status_code == 400


async def test_get_user_not_found(client: AsyncClient, auth_headers):
    response = await client.get("/api/v1/users/9999", headers=auth_headers)
    assert response.status_code == 404


async def test_list_users_requires_auth(client: AsyncClient):
    response = await client.get("/api/v1/users/")
    assert response.status_code == 401


async def test_login(client: AsyncClient):
    # First create user
    await client.post("/api/v1/users/", json={
        "name": "Carol",
        "email": "carol@example.com",
        "password": "mypassword",
    })
    # Then login
    response = await client.post(
        "/api/v1/auth/token",
        data={"username": "carol@example.com", "password": "mypassword"},
    )
    assert response.status_code == 200
    data = response.json()
    assert "access_token" in data
    assert data["token_type"] == "bearer"
```

---

## Testing Services Directly

```python
# tests/test_user_service.py
import pytest
from app.services import user_service
from app.schemas.user import UserCreate


async def test_create_user_service(db):
    payload = UserCreate(name="Dave", email="dave@example.com", password="pw123")
    user = await user_service.create(db, payload)
    assert user.id is not None
    assert user.email == "dave@example.com"
    assert not hasattr(user, "password")  # ORM model shouldn't expose plain password


async def test_create_duplicate_raises(db):
    payload = UserCreate(name="Eve", email="eve@example.com", password="pw")
    await user_service.create(db, payload)
    with pytest.raises(Exception):
        await user_service.create(db, payload)
```

---

## Coverage

```bash
pytest --cov=app --cov-report=term-missing --cov-report=html
# opens htmlcov/index.html for detailed report
```
