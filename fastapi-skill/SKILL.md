---
name: fastapi
description: Use this skill whenever the user wants to build, scaffold, extend, or review a FastAPI application or API. Trigger on any request involving FastAPI endpoints, routers, middleware, dependency injection, Pydantic models, async routes, authentication (OAuth2/JWT), background tasks, WebSockets, testing with pytest, Docker deployment, or OpenAPI docs. Also trigger when the user wants to add features to an existing FastAPI project, debug FastAPI issues, set up database integration (SQLAlchemy, async SQLAlchemy, Tortoise ORM), or asks questions like "how do I structure my FastAPI app", "create a REST API with FastAPI", "add auth to my FastAPI app", or "make a CRUD API". Always use this skill before writing any FastAPI code — it contains production-grade patterns that significantly improve output quality.
---

# FastAPI Skill

Production-grade FastAPI development with best practices. For full code examples of specific topics, read the relevant reference file — don't guess at patterns.

## Reference Files

| File | When to read |
|------|-------------|
| `references/project-structure.md` | Scaffolding a new project or restructuring |
| `references/routing-and-endpoints.md` | Creating routes, path params, query params, request bodies |
| `references/auth.md` | OAuth2, JWT, API keys, permissions |
| `references/database.md` | SQLAlchemy (sync/async), migrations with Alembic |
| `references/testing.md` | pytest, TestClient, async tests, fixtures |
| `references/deployment.md` | Docker, Docker Compose, environment config, production tips |

---

## Core Principles

1. **Always use Pydantic v2** for request/response models (`model_config`, not `class Config`)
2. **Separate concerns**: routers → services → repositories → models
3. **Use dependency injection** (`Depends`) for DB sessions, auth, config
4. **Async by default** for I/O-bound routes; sync only when CPU-bound
5. **Never put business logic in route handlers** — delegate to service layer
6. **Always version your API** (`/api/v1/...`)
7. **Return typed responses** — always declare `response_model`

---

## Quick Start: Minimal App

```python
# main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.api.v1.router import api_router
from app.core.config import settings


@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup
    yield
    # shutdown


app = FastAPI(
    title=settings.PROJECT_NAME,
    version=settings.VERSION,
    lifespan=lifespan,
)

app.include_router(api_router, prefix="/api/v1")
```

---

## Essential Patterns (Quick Reference)

### Pydantic v2 Models
```python
from pydantic import BaseModel, ConfigDict, EmailStr, field_validator

class UserCreate(BaseModel):
    name: str
    email: EmailStr
    age: int

    @field_validator("age")
    @classmethod
    def age_must_be_positive(cls, v: int) -> int:
        if v < 0:
            raise ValueError("Age must be positive")
        return v

class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)  # replaces orm_mode=True
    id: int
    name: str
    email: str
```

### Dependency Injection
```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.db.session import get_async_session

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_async_session),
) -> User:
    ...

@router.get("/me", response_model=UserRead)
async def get_me(current_user: User = Depends(get_current_user)):
    return current_user
```

### Error Handling
```python
from fastapi import HTTPException, status

# Use status codes from fastapi.status — never raw numbers
raise HTTPException(
    status_code=status.HTTP_404_NOT_FOUND,
    detail="User not found",
)

# Custom exception handler
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(SomeCustomError)
async def custom_error_handler(request: Request, exc: SomeCustomError):
    return JSONResponse(status_code=400, content={"detail": str(exc)})
```

### Settings with pydantic-settings
```python
# app/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", case_sensitive=False)

    PROJECT_NAME: str = "My API"
    VERSION: str = "0.1.0"
    DATABASE_URL: str
    SECRET_KEY: str
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30

settings = Settings()
```

### Background Tasks
```python
from fastapi import BackgroundTasks

def send_email(to: str, subject: str, body: str):
    ...  # blocking I/O is fine here — runs in threadpool

@router.post("/users/", response_model=UserRead, status_code=status.HTTP_201_CREATED)
async def create_user(
    payload: UserCreate,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_async_session),
):
    user = await user_service.create(db, payload)
    background_tasks.add_task(send_email, user.email, "Welcome!", "Hello!")
    return user
```

---

## Common Mistakes to Avoid

| ❌ Don't | ✅ Do |
|----------|-------|
| Business logic in route handlers | Move to service layer |
| `orm_mode = True` in `class Config` | Use `model_config = ConfigDict(from_attributes=True)` |
| Sync DB calls in async routes | Use async SQLAlchemy or run_in_executor |
| Hardcode secrets | Use pydantic-settings + `.env` |
| Return raw dicts | Declare `response_model=` on every route |
| One giant `main.py` | Split into routers, services, models |
| `from app import *` | Explicit imports always |
| Swallow exceptions silently | Use exception handlers + logging |

---

## Package Recommendations

```toml
# pyproject.toml (using uv or poetry)
[tool.poetry.dependencies]
python = "^3.11"
fastapi = {extras = ["standard"], version = "^0.115"}  # includes uvicorn, python-multipart
pydantic = {extras = ["email"], version = "^2.7"}
pydantic-settings = "^2.3"
sqlalchemy = {extras = ["asyncio"], version = "^2.0"}
asyncpg = "^0.29"        # async PostgreSQL driver
alembic = "^1.13"
python-jose = {extras = ["cryptography"], version = "^3.3"}
passlib = {extras = ["bcrypt"], version = "^1.7"}
httpx = "^0.27"          # for async TestClient

[tool.poetry.dev-dependencies]
pytest = "^8.2"
pytest-asyncio = "^0.23"
pytest-cov = "^5.0"
ruff = "^0.4"
mypy = "^1.10"
```

---

## Quick Commands

```bash
# Run dev server
uvicorn app.main:app --reload --port 8000

# Auto-format + lint
ruff check . --fix && ruff format .

# Type check
mypy app/

# Run tests with coverage
pytest --cov=app --cov-report=term-missing

# Create Alembic migration
alembic revision --autogenerate -m "add users table"
alembic upgrade head
```
