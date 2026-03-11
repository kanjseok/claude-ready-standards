# 14. Backend Skeleton (FastAPI)

> FastAPI backend skeleton guide.
> Covers dependencies, directory structure, entry point, common utilities, and Swagger patterns.

---

## 1. Dependencies

### `pyproject.toml` (managed by `uv`)

```toml
[project]
name = "backend-py"
version = "0.1.0"
description = "FastAPI backend application"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.6",
    "pydantic>=2.9.0",
    "pydantic-settings>=2.5.0",
    "sqlalchemy[asyncio]>=2.0.35",
    "asyncpg>=0.29.0",
    "motor>=3.6.0",
    "alembic>=1.13.2",
    "pyjwt[crypto]>=2.9.0",
    "google-auth>=2.34.0",
    "redis>=5.0.8",
    "python-multipart>=0.0.9",
    "passlib[bcrypt]>=1.7.4",
]

[dependency-groups]
dev = [
    "pytest>=8.3.3",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=5.0.0",
    "httpx>=0.27.2",
    "mypy>=1.11.2",
    "ruff>=0.6.7",
]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.mypy]
python_version = "3.12"
strict = true
```

---

## 2. Directory Structure

```
apps/backend-py/
├── alembic/              # PostgreSQL migrations (SQLAlchemy)
├── app/
│   ├── common/           # Common utilities, decorators, exceptions
│   │   ├── exceptions/   # Custom exceptions (ApiError, etc.)
│   │   ├── schemas/      # Common Pydantic schemas (Pagination, Response)
│   │   └── utils/        # Utility functions (Encryption, etc.)
│   ├── core/             # Core settings and DI
│   │   ├── config.py     # Environment variables (pydantic-settings)
│   │   ├── deps.py       # Dependency Injection (DB, Auth)
│   │   └── security.py   # JWT, password hashing
│   ├── db/               # Database setup
│   │   ├── postgres.py   # SQLAlchemy async engine & session
│   │   └── mongodb.py    # Motor async client
│   ├── models/           # SQLAlchemy ORM models
│   ├── modules/          # Feature modules
│   │   ├── auth/
│   │   │   ├── router.py # FastAPI APIRouter
│   │   │   └── service.py# Business logic
│   │   └── users/
│   └── main.py           # Application entry point
├── tests/                # Pytest suites
├── alembic.ini
└── pyproject.toml
```

**Key Differences from NestJS/Kotlin**:

- **No Controller/Module Classes**: Routing uses `APIRouter` in `router.py`, equivalent to controllers.
- **Dependency Injection**: Relies on FastAPI's `Depends()` defined mostly in `core/deps.py`.
- **Schemas**: Pydantic `BaseModel` handles both DTOs and Swagger schema definitions.

---

## 3. Entry Point (main.py)

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse
from fastapi.middleware.cors import CORSMiddleware

from app.core.config import settings
from app.db.postgres import init_db as init_pg
from app.db.mongodb import init_db as init_mongo
from app.modules.auth.router import router as auth_router
from app.modules.users.router import router as users_router
from app.common.exceptions import ApiError

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: Initialize DB connections
    await init_pg()
    await init_mongo()
    yield
    # Shutdown logic (handled automatically by connection pools usually)

app = FastAPI(
    title=settings.PROJECT_NAME,
    version="1.0.0",
    docs_url="/docs",
    redoc_url=None,
    openapi_url="/openapi.json",
    lifespan=lifespan,
)

# CORS Middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Global Exception Handler
@app.exception_handler(ApiError)
async def api_error_handler(request: Request, exc: ApiError):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "success": False,
            "error": {
                "code": exc.code,
                "message": exc.message,
                "details": exc.details,
            },
            "timestamp": exc.timestamp,
        },
    )

# Routers
app.include_router(auth_router, prefix="/api")
app.include_router(users_router, prefix="/api")

@app.get("/health", tags=["Health"])
async def health_check():
    return {"status": "ok"}
```

---

## 4. Settings (pydantic-settings)

Handles `.env` loading and validation. Replaces NestJS `ConfigModule`.

```python
# app/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import AnyHttpUrl, computed_field

class Settings(BaseSettings):
    PROJECT_NAME: str = "Claude Ready Standards API"
    ENVIRONMENT: str = "development"

    # Database
    DATABASE_URL: str
    MONGODB_URI: str

    # Redis
    REDIS_URL: str

    # Auth
    JWT_PRIVATE_KEY: str
    JWT_PUBLIC_KEY: str
    JWT_ACCESS_EXPIRY: str = "15m"
    GOOGLE_CLIENT_ID: str

    # CORS
    CORS_ORIGINS_STR: str = "http://localhost:3000,http://localhost:15310"

    @computed_field
    @property
    def CORS_ORIGINS(self) -> list[str]:
        return [origin.strip() for origin in self.CORS_ORIGINS_STR.split(",")]

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=True,
        extra="ignore"
    )

settings = Settings()
```

---

## 5. Dependency Injection (core/deps.py)

FastAPI's DI system uses `Depends()`. We centralize common dependencies here to emulate NestJS Guard and Service injection patterns.

```python
# app/core/deps.py
from typing import Annotated
from fastapi import Depends, Request, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession
import jwt

from app.core.config import settings
from app.db.postgres import get_db_session
from app.models.user import User
from app.common.exceptions import ApiError

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/auth/login")

DbSession = Annotated[AsyncSession, Depends(get_db_session)]
Token = Annotated[str, Depends(oauth2_scheme)]

async def get_current_user(token: Token, db: DbSession) -> User:
    try:
        payload = jwt.decode(
            token,
            settings.JWT_PUBLIC_KEY,
            algorithms=["RS256"]
        )
        user_id: str = payload.get("sub")
        if user_id is None:
            raise ApiError(status.HTTP_401_UNAUTHORIZED, "Invalid token format")
    except jwt.InvalidTokenError:
        raise ApiError(status.HTTP_401_UNAUTHORIZED, "Invalid or expired token")

    # Fetch user from DB (SQLAlchemy)
    user = await db.get(User, user_id)
    if not user or not user.is_active:
        raise ApiError(status.HTTP_401_UNAUTHORIZED, "User not found or inactive")

    return user

CurrentUser = Annotated[User, Depends(get_current_user)]

def require_role(allowed_roles: list[str]):
    async def role_checker(user: CurrentUser) -> User:
        if user.role not in allowed_roles:
            raise ApiError(status.HTTP_403_FORBIDDEN, "Insufficient permissions")
        return user
    return Depends(role_checker)
```

**Usage in Router**:

```python
@router.get("/admin/stats")
async def get_stats(
    db: DbSession,
    user: User = require_role(["admin", "editor"])
):
    ...
```

---

## 6. Common Utilities

### 6.1 ApiError Exception

Matches the standardized JSON error response structure.

```python
# app/common/exceptions.py
from datetime import datetime, timezone
from fastapi import status

class ApiError(Exception):
    def __init__(
        self,
        status_code: int,
        message: str,
        code: str | None = None,
        details: dict | list | None = None,
    ):
        self.status_code = status_code
        self.message = message
        # Default code generation based on HTTP status
        self.code = code or f"ERR_{status_code}"
        self.details = details
        self.timestamp = datetime.now(timezone.utc).isoformat()
```

### 6.2 Pagination Request/Response

Replaces NestJS `PaginationDto`.

```python
# app/common/schemas.py
from pydantic import BaseModel, Field, conint
from typing import Generic, TypeVar

T = TypeVar('T')

class PaginationParams(BaseModel):
    page: conint(ge=1) = Field(default=1, description="Page number")
    limit: conint(ge=1, le=100) = Field(default=20, description="Items per page")

class PaginationMeta(BaseModel):
    totalItems: int
    itemCount: int
    itemsPerPage: int
    totalPages: int
    currentPage: int

class PaginatedResponse(BaseModel, Generic[T]):
    success: bool = True
    data: list[T]
    meta: PaginationMeta
    timestamp: str

class ApiResponse(BaseModel, Generic[T]):
    success: bool = True
    data: T
    timestamp: str
```

---

## 7. Database & Repositories

FastAPI doesn't strictly enforce the Repository pattern, but grouping DB calls within `service.py` functions using the injected `AsyncSession` is recommended.

```python
# app/modules/users/service.py
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from app.models.user import User

async def get_user_by_email(db: AsyncSession, email: str) -> User | None:
    result = await db.execute(select(User).where(User.email == email))
    return result.scalars().first()

async def create_user(db: AsyncSession, user_data: dict) -> User:
    new_user = User(**user_data)
    db.add(new_user)
    await db.commit()
    await db.refresh(new_user)
    return new_user
```

---

## 8. OpenAPI (Swagger) Patterns

FastAPI auto-generates Swagger (at `/docs`) based on Pydantic schemas and Router decorators. We must explicitly define metadata to match the NestJS Swagger quality.

### 8.1 DTO Pydantic Schemas

Use `Field` for descriptions and examples.

```python
# app/modules/users/schemas.py
from pydantic import BaseModel, EmailStr, Field

class CreateUserRequest(BaseModel):
    email: EmailStr = Field(
        ...,
        description="email address",
        examples=["user@example.com"]
    )
    role: str = Field(
        default="viewer",
        description="role",
        examples=["viewer"]
    )

class UserResponse(BaseModel):
    id: str = Field(..., examples=["550e8400-e29b-41d4-a716-446655440000"])
    email: str = Field(..., examples=["user@example.com"])
    role: str = Field(..., examples=["viewer"])

    class Config:
        from_attributes = True  # Allows parsing from SQLAlchemy models
```

### 8.2 Response Schema Wrapping

Unlike NestJS's interceptor wrapper, FastAPI requires the return model to encompass the wrapper. We utilize the Generic `ApiResponse`.

```python
from app.common.schemas import PaginatedResponse, PaginationMeta

class UserPaginatedResponse(BaseModel):
    """GET /users pagination response."""

    success: bool = Field(default=True, examples=[True])
    data: list[UserResponse]
    meta: PaginationMeta
    timestamp: str = Field(examples=["2026-02-28T12:00:00+00:00"])
```

### 8.3 Router Decorator Pattern

Matches the NestJS Controller Swagger decorators.

```python
# app/modules/users/router.py
from fastapi import APIRouter, HTTPException, status

from app.common.schemas import ApiError, ApiResponse, PaginationParams
from app.core.deps import CurrentUser, DbSession
from app.modules.users.schemas import (
    CreateUserRequest,
    UserPaginatedResponse,
    UserResponse,
)

router = APIRouter(prefix="/users", tags=["Users"])


@router.get(
    "",
    summary="Fetch paginated users",
    description="Returns a paginated list of users.",
    response_model=UserPaginatedResponse,
    responses={
        status.HTTP_401_UNAUTHORIZED: {"model": ApiError, "description": "Authentication required"},
    },
)
async def find_all(
    db: DbSession,
    user: CurrentUser,
    pagination: PaginationParams = Depends(),
) -> dict:
    ...


@router.get(
    "/{user_id}",
    summary="Fetch single user",
    response_model=ApiResponse[UserResponse],
    responses={
        status.HTTP_401_UNAUTHORIZED: {"model": ApiError, "description": "Authentication required"},
        status.HTTP_404_NOT_FOUND: {"model": ApiError, "description": "User not found"},
    },
)
async def find_one(user_id: str, db: DbSession, user: CurrentUser) -> dict:
    ...


@router.post(
    "",
    summary="Create user",
    status_code=status.HTTP_201_CREATED,
    response_model=ApiResponse[UserResponse],
    responses={
        status.HTTP_400_BAD_REQUEST: {"model": ApiError, "description": "Validation failed"},
        status.HTTP_401_UNAUTHORIZED: {"model": ApiError, "description": "Authentication required"},
        status.HTTP_409_CONFLICT: {"model": ApiError, "description": "Email conflict"},
    },
)
async def create(dto: CreateUserRequest, db: DbSession, user: CurrentUser) -> dict:
    ...
```

### 8.4 Public Endpoint Pattern

Endpoints not requiring auth should omit the `CurrentUser` dependency. The open padlock icon in the OpenAPI UI will intuitively broadcast its public accessibility.

```python
health_router = APIRouter(prefix="/health", tags=["Health"])


@health_router.get(
    "",
    summary="Check server status",
    response_model=dict,
    responses={status.HTTP_200_OK: {"description": "Normal operation"}},
)
async def health_check() -> dict:
    return {"status": "ok"}
```

### 8.5 OpenAPI Schema Checklist

When writing all endpoints, check the following:

- [ ] `tags=["..."]` — Setup router level tags
- [ ] `summary` — Add a one-line description for each endpoint
- [ ] `description` — Add detailed description if necessary
- [ ] `response_model` — Designate 200/201 success response model
- [ ] `responses={}` — Define all possible error status codes (using the `ApiError` model)
- [ ] Apply `Field(description=..., examples=[...])` dynamically alongside every Pydantic attribute
- [ ] Check if `examples` value reflects explicit real-world representations

---

## Verification

```bash
# Typecheck
cd apps/backend-py && uv run mypy app/

# Lint
cd apps/backend-py && uv run ruff check app/

# Start dev server
cd apps/backend-py && uv run uvicorn app.main:app --reload --port 15320

# Test
cd apps/backend-py && uv run pytest --cov=app
```
