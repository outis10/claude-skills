# FastAPI Project Structure

## Recommended Layout

```
my-api/
├── app/
│   ├── __init__.py
│   ├── main.py                  # FastAPI app creation, lifespan, middleware
│   ├── api/
│   │   ├── __init__.py
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── router.py        # Aggregate all v1 routers
│   │       └── endpoints/
│   │           ├── __init__.py
│   │           ├── users.py
│   │           └── items.py
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py            # Settings (pydantic-settings)
│   │   └── security.py         # Password hashing, JWT utils
│   ├── db/
│   │   ├── __init__.py
│   │   ├── base.py              # SQLAlchemy Base
│   │   ├── session.py           # Engine + get_async_session
│   │   └── models/              # ORM models
│   │       ├── __init__.py
│   │       └── user.py
│   ├── schemas/                 # Pydantic models (request/response)
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── token.py
│   ├── services/                # Business logic
│   │   ├── __init__.py
│   │   └── user_service.py
│   └── repositories/            # DB queries (optional, for complex apps)
│       ├── __init__.py
│       └── user_repo.py
├── tests/
│   ├── conftest.py
│   ├── test_users.py
│   └── test_items.py
├── alembic/
│   ├── env.py
│   └── versions/
├── alembic.ini
├── pyproject.toml
├── .env
├── .env.example
├── Dockerfile
└── docker-compose.yml
```

---

## v1 Router Aggregator

```python
# app/api/v1/router.py
from fastapi import APIRouter
from app.api.v1.endpoints import users, items, auth

api_router = APIRouter()
api_router.include_router(auth.router, prefix="/auth", tags=["auth"])
api_router.include_router(users.router, prefix="/users", tags=["users"])
api_router.include_router(items.router, prefix="/items", tags=["items"])
```

---

## main.py Full Template

```python
from contextlib import asynccontextmanager
import logging
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.api.v1.router import api_router
from app.core.config import settings
from app.db.session import engine
from app.db.base import Base

logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    logger.info("Starting up...")
    # Optional: create tables (use Alembic in production instead)
    # async with engine.begin() as conn:
    #     await conn.run_sync(Base.metadata.create_all)
    yield
    logger.info("Shutting down...")
    await engine.dispose()


app = FastAPI(
    title=settings.PROJECT_NAME,
    version=settings.VERSION,
    description=settings.DESCRIPTION,
    openapi_url=f"{settings.API_PREFIX}/openapi.json",
    lifespan=lifespan,
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(api_router, prefix=settings.API_PREFIX)


@app.get("/health", tags=["health"])
async def health_check():
    return {"status": "ok"}
```

---

## config.py Full Template

```python
from pydantic import AnyHttpUrl
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )

    PROJECT_NAME: str = "FastAPI App"
    VERSION: str = "0.1.0"
    DESCRIPTION: str = ""
    API_PREFIX: str = "/api/v1"

    # Database
    DATABASE_URL: str  # e.g. postgresql+asyncpg://user:pass@localhost/db

    # Auth
    SECRET_KEY: str
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30

    # CORS
    CORS_ORIGINS: list[AnyHttpUrl] = []

    # Debug
    DEBUG: bool = False


settings = Settings()
```
