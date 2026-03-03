# FastAPI Routing & Endpoints

## Router Setup

```python
# app/api/v1/endpoints/users.py
from fastapi import APIRouter, Depends, status
from sqlalchemy.ext.asyncio import AsyncSession
from app.db.session import get_async_session
from app.schemas.user import UserCreate, UserRead, UserUpdate
from app.services import user_service
from app.api.deps import get_current_user
from app.db.models.user import User

router = APIRouter()
```

---

## CRUD Endpoints (Full Example)

```python
@router.get("/", response_model=list[UserRead])
async def list_users(
    skip: int = 0,
    limit: int = 100,
    db: AsyncSession = Depends(get_async_session),
    _: User = Depends(get_current_user),  # require auth, don't need the value
):
    return await user_service.get_all(db, skip=skip, limit=limit)


@router.post("/", response_model=UserRead, status_code=status.HTTP_201_CREATED)
async def create_user(
    payload: UserCreate,
    db: AsyncSession = Depends(get_async_session),
):
    return await user_service.create(db, payload)


@router.get("/{user_id}", response_model=UserRead)
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_async_session),
):
    user = await user_service.get_by_id(db, user_id)
    if not user:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="User not found")
    return user


@router.patch("/{user_id}", response_model=UserRead)
async def update_user(
    user_id: int,
    payload: UserUpdate,
    db: AsyncSession = Depends(get_async_session),
    current_user: User = Depends(get_current_user),
):
    return await user_service.update(db, user_id, payload, current_user)


@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(
    user_id: int,
    db: AsyncSession = Depends(get_async_session),
    current_user: User = Depends(get_current_user),
):
    await user_service.delete(db, user_id, current_user)
```

---

## Pydantic Schemas

```python
# app/schemas/user.py
from pydantic import BaseModel, ConfigDict, EmailStr
from datetime import datetime


class UserBase(BaseModel):
    name: str
    email: EmailStr


class UserCreate(UserBase):
    password: str


class UserUpdate(BaseModel):
    name: str | None = None
    email: EmailStr | None = None


class UserRead(UserBase):
    model_config = ConfigDict(from_attributes=True)
    id: int
    created_at: datetime
    is_active: bool
```

---

## Path, Query, and Body Parameters

```python
from fastapi import Path, Query, Body
from typing import Annotated

@router.get("/{item_id}")
async def get_item(
    # Path param with validation
    item_id: Annotated[int, Path(gt=0, description="The item ID")],
    # Optional query param
    include_deleted: Annotated[bool, Query()] = False,
):
    ...

@router.post("/search")
async def search(
    # Body param with example
    q: Annotated[str, Body(min_length=1, max_length=200, examples=["laptop"])],
):
    ...
```

---

## File Upload

```python
from fastapi import UploadFile, File
import shutil

@router.post("/upload")
async def upload_file(file: UploadFile = File(...)):
    if file.content_type not in ["image/jpeg", "image/png"]:
        raise HTTPException(status_code=400, detail="Only JPEG/PNG allowed")

    dest = f"/tmp/{file.filename}"
    with open(dest, "wb") as buf:
        shutil.copyfileobj(file.file, buf)

    return {"filename": file.filename, "size": file.size}
```

---

## Pagination Pattern

```python
from pydantic import BaseModel
from typing import Generic, TypeVar

T = TypeVar("T")

class Page(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    size: int
    pages: int

@router.get("/", response_model=Page[UserRead])
async def list_users(
    page: int = 1,
    size: int = 20,
    db: AsyncSession = Depends(get_async_session),
):
    skip = (page - 1) * size
    items, total = await user_service.paginate(db, skip=skip, limit=size)
    return Page(
        items=items,
        total=total,
        page=page,
        size=size,
        pages=(total + size - 1) // size,
    )
```

---

## WebSocket Endpoint

```python
from fastapi import WebSocket, WebSocketDisconnect

class ConnectionManager:
    def __init__(self):
        self.active_connections: list[WebSocket] = []

    async def connect(self, ws: WebSocket):
        await ws.accept()
        self.active_connections.append(ws)

    def disconnect(self, ws: WebSocket):
        self.active_connections.remove(ws)

    async def broadcast(self, message: str):
        for connection in self.active_connections:
            await connection.send_text(message)

manager = ConnectionManager()

@router.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: str):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast(f"Client {client_id}: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

---

## Custom Response Types

```python
from fastapi.responses import StreamingResponse, FileResponse
import io

@router.get("/export/csv")
async def export_csv():
    def generate():
        yield "id,name,email\n"
        for user in get_users():
            yield f"{user.id},{user.name},{user.email}\n"

    return StreamingResponse(
        generate(),
        media_type="text/csv",
        headers={"Content-Disposition": "attachment; filename=users.csv"},
    )
```

---

## Dependency Hierarchy (app/api/deps.py)

```python
# app/api/deps.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession
from app.db.session import get_async_session
from app.core import security
from app.services import user_service
from app.db.models.user import User

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/token")


async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_async_session),
) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    payload = security.decode_token(token)
    if payload is None:
        raise credentials_exception
    user = await user_service.get_by_id(db, int(payload.get("sub")))
    if user is None:
        raise credentials_exception
    return user


async def get_current_active_user(
    current_user: User = Depends(get_current_user),
) -> User:
    if not current_user.is_active:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user


async def get_current_superuser(
    current_user: User = Depends(get_current_user),
) -> User:
    if not current_user.is_superuser:
        raise HTTPException(status_code=403, detail="Not enough permissions")
    return current_user
```
