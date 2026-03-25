---
name: fastapi-patterns
description: FastAPI patterns for building production-grade APIs — routing, Pydantic models, dependency injection, middleware, async patterns, WebSockets, and background tasks.
origin: custom
---

# FastAPI Development Patterns

Production-grade FastAPI architecture patterns for high-performance async APIs.

## When to Activate

- Building FastAPI web applications
- Designing REST or GraphQL APIs with FastAPI
- Working with async Python and ASGI
- Setting up dependency injection
- Implementing WebSockets or background tasks

## Project Structure

```
myproject/
├── src/
│   └── app/
│       ├── __init__.py
│       ├── main.py              # FastAPI app factory
│       ├── config.py            # Settings (Pydantic BaseSettings)
│       ├── dependencies.py      # Shared dependencies
│       ├── middleware.py         # Custom middleware
│       ├── api/
│       │   ├── __init__.py
│       │   ├── v1/
│       │   │   ├── __init__.py
│       │   │   ├── router.py    # APIRouter aggregator
│       │   │   ├── users.py
│       │   │   ├── products.py
│       │   │   └── auth.py
│       │   └── deps.py          # API-specific dependencies
│       ├── models/              # SQLAlchemy models
│       │   ├── __init__.py
│       │   ├── user.py
│       │   └── product.py
│       ├── schemas/             # Pydantic schemas
│       │   ├── __init__.py
│       │   ├── user.py
│       │   └── product.py
│       ├── services/            # Business logic
│       │   ├── __init__.py
│       │   ├── user_service.py
│       │   └── product_service.py
│       ├── repositories/        # Data access
│       │   ├── __init__.py
│       │   └── user_repo.py
│       └── core/
│           ├── __init__.py
│           ├── security.py      # JWT, hashing
│           ├── exceptions.py    # Custom exceptions
│           └── database.py      # DB session management
├── tests/
│   ├── conftest.py
│   ├── test_users.py
│   └── test_products.py
├── alembic/
│   └── versions/
├── alembic.ini
├── pyproject.toml
├── Dockerfile
└── docker-compose.yml
```

## App Factory

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager
from app.api.v1.router import api_router
from app.config import settings
from app.core.database import engine

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    # await init_db()
    yield
    # Shutdown
    await engine.dispose()

def create_app() -> FastAPI:
    app = FastAPI(
        title=settings.PROJECT_NAME,
        version=settings.VERSION,
        docs_url="/docs" if settings.DEBUG else None,
        lifespan=lifespan,
    )

    app.include_router(api_router, prefix="/api/v1")

    return app

app = create_app()
```

## Configuration with Pydantic Settings

```python
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    PROJECT_NAME: str = "MyAPI"
    VERSION: str = "1.0.0"
    DEBUG: bool = False

    DATABASE_URL: str
    REDIS_URL: str = "redis://localhost:6379/0"

    SECRET_KEY: str
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    ALGORITHM: str = "HS256"

    CORS_ORIGINS: list[str] = ["http://localhost:3000"]

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

@lru_cache
def get_settings() -> Settings:
    return Settings()

settings = get_settings()
```

## Dependency Injection

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.database import get_db_session

# Database session dependency
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with get_db_session() as session:
        yield session

# Current user dependency
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    payload = jwt.decode(token, settings.SECRET_KEY, algorithms=[settings.ALGORITHM])
    user_id = payload.get("sub")
    if not user_id:
        raise HTTPException(status_code=401, detail="Invalid token")
    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=401, detail="User not found")
    return user

# Service dependency
async def get_user_service(db: AsyncSession = Depends(get_db)) -> UserService:
    return UserService(db)
```

## Request/Response Schemas

```python
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime

class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8, max_length=128)
    name: str = Field(min_length=1, max_length=255)

class UserResponse(BaseModel):
    id: int
    email: str
    name: str
    created_at: datetime

    model_config = {"from_attributes": True}

class UserUpdate(BaseModel):
    name: str | None = None
    email: EmailStr | None = None

class PaginatedResponse(BaseModel):
    items: list[UserResponse]
    total: int
    page: int
    per_page: int
    pages: int
```

## Router Patterns

```python
from fastapi import APIRouter, Depends, HTTPException, Query, status
from app.schemas.user import UserCreate, UserResponse, UserUpdate, PaginatedResponse
from app.services.user_service import UserService
from app.api.deps import get_current_user, get_user_service

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/", response_model=PaginatedResponse)
async def list_users(
    page: int = Query(1, ge=1),
    per_page: int = Query(20, ge=1, le=100),
    service: UserService = Depends(get_user_service),
):
    return await service.list_users(page=page, per_page=per_page)

@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    data: UserCreate,
    service: UserService = Depends(get_user_service),
):
    return await service.create_user(data)

@router.get("/me", response_model=UserResponse)
async def get_me(current_user: User = Depends(get_current_user)):
    return current_user

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service),
):
    user = await service.get_user(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@router.patch("/{user_id}", response_model=UserResponse)
async def update_user(
    user_id: int,
    data: UserUpdate,
    current_user: User = Depends(get_current_user),
    service: UserService = Depends(get_user_service),
):
    if current_user.id != user_id:
        raise HTTPException(status_code=403, detail="Not authorized")
    return await service.update_user(user_id, data)

@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(
    user_id: int,
    current_user: User = Depends(get_current_user),
    service: UserService = Depends(get_user_service),
):
    await service.delete_user(user_id)
```

## Error Handling

```python
from fastapi import Request
from fastapi.responses import JSONResponse

class AppException(Exception):
    def __init__(self, status_code: int, detail: str, error_code: str | None = None):
        self.status_code = status_code
        self.detail = detail
        self.error_code = error_code

class NotFoundError(AppException):
    def __init__(self, resource: str, id: str | int):
        super().__init__(404, f"{resource} with id {id} not found", "NOT_FOUND")

class ConflictError(AppException):
    def __init__(self, detail: str):
        super().__init__(409, detail, "CONFLICT")

# Register exception handler
@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": exc.error_code or "ERROR",
            "detail": exc.detail,
        },
    )
```

## Middleware

```python
import time
import logging
from starlette.middleware.base import BaseHTTPMiddleware

logger = logging.getLogger(__name__)

class RequestLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        start = time.perf_counter()
        response = await call_next(request)
        duration = time.perf_counter() - start
        logger.info(
            f"{request.method} {request.url.path} "
            f"status={response.status_code} duration={duration:.3f}s"
        )
        return response

# CORS setup
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## Background Tasks

```python
from fastapi import BackgroundTasks

@router.post("/users/", response_model=UserResponse, status_code=201)
async def create_user(
    data: UserCreate,
    background_tasks: BackgroundTasks,
    service: UserService = Depends(get_user_service),
):
    user = await service.create_user(data)
    background_tasks.add_task(send_welcome_email, user.email, user.name)
    return user

async def send_welcome_email(email: str, name: str):
    # Send email asynchronously
    ...
```

## WebSocket

```python
from fastapi import WebSocket, WebSocketDisconnect

class ConnectionManager:
    def __init__(self):
        self.active: list[WebSocket] = []

    async def connect(self, ws: WebSocket):
        await ws.accept()
        self.active.append(ws)

    def disconnect(self, ws: WebSocket):
        self.active.remove(ws)

    async def broadcast(self, message: str):
        for ws in self.active:
            await ws.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws")
async def websocket_endpoint(ws: WebSocket):
    await manager.connect(ws)
    try:
        while True:
            data = await ws.receive_text()
            await manager.broadcast(f"Message: {data}")
    except WebSocketDisconnect:
        manager.disconnect(ws)
```

## Testing

```python
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app

@pytest.fixture
async def client():
    async with AsyncClient(
        transport=ASGITransport(app=app), base_url="http://test"
    ) as ac:
        yield ac

@pytest.mark.asyncio
async def test_create_user(client: AsyncClient):
    response = await client.post("/api/v1/users/", json={
        "email": "test@example.com",
        "password": "securepass123",
        "name": "Test User",
    })
    assert response.status_code == 201
    assert response.json()["email"] == "test@example.com"

@pytest.mark.asyncio
async def test_list_users(client: AsyncClient):
    response = await client.get("/api/v1/users/")
    assert response.status_code == 200
    assert "items" in response.json()
```

## Security

```python
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext
from datetime import datetime, timedelta

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=30))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, settings.SECRET_KEY, algorithm=settings.ALGORITHM)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def hash_password(password: str) -> str:
    return pwd_context.hash(password)
```

## Quick Reference

| Pattern | Description |
|---------|-------------|
| App factory | `create_app()` with lifespan |
| Pydantic Settings | Config from env vars |
| Depends() | Dependency injection |
| APIRouter | Modular route organization |
| BackgroundTasks | Fire-and-forget tasks |
| WebSocket | Real-time communication |
| httpx + AsyncClient | Async test client |
| OAuth2 + JWT | Token-based auth |

**Remember**: FastAPI is async-first. Use `async def` for I/O-bound endpoints. Avoid blocking calls in async routes -- use `run_in_executor` or background tasks for CPU-bound work.
