---
description: Generate OOP-based API test suite with httpx, Pydantic models, Allure reporting, and proper test data factories. Invokes api-test-writer agent.
---

# API Test Generator

Invokes the **api-test-writer** agent to generate production-grade API tests.

## What This Command Does

1. **Analyze endpoint** -- Read API contract (OpenAPI spec or manual description)
2. **Create Pydantic models** -- Request/response DTOs
3. **Create API service class** -- Methods with @allure.step for each endpoint
4. **Generate test class** -- BaseTest inheritance, Arrange-Act-Assert
5. **Cover all scenarios** -- Positive, negative, edge cases, parametrized
6. **Add fixtures** -- conftest.py with shared setup/teardown

## When to Use

- Adding tests for a new API endpoint
- Expanding test coverage for existing endpoints
- Creating test suite for a new microservice
- Setting up test framework for a new project

## Example

```
/api-test POST /api/v1/users -- create user endpoint, needs auth token

Agent generates:
1. models/user.py -- CreateUserRequest, UserResponse
2. api/users_api.py -- UsersAPI service class
3. tests/test_users/conftest.py -- fixtures (created_user, users_api)
4. tests/test_users/test_create_user.py -- TestCreateUser class
   - test_create_user_returns_201
   - test_create_user_with_duplicate_email_returns_409
   - test_create_user_without_auth_returns_401
   - test_create_user_with_invalid_email_returns_422 (parametrized)
   - test_create_user_with_missing_fields_returns_422 (parametrized)
```

## Generated Code Standards

- All tests in classes inheriting `BaseTest`
- All API calls through service classes
- All methods decorated with `@allure.step`
- All data from `DataGenerator` (no hardcoded values)
- Allure: `@epic`, `@feature`, `@story`, `@severity`
- Arrange-Act-Assert pattern in every test

## Integration

- Use `/qa-review` to review generated tests
- Use `/verify` to run full test suite
- Use `/test-coverage` to check coverage gaps
