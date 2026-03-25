---
name: pydantic-patterns
description: Pydantic v2 patterns — models, validators, serialization, computed fields, discriminated unions, settings management, and integration with FastAPI/SQLAlchemy.
origin: custom
---

# Pydantic v2 Patterns

Data validation and serialization patterns using Pydantic v2.

## When to Activate

- Defining request/response schemas for APIs
- Validating external data (user input, API responses, file content)
- Managing application settings with environment variables
- Converting between domain models and DTOs

## Basic Models

```python
from pydantic import BaseModel, Field, EmailStr
from datetime import datetime

class UserCreate(BaseModel):
    email: EmailStr
    name: str = Field(min_length=1, max_length=255)
    password: str = Field(min_length=8, max_length=128)
    age: int | None = Field(None, ge=0, le=150)

class UserResponse(BaseModel):
    id: int
    email: str
    name: str
    created_at: datetime

    model_config = {"from_attributes": True}  # Enable ORM mode
```

## Validators

### Field Validators

```python
from pydantic import BaseModel, field_validator, model_validator
import re

class UserCreate(BaseModel):
    username: str
    password: str
    password_confirm: str

    @field_validator("username")
    @classmethod
    def username_alphanumeric(cls, v: str) -> str:
        if not re.match(r"^[a-zA-Z0-9_]+$", v):
            raise ValueError("Username must be alphanumeric")
        return v.lower()

    @field_validator("password")
    @classmethod
    def password_strength(cls, v: str) -> str:
        if not re.search(r"[A-Z]", v):
            raise ValueError("Password must contain uppercase letter")
        if not re.search(r"[0-9]", v):
            raise ValueError("Password must contain a digit")
        return v

    @model_validator(mode="after")
    def passwords_match(self) -> "UserCreate":
        if self.password != self.password_confirm:
            raise ValueError("Passwords do not match")
        return self
```

### Before and After Validators

```python
from pydantic import field_validator

class Product(BaseModel):
    name: str
    price: float
    tags: list[str] = []

    @field_validator("name", mode="before")
    @classmethod
    def strip_name(cls, v):
        if isinstance(v, str):
            return v.strip()
        return v

    @field_validator("tags", mode="before")
    @classmethod
    def parse_tags(cls, v):
        if isinstance(v, str):
            return [t.strip() for t in v.split(",")]
        return v

    @field_validator("price")
    @classmethod
    def round_price(cls, v: float) -> float:
        return round(v, 2)
```

## Computed Fields

```python
from pydantic import BaseModel, computed_field

class OrderItem(BaseModel):
    product_name: str
    quantity: int
    unit_price: float

    @computed_field
    @property
    def total_price(self) -> float:
        return round(self.quantity * self.unit_price, 2)

class Order(BaseModel):
    items: list[OrderItem]

    @computed_field
    @property
    def grand_total(self) -> float:
        return round(sum(item.total_price for item in self.items), 2)
```

## Discriminated Unions

```python
from pydantic import BaseModel, Discriminator, Tag
from typing import Annotated, Literal

class EmailNotification(BaseModel):
    type: Literal["email"] = "email"
    to: str
    subject: str
    body: str

class SMSNotification(BaseModel):
    type: Literal["sms"] = "sms"
    phone: str
    message: str

class PushNotification(BaseModel):
    type: Literal["push"] = "push"
    device_token: str
    title: str
    body: str

Notification = Annotated[
    EmailNotification | SMSNotification | PushNotification,
    Discriminator("type"),
]

class NotificationRequest(BaseModel):
    notifications: list[Notification]
```

## Serialization Control

```python
from pydantic import BaseModel, field_serializer, model_serializer
from datetime import datetime

class Event(BaseModel):
    name: str
    start_time: datetime
    metadata: dict

    @field_serializer("start_time")
    def serialize_time(self, v: datetime, _info) -> str:
        return v.strftime("%Y-%m-%d %H:%M")

    # Exclude None values
    model_config = {
        "json_schema_extra": {"examples": [{"name": "Conference", "start_time": "2025-01-15 09:00"}]}
    }

# Usage
event = Event(name="Conf", start_time=datetime.now(), metadata={"key": "val"})
event.model_dump(exclude_none=True)          # dict without None values
event.model_dump(include={"name", "start_time"})  # only specific fields
event.model_dump_json()                      # JSON string
```

## Nested Models

```python
class Address(BaseModel):
    street: str
    city: str
    country: str = "US"
    zip_code: str

class Company(BaseModel):
    name: str
    address: Address
    employees: list["Employee"] = []

class Employee(BaseModel):
    name: str
    email: EmailStr
    department: str
    company: Company | None = None

# Parsing nested data
data = {
    "name": "Acme Inc",
    "address": {"street": "123 Main St", "city": "NYC", "zip_code": "10001"},
    "employees": [
        {"name": "Alice", "email": "alice@acme.com", "department": "Engineering"},
    ],
}
company = Company.model_validate(data)
```

## Settings Management

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import SecretStr

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        env_prefix="APP_",
        case_sensitive=False,
    )

    # Required
    database_url: str
    secret_key: SecretStr
    redis_url: str

    # Optional with defaults
    debug: bool = False
    log_level: str = "INFO"
    port: int = 8000
    allowed_origins: list[str] = ["http://localhost:3000"]

    # Nested settings
    class DBConfig(BaseSettings):
        pool_size: int = 20
        max_overflow: int = 10

    db: DBConfig = DBConfig()

# .env file:
# APP_DATABASE_URL=postgres://user:pass@localhost/db
# APP_SECRET_KEY=my-secret-key
# APP_DEBUG=true
# APP_ALLOWED_ORIGINS=["http://localhost:3000","http://localhost:8080"]

settings = Settings()
```

## Generic Response Models

```python
from pydantic import BaseModel
from typing import TypeVar, Generic

T = TypeVar("T")

class APIResponse(BaseModel, Generic[T]):
    success: bool = True
    data: T | None = None
    error: str | None = None

class PaginatedData(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    per_page: int

    @computed_field
    @property
    def pages(self) -> int:
        return (self.total + self.per_page - 1) // self.per_page

# Usage in FastAPI
@router.get("/users", response_model=APIResponse[PaginatedData[UserResponse]])
async def list_users(): ...
```

## Integration with SQLAlchemy

```python
from pydantic import BaseModel

class UserResponse(BaseModel):
    id: int
    email: str
    name: str

    model_config = {"from_attributes": True}

# Converts SQLAlchemy model to Pydantic
user_orm = await session.get(User, 1)
user_schema = UserResponse.model_validate(user_orm)
```

## Custom Types

```python
from pydantic import GetCoreSchemaHandler
from pydantic_core import core_schema

class PhoneNumber(str):
    @classmethod
    def __get_pydantic_core_schema__(cls, source, handler: GetCoreSchemaHandler):
        return core_schema.no_info_after_validator_function(
            cls._validate,
            core_schema.str_schema(min_length=10, max_length=15),
        )

    @classmethod
    def _validate(cls, v: str) -> "PhoneNumber":
        cleaned = "".join(c for c in v if c.isdigit() or c == "+")
        if not cleaned.startswith("+"):
            cleaned = f"+{cleaned}"
        return cls(cleaned)

class Contact(BaseModel):
    name: str
    phone: PhoneNumber
```

## Quick Reference

| Feature | Usage |
|---------|-------|
| `model_validate(data)` | Parse dict/ORM to model |
| `model_dump()` | Convert to dict |
| `model_dump_json()` | Convert to JSON string |
| `model_json_schema()` | Generate JSON Schema |
| `@field_validator` | Validate single field |
| `@model_validator` | Validate across fields |
| `@computed_field` | Derived field in output |
| `from_attributes=True` | Enable ORM mode |
| `Field(ge=0, le=100)` | Numeric constraints |
| `SecretStr` | Hide sensitive values |
| `Discriminator("field")` | Tagged unions |

**Remember**: Pydantic v2 uses `model_validate` instead of `parse_obj`, `model_dump` instead of `dict()`. Always use the v2 API.
