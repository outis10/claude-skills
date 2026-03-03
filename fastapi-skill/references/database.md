# FastAPI Database (Async SQLAlchemy + Alembic)

## Session Setup

```python
# app/db/session.py
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from app.core.config import settings

engine = create_async_engine(
    settings.DATABASE_URL,
    echo=settings.DEBUG,
    pool_pre_ping=True,
    pool_size=10,
    max_overflow=20,
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)


async def get_async_session() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

---

## Base + ORM Model

```python
# app/db/base.py
from datetime import datetime, timezone
from sqlalchemy import func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        server_default=func.now(), nullable=False
    )
    updated_at: Mapped[datetime] = mapped_column(
        server_default=func.now(), onupdate=func.now(), nullable=False
    )
```

```python
# app/db/models/user.py
from sqlalchemy import String, Boolean
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, TimestampMixin


class User(Base, TimestampMixin):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True, index=True)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True, nullable=False)
    hashed_password: Mapped[str] = mapped_column(String(255), nullable=False)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    is_superuser: Mapped[bool] = mapped_column(Boolean, default=False)

    # items: Mapped[list["Item"]] = relationship(back_populates="owner")
```

---

## Repository Pattern

```python
# app/repositories/user_repo.py
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from app.db.models.user import User
from app.schemas.user import UserCreate, UserUpdate
from app.core.security import hash_password


class UserRepository:
    async def get_by_id(self, db: AsyncSession, user_id: int) -> User | None:
        result = await db.execute(select(User).where(User.id == user_id))
        return result.scalar_one_or_none()

    async def get_by_email(self, db: AsyncSession, email: str) -> User | None:
        result = await db.execute(select(User).where(User.email == email))
        return result.scalar_one_or_none()

    async def get_all(self, db: AsyncSession, skip: int = 0, limit: int = 100) -> list[User]:
        result = await db.execute(select(User).offset(skip).limit(limit))
        return list(result.scalars().all())

    async def create(self, db: AsyncSession, payload: UserCreate) -> User:
        user = User(
            name=payload.name,
            email=payload.email,
            hashed_password=hash_password(payload.password),
        )
        db.add(user)
        await db.flush()  # flush to get id without commit
        await db.refresh(user)
        return user

    async def update(self, db: AsyncSession, user: User, payload: UserUpdate) -> User:
        for field, value in payload.model_dump(exclude_unset=True).items():
            setattr(user, field, value)
        await db.flush()
        await db.refresh(user)
        return user

    async def delete(self, db: AsyncSession, user: User) -> None:
        await db.delete(user)
        await db.flush()


user_repo = UserRepository()
```

---

## Service Layer

```python
# app/services/user_service.py
from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import HTTPException, status
from app.repositories.user_repo import user_repo
from app.schemas.user import UserCreate, UserUpdate
from app.db.models.user import User


async def get_by_id(db: AsyncSession, user_id: int) -> User:
    user = await user_repo.get_by_id(db, user_id)
    if not user:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="User not found")
    return user


async def get_by_email(db: AsyncSession, email: str) -> User | None:
    return await user_repo.get_by_email(db, email)


async def get_all(db: AsyncSession, skip: int = 0, limit: int = 100) -> list[User]:
    return await user_repo.get_all(db, skip=skip, limit=limit)


async def create(db: AsyncSession, payload: UserCreate) -> User:
    existing = await user_repo.get_by_email(db, payload.email)
    if existing:
        raise HTTPException(status_code=400, detail="Email already registered")
    return await user_repo.create(db, payload)


async def update(db: AsyncSession, user_id: int, payload: UserUpdate, actor: User) -> User:
    user = await get_by_id(db, user_id)
    if user.id != actor.id and not actor.is_superuser:
        raise HTTPException(status_code=403, detail="Not allowed")
    return await user_repo.update(db, user, payload)


async def delete(db: AsyncSession, user_id: int, actor: User) -> None:
    user = await get_by_id(db, user_id)
    if user.id != actor.id and not actor.is_superuser:
        raise HTTPException(status_code=403, detail="Not allowed")
    await user_repo.delete(db, user)
```

---

## Alembic Setup

```ini
# alembic.ini (key line)
sqlalchemy.url = driver://user:pass@localhost/db  # override in env.py
```

```python
# alembic/env.py
import asyncio
from alembic import context
from sqlalchemy.ext.asyncio import create_async_engine
from app.core.config import settings
from app.db.base import Base
# import all models so Alembic detects them
import app.db.models  # noqa: F401

target_metadata = Base.metadata


def run_migrations_offline():
    context.configure(url=settings.DATABASE_URL, target_metadata=target_metadata, literal_binds=True)
    with context.begin_transaction():
        context.run_migrations()


async def run_migrations_online():
    connectable = create_async_engine(settings.DATABASE_URL)
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def do_run_migrations(connection):
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    asyncio.run(run_migrations_online())
```

---

## Common SQLAlchemy Queries

```python
from sqlalchemy import select, and_, or_, func

# Filter
result = await db.execute(select(User).where(User.is_active == True))

# Multiple conditions
result = await db.execute(
    select(User).where(and_(User.is_active == True, User.is_superuser == False))
)

# Count
count_result = await db.execute(select(func.count()).select_from(User))
total = count_result.scalar()

# Order + paginate
result = await db.execute(
    select(User).order_by(User.created_at.desc()).offset(skip).limit(limit)
)

# Join
from app.db.models.item import Item
result = await db.execute(
    select(Item).join(User, Item.owner_id == User.id).where(User.id == user_id)
)
```
