---
name: api-testing-patterns
description: API test automation patterns — httpx/requests client wrappers, response validation with Pydantic, test data factories, retry/polling utilities, schema testing, and contract testing.
origin: custom
---

# API Testing Patterns

Production patterns for API test automation with httpx, requests, Pydantic, and pytest.

## When to Activate

- Writing HTTP API tests (REST, GraphQL)
- Building reusable HTTP client wrappers
- Validating API responses against schemas
- Generating test data with factories
- Implementing retry/polling for async operations
- Setting up contract testing

## HTTP Client: httpx (Recommended)

### Why httpx over requests

| Feature | httpx | requests |
|---------|-------|----------|
| Async support | Native | No |
| HTTP/2 | Yes | No |
| Timeout config | Granular | Basic |
| Connection pooling | Built-in | Built-in |
| Type hints | Full | Partial |
| API compatibility | requests-like | N/A |

### Client Wrapper

```python
import httpx
import allure
import logging
from framework.config.settings import Settings

logger = logging.getLogger(__name__)


class HTTPClient:
    def __init__(self, settings: Settings, token: str | None = None):
        headers = {"Content-Type": "application/json", "Accept": "application/json"}
        if token:
            headers["Authorization"] = f"Bearer {token}"

        self._client = httpx.Client(
            base_url=settings.BASE_URL,
            headers=headers,
            timeout=httpx.Timeout(
                connect=5.0,
                read=settings.REQUEST_TIMEOUT,
                write=5.0,
                pool=5.0,
            ),
        )

    @allure.step("{method} {url}")
    def request(self, method: str, url: str, **kwargs) -> httpx.Response:
        response = self._client.request(method, url, **kwargs)
        self._log(method, url, response)
        self._attach_allure(method, url, response)
        return response

    def get(self, url: str, **kwargs) -> httpx.Response:
        return self.request("GET", url, **kwargs)

    def post(self, url: str, **kwargs) -> httpx.Response:
        return self.request("POST", url, **kwargs)

    def put(self, url: str, **kwargs) -> httpx.Response:
        return self.request("PUT", url, **kwargs)

    def patch(self, url: str, **kwargs) -> httpx.Response:
        return self.request("PATCH", url, **kwargs)

    def delete(self, url: str, **kwargs) -> httpx.Response:
        return self.request("DELETE", url, **kwargs)

    def _log(self, method: str, url: str, response: httpx.Response):
        logger.info(f"{method} {response.request.url} -> {response.status_code} ({response.elapsed.total_seconds():.2f}s)")

    def _attach_allure(self, method: str, url: str, response: httpx.Response):
        allure.attach(str(response.request.url), "URL", allure.attachment_type.TEXT)
        if response.request.content:
            allure.attach(response.request.content.decode(), "Request", allure.attachment_type.JSON)
        allure.attach(response.text, f"Response [{response.status_code}]", allure.attachment_type.JSON)

    def close(self):
        self._client.close()
```

## Response Validation with Pydantic

```python
from pydantic import BaseModel
from httpx import Response


class ResponseValidator:
    """Validate API responses against Pydantic models."""

    @staticmethod
    @allure.step("Validate response against {model.__name__}")
    def validate(response: Response, model: type[BaseModel]) -> BaseModel:
        assert response.status_code < 400, (
            f"Expected success, got {response.status_code}: {response.text[:200]}"
        )
        return model.model_validate(response.json())

    @staticmethod
    @allure.step("Validate list response against {model.__name__}")
    def validate_list(response: Response, model: type[BaseModel]) -> list[BaseModel]:
        data = response.json()
        items = data if isinstance(data, list) else data.get("items", data.get("results", []))
        return [model.model_validate(item) for item in items]

    @staticmethod
    @allure.step("Validate error response: expected {expected_status}")
    def validate_error(response: Response, expected_status: int, contains: str | None = None):
        assert response.status_code == expected_status, (
            f"Expected {expected_status}, got {response.status_code}"
        )
        if contains:
            assert contains.lower() in response.text.lower(), (
                f"Expected '{contains}' in response: {response.text[:200]}"
            )
```

## Test Data Factory

```python
from faker import Faker
from pydantic import BaseModel, EmailStr, Field

fake = Faker()


class CreateUserRequest(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8)
    name: str


class DataGenerator:
    """Test data factory. NEVER hardcode values in tests."""

    @staticmethod
    def user(**overrides) -> CreateUserRequest:
        defaults = {
            "email": fake.unique.email(),
            "password": fake.password(length=12, special_chars=True),
            "name": fake.name(),
        }
        defaults.update(overrides)
        return CreateUserRequest(**defaults)

    @staticmethod
    def users(count: int, **overrides) -> list[CreateUserRequest]:
        return [DataGenerator.user(**overrides) for _ in range(count)]

    @staticmethod
    def random_string(length: int = 10) -> str:
        return fake.pystr(max_chars=length)

    @staticmethod
    def random_email() -> str:
        return fake.unique.email()

    @staticmethod
    def random_int(min_val: int = 1, max_val: int = 10000) -> int:
        return fake.random_int(min=min_val, max=max_val)
```

## Polling / Waiter (Replace time.sleep)

```python
import time
import allure
from typing import Callable


class Waiter:
    """Poll until condition is met. NEVER use time.sleep() in tests."""

    @staticmethod
    @allure.step("Wait for condition (timeout={timeout}s, poll={poll_interval}s)")
    def wait_for(
        condition: Callable[[], bool],
        timeout: float = 30.0,
        poll_interval: float = 1.0,
        message: str = "Condition not met",
    ) -> None:
        deadline = time.time() + timeout
        last_error = None
        while time.time() < deadline:
            try:
                if condition():
                    return
            except Exception as e:
                last_error = e
            time.sleep(poll_interval)
        raise TimeoutError(f"{message} after {timeout}s. Last error: {last_error}")

    @staticmethod
    @allure.step("Wait for status {expected_status} on {url}")
    def wait_for_status(
        client: "HTTPClient",
        url: str,
        expected_status: int,
        timeout: float = 30.0,
    ) -> "httpx.Response":
        deadline = time.time() + timeout
        while time.time() < deadline:
            response = client.get(url)
            if response.status_code == expected_status:
                return response
            time.sleep(1.0)
        raise TimeoutError(
            f"Expected status {expected_status} on {url}, "
            f"last got {response.status_code} after {timeout}s"
        )
```

## Retry Decorator

```python
import functools
import time


def retry(max_attempts: int = 3, delay: float = 1.0, backoff: float = 2.0, exceptions: tuple = (Exception,)):
    """Retry decorator for flaky operations (NOT for masking real failures)."""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exc = None
            current_delay = delay
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    last_exc = e
                    if attempt < max_attempts - 1:
                        time.sleep(current_delay)
                        current_delay *= backoff
            raise last_exc
        return wrapper
    return decorator
```

## Configuration

```python
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    BASE_URL: str = "http://localhost:8000"
    ENV_NAME: str = "local"

    # Auth
    TEST_USER_EMAIL: str = "test@example.com"
    TEST_USER_PASSWORD: str = "testpass123"

    # Timeouts
    REQUEST_TIMEOUT: float = 30.0
    POLL_TIMEOUT: float = 60.0
    POLL_INTERVAL: float = 2.0

    model_config = {"env_file": ".env", "env_prefix": "TEST_"}
```

## Schema / Contract Testing

```python
import allure
from jsonschema import validate, ValidationError


class SchemaValidator:

    @staticmethod
    @allure.step("Validate JSON schema for {schema_name}")
    def validate_schema(response_json: dict, schema: dict, schema_name: str = "response"):
        try:
            validate(instance=response_json, schema=schema)
        except ValidationError as e:
            allure.attach(str(e), "Schema Validation Error", allure.attachment_type.TEXT)
            raise AssertionError(f"Schema validation failed for {schema_name}: {e.message}")


# Usage in test
class TestUserSchema(BaseTest):
    USER_SCHEMA = {
        "type": "object",
        "required": ["id", "email", "name", "created_at"],
        "properties": {
            "id": {"type": "integer"},
            "email": {"type": "string", "format": "email"},
            "name": {"type": "string", "minLength": 1},
            "created_at": {"type": "string", "format": "date-time"},
        },
        "additionalProperties": False,
    }

    @allure.story("Response matches schema")
    def test_get_user_matches_schema(self, created_user):
        response = self.users_api.get_user(created_user["id"])
        SchemaValidator.validate_schema(response.json(), self.USER_SCHEMA, "User")
```

## Soft Assertions

```python
class SoftAssertions:
    """Collect multiple assertion failures before raising."""

    def __init__(self):
        self._failures: list[str] = []

    def check(self, condition: bool, message: str):
        if not condition:
            self._failures.append(message)

    def check_equal(self, actual, expected, field: str):
        if actual != expected:
            self._failures.append(f"{field}: expected {expected!r}, got {actual!r}")

    def assert_all(self):
        if self._failures:
            allure.attach(
                "\n".join(f"- {f}" for f in self._failures),
                "Soft Assertion Failures",
                allure.attachment_type.TEXT,
            )
            raise AssertionError(f"{len(self._failures)} assertion(s) failed:\n" + "\n".join(self._failures))


# Usage
def test_user_fields(self, created_user):
    response = self.users_api.get_user(created_user["id"])
    body = response.json()

    soft = SoftAssertions()
    soft.check_equal(body["email"], created_user["email"], "email")
    soft.check_equal(body["name"], created_user["name"], "name")
    soft.check("id" in body, "Response must contain 'id'")
    soft.check("created_at" in body, "Response must contain 'created_at'")
    soft.assert_all()
```

## Quick Reference

| Pattern | Purpose |
|---------|---------|
| `HTTPClient` | Wrapper with logging + Allure |
| `ResponseValidator` | Pydantic-based response validation |
| `DataGenerator` | Faker-based test data factory |
| `Waiter` | Polling instead of `time.sleep()` |
| `SoftAssertions` | Collect multiple failures |
| `SchemaValidator` | JSON Schema contract testing |
| `retry()` | Retry flaky operations (not tests!) |
| `Settings` | Environment-based config |
