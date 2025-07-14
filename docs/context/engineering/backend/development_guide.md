# Backend Development Guide

A concise guide for extending the FastAPI backend with service-oriented architecture.

## Architecture Overview

The backend follows a service-oriented architecture:

```
backend/src/backend/
├── app.py              # FastAPI app setup and middleware
├── main.py             # Application entry point
├── db.py               # Database session management
├── settings.py         # Configuration and environment variables
└── services/           # Feature-based service modules
    ├── auth/           # Authentication (JWT)
    ├── users/          # User management
    ├── email/          # Email service
    └── your_service/   # New services follow this pattern
```

**Key Components**: FastAPI + SQLAlchemy (async) + FastAPI-Users + Alembic + Pydantic

## Service Pattern

Each service follows this structure:

```
services/your_service/
├── __init__.py
├── models.py           # SQLAlchemy database models
├── schemas.py          # Pydantic request/response schemas
├── service.py          # Business logic
├── routes.py           # API endpoints
└── dependencies.py     # FastAPI dependencies (optional)
```

## Database Models

```python
# models.py
from sqlalchemy import Column, Integer, String, DateTime, Boolean
from datetime import datetime
from backend.db import Base

class YourModel(Base):
    __tablename__ = "your_table"
    
    id = Column(Integer, primary_key=True)
    name = Column(String, nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
```

**Migration Workflow**:
1. Create model in `models.py`
2. Generate migration: `uv run alembic revision --autogenerate -m "Add your_table"`
3. Apply migration: `uv run alembic upgrade head`

## API Development

### Schemas
```python
# schemas.py
from pydantic import BaseModel, ConfigDict
from typing import Optional

class YourModelCreate(BaseModel):
    name: str

class YourModelResponse(BaseModel):
    id: int
    name: str
    is_active: bool
    
    model_config = ConfigDict(from_attributes=True)
```

### Service Layer
```python
# service.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from .models import YourModel
from .schemas import YourModelCreate

class YourService:
    @staticmethod
    async def create_item(session: AsyncSession, item_data: YourModelCreate) -> YourModel:
        db_item = YourModel(**item_data.model_dump())
        session.add(db_item)
        await session.commit()
        await session.refresh(db_item)
        return db_item
    
    @staticmethod
    async def get_item(session: AsyncSession, item_id: int) -> YourModel:
        result = await session.execute(select(YourModel).where(YourModel.id == item_id))
        return result.scalar_one_or_none()
```

### Routes
```python
# routes.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession
from backend.db import get_async_session
from backend.services.auth.utils import fastapi_users
from .service import YourService
from .schemas import YourModelResponse, YourModelCreate

router = APIRouter(prefix="/your-service", tags=["your-service"])

@router.post("/", response_model=YourModelResponse)
async def create_item(
    item_data: YourModelCreate,
    session: AsyncSession = Depends(get_async_session),
    current_user = Depends(fastapi_users.current_user())  # Require auth
):
    return await YourService.create_item(session, item_data)

@router.get("/{item_id}", response_model=YourModelResponse)
async def get_item(item_id: int, session: AsyncSession = Depends(get_async_session)):
    item = await YourService.get_item(session, item_id)
    if not item:
        raise HTTPException(status_code=404, detail="Item not found")
    return item
```

## Authentication

Use FastAPI-Users for protected endpoints:

```python
from backend.services.auth.utils import fastapi_users

# Require authentication
current_user = Depends(fastapi_users.current_user())

# Optional authentication
current_user = Depends(fastapi_users.current_user(optional=True))

# Require verified user
current_user = Depends(fastapi_users.current_user(verified=True))
```

## Testing

### Test Structure
```
tests/services/your_service/
├── test_your_service_models.py
├── test_your_service_service.py
└── test_your_service_routes.py
```

### Test Example
```python
# test_your_service_service.py
import pytest
from sqlalchemy.ext.asyncio import AsyncSession
from backend.services.your_service.service import YourService
from backend.services.your_service.schemas import YourModelCreate

@pytest.mark.asyncio
async def test_create_item(async_session: AsyncSession):
    item_data = YourModelCreate(name="Test Item")
    item = await YourService.create_item(async_session, item_data)
    assert item.name == "Test Item"
    assert item.id is not None
```

## Error Handling

Use FastAPI's HTTPException for API errors:

```python
from fastapi import HTTPException, status

# Standard error responses
raise HTTPException(status_code=404, detail="Item not found")
raise HTTPException(status_code=400, detail="Invalid input")
raise HTTPException(status_code=401, detail="Authentication required")
```

## Adding a New Service

1. **Create service directory**:
   ```bash
   mkdir -p src/backend/services/your_service
   touch src/backend/services/your_service/{__init__.py,models.py,schemas.py,service.py,routes.py}
   ```

2. **Implement models, schemas, service, and routes** following the patterns above

3. **Register routes in app.py**:
   ```python
   from backend.services.your_service.routes import router as your_service_router
   app.include_router(your_service_router, prefix="/api")
   ```

4. **Create and run migrations**:
   ```bash
   uv run alembic revision --autogenerate -m "Add your_service tables"
   uv run alembic upgrade head
   ```

5. **Write tests**:
   ```bash
   mkdir -p tests/services/your_service
   touch tests/services/your_service/test_your_service_{models,service,routes}.py
   ```

## Code Standards

- **Follow PEP 8** for formatting
- **Use type hints** for function parameters and returns
- **Use async/await** for all database operations
- **Write docstrings** for public functions
- **Use meaningful variable names**
- **Keep functions small and focused**

## Extension Guidelines

When extending the backend:

1. **Follow existing patterns** - Use the same structure and naming conventions
2. **Handle migrations** - Always create migrations for model changes
3. **Write tests** - Include tests for new functionality
4. **Use proper authentication** - Protect sensitive endpoints
5. **Handle errors** - Use HTTPException for API errors
6. **Use environment variables** - Never hardcode secrets
