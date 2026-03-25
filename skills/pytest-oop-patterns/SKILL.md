---
name: pytest-oop-patterns
description: OOP-based pytest patterns for API test automation — base classes, service layers, fixtures hierarchy, parametrization, markers, parallel execution, and test isolation.
origin: custom
---

# OOP Pytest Patterns for API Testing

Strict OOP patterns for building maintainable, scalable pytest test suites.

## When to Activate

- Designing test framework architecture
- Writing new API test suites
- Reviewing test code for OOP compliance
- Setting up pytest infrastructure (conftest, fixtures, markers)

## RULE: No Bare Functions

Every test MUST be inside a class. This is non-negotiable.

```python
# WRONG -- procedural, no encapsulation
def test_create_user():
    response = httpx.post("/users", json={"email": "a@b.com"})
    assert response.status_code == 201

# CORRECT -- OOP, inherits BaseTest, uses service class
@allure.feature("User Management")
class TestCreateUser(BaseTest):
    def test_create_user_returns_201(self):
        data = self.gen.create_user_request()
        response = self.users_api.create_user(data)
        assert response.status_code == 201
```

## Base Test Class

```python
import pytest
import allure
from framework.clients.http_client import HTTPClient
from framework.config.settings import Settings


class BaseTest:
    """Base class for ALL test classes."""

    client: HTTPClient
    settings: Settings

    @pytest.fixture(autouse=True)
    def _setup_base(self, http_client: HTTPClient, settings: Settings):
        """Inject shared dependencies."""
        self.client = http_client
        self.settings = settings


class BaseAuthenticatedTest(BaseTest):
    """Base for tests requiring authentication."""

    @pytest.fixture(autouse=True)
    def _setup_auth(self, authenticated_client: HTTPClient):
        self.client = authenticated_client
```

## API Service Layer

```python
from framework.base.base_api_client import BaseAPIClient


class BaseAPIClient:
    """Base for all API service classes."""

    def __init__(self, client: HTTPClient):
        self.client = client


class UsersAPI(BaseAPIClient):
    ENDPOINT = "/api/v1/users"

    @allure.step("POST /users — Create user: {data.email}")
    def create_user(self, data: CreateUserRequest) -> httpx.Response:
        return self.client.post(self.ENDPOINT, json=data.model_dump())

    @allure.step("GET /users/{user_id}")
    def get_user(self, user_id: int) -> httpx.Response:
        return self.client.get(f"{self.ENDPOINT}/{user_id}")

    @allure.step("GET /users — List (page={page})")
    def list_users(self, page: int = 1, per_page: int = 20) -> httpx.Response:
        return self.client.get(self.ENDPOINT, params={"page": page, "per_page": per_page})

    @allure.step("PATCH /users/{user_id}")
    def update_user(self, user_id: int, data: dict) -> httpx.Response:
        return self.client.patch(f"{self.ENDPOINT}/{user_id}", json=data)

    @allure.step("DELETE /users/{user_id}")
    def delete_user(self, user_id: int) -> httpx.Response:
        return self.client.delete(f"{self.ENDPOINT}/{user_id}")
```

## Fixtures Hierarchy

### Root conftest.py (session-scoped)

```python
# conftest.py
import pytest
from framework.clients.http_client import HTTPClient
from framework.config.settings import Settings


@pytest.fixture(scope="session")
def settings() -> Settings:
    return Settings()

@pytest.fixture(scope="session")
def http_client(settings: Settings) -> HTTPClient:
    client = HTTPClient(settings)
    yield client
    client.close()

@pytest.fixture(scope="session")
def auth_token(http_client, settings) -> str:
    from api.auth_api import AuthAPI
    auth = AuthAPI(http_client)
    resp = auth.login(settings.TEST_USER_EMAIL, settings.TEST_USER_PASSWORD)
    return resp.json()["access_token"]

@pytest.fixture
def authenticated_client(settings, auth_token) -> HTTPClient:
    client = HTTPClient(settings, token=auth_token)
    yield client
    client.close()
```

### Module conftest.py (test-specific)

```python
# tests/test_users/conftest.py
import pytest
from api.users_api import UsersAPI
from framework.helpers.data_generator import DataGenerator


@pytest.fixture
def users_api(http_client) -> UsersAPI:
    return UsersAPI(http_client)

@pytest.fixture
def created_user(users_api: UsersAPI) -> dict:
    """Create a user and return response body. Cleanup after test."""
    data = DataGenerator.create_user_request()
    response = users_api.create_user(data)
    user = response.json()
    yield user
    users_api.delete_user(user["id"])  # Cleanup
```

## Parametrization Patterns

### Data-driven tests

```python
@allure.feature("Input Validation")
class TestUserValidation(BaseTest):

    @pytest.mark.parametrize("field,value,error_field", [
        ("email", "not-email", "email"),
        ("email", "", "email"),
        ("password", "123", "password"),
        ("name", "", "name"),
        ("name", "x" * 256, "name"),
    ], ids=[
        "invalid-email-format",
        "empty-email",
        "short-password",
        "empty-name",
        "name-too-long",
    ])
    @allure.story("Field validation")
    def test_create_user_field_validation(self, field: str, value: str, error_field: str):
        data = self.gen.create_user_request(**{field: value})
        response = self.users_api.create_user(data)

        assert response.status_code == 422
        errors = response.json()
        assert any(error_field in str(e) for e in errors.get("detail", []))
```

### Status code matrix

```python
@allure.feature("Authorization")
class TestUserAuthorization(BaseTest):

    @pytest.mark.parametrize("method,endpoint,expected", [
        ("GET", "/api/v1/users", 401),
        ("POST", "/api/v1/users", 401),
        ("GET", "/api/v1/users/1", 401),
        ("DELETE", "/api/v1/users/1", 401),
    ], ids=["list", "create", "get", "delete"])
    @allure.story("Unauthorized access returns 401")
    def test_unauthenticated_returns_401(self, method: str, endpoint: str, expected: int):
        response = getattr(self.client, method.lower())(endpoint)
        assert response.status_code == expected
```

## Markers

```ini
# pyproject.toml
[tool.pytest.ini_options]
markers = [
    "smoke: Quick sanity checks (< 30 seconds total)",
    "regression: Full regression suite",
    "critical: Business-critical flows",
    "negative: Negative/error scenarios",
    "slow: Tests > 10 seconds",
    "flaky: Known flaky tests (quarantined)",
]
```

```python
@pytest.mark.smoke
@pytest.mark.critical
class TestAuthSmoke(BaseAuthenticatedTest):
    def test_login_returns_token(self): ...

    def test_token_refresh_works(self): ...
```

```bash
# Run smoke suite
pytest -m smoke --alluredir=allure-results

# Run everything except flaky
pytest -m "not flaky"

# Run critical regression
pytest -m "critical and regression"
```

## Parallel Execution

```bash
# Install
pip install pytest-xdist

# Run parallel (auto-detect CPU count)
pytest -n auto --alluredir=allure-results

# Fixed worker count
pytest -n 4
```

**Rules for parallel-safe tests:**
- No shared mutable state (class variables, global dicts)
- Each test creates its own entities
- Fixtures with `scope="session"` must be thread-safe
- Use unique identifiers (UUID) in test data, not sequential IDs

## Custom Assertions

```python
import allure
from framework.models.response_models import ErrorResponse


class Assertions:
    """Custom assertion helpers with Allure integration."""

    @staticmethod
    @allure.step("Assert status code is {expected}")
    def assert_status(response, expected: int):
        assert response.status_code == expected, (
            f"Expected {expected}, got {response.status_code}. "
            f"Body: {response.text[:200]}"
        )

    @staticmethod
    @allure.step("Assert response matches schema {model.__name__}")
    def assert_schema(response, model: type):
        """Validate response body against Pydantic model."""
        data = response.json()
        parsed = model.model_validate(data)
        return parsed

    @staticmethod
    @allure.step("Assert error response contains: {expected_detail}")
    def assert_error(response, expected_status: int, expected_detail: str | None = None):
        assert response.status_code == expected_status
        if expected_detail:
            error = ErrorResponse.model_validate(response.json())
            assert expected_detail in error.detail
```

## Quick Reference

| Pattern | Description |
|---------|-------------|
| `BaseTest` | All test classes inherit from this |
| `BaseAPIClient` | All API services inherit from this |
| `@allure.step` | Every API method gets this decorator |
| `DataGenerator` | Factory for test data (faker-based) |
| `Assertions` | Custom assertion helpers |
| `conftest.py` hierarchy | Root (session) -> module -> suite |
| `@pytest.mark.parametrize` | Data-driven testing |
| `pytest -n auto` | Parallel execution |
| Arrange-Act-Assert | Mandatory test structure |
