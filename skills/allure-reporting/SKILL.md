---
name: allure-reporting
description: Allure Report integration with pytest — decorators, steps, attachments, severity levels, test categorization, CI/CD integration, and custom report plugins.
origin: custom
---

# Allure Reporting Patterns

Comprehensive Allure Report integration for pytest test suites.

## When to Activate

- Setting up Allure in a pytest project
- Adding Allure metadata to test classes
- Configuring Allure in CI/CD pipelines
- Debugging test report rendering issues
- Creating custom Allure plugins or extensions

## Setup

```bash
# Install
pip install allure-pytest

# Run tests with Allure
pytest --alluredir=allure-results -v

# Generate and open report
allure serve allure-results

# Generate static report
allure generate allure-results -o allure-report --clean
```

### pyproject.toml

```toml
[tool.pytest.ini_options]
addopts = "--alluredir=allure-results --clean-alluredir"
```

## Allure Decorator Hierarchy

### Required on EVERY test class and method:

```python
import allure

@allure.epic("E-Commerce Platform")           # Top-level product area
@allure.feature("Shopping Cart")               # Feature being tested
class TestAddToCart(BaseTest):

    @allure.story("Add single item")           # User story
    @allure.severity(allure.severity_level.CRITICAL)  # Priority
    @allure.title("User can add product to cart")     # Human-readable title
    @allure.description("Verify that authenticated user can add a product to their cart")
    @allure.tag("cart", "positive")
    @allure.link("https://jira.example.com/PROJ-123", name="PROJ-123")
    def test_add_product_to_cart(self):
        ...
```

### Severity Levels

| Level | When to Use | Example |
|-------|-------------|---------|
| `BLOCKER` | App unusable if fails | Login, payment processing |
| `CRITICAL` | Major feature broken | User creation, order placement |
| `NORMAL` | Standard functionality | Profile update, search |
| `MINOR` | Minor inconvenience | Pagination, sorting |
| `TRIVIAL` | Cosmetic | UI text, formatting |

## Allure Steps

### Method-level steps (API service classes)

```python
class OrdersAPI(BaseAPIClient):

    @allure.step("Create order with {len(items)} items")
    def create_order(self, items: list[dict]) -> httpx.Response:
        return self.client.post("/api/v1/orders", json={"items": items})

    @allure.step("Get order #{order_id} status")
    def get_order_status(self, order_id: int) -> httpx.Response:
        return self.client.get(f"/api/v1/orders/{order_id}/status")
```

### Inline steps (inside test methods)

```python
def test_complete_order_flow(self):
    with allure.step("Create a new user"):
        user = self.create_test_user()

    with allure.step("Add items to cart"):
        self.cart_api.add_item(user["id"], product_id=1, quantity=2)

    with allure.step("Place order"):
        response = self.orders_api.create_order(user["id"])
        assert response.status_code == 201
        order_id = response.json()["id"]

    with allure.step("Verify order status is 'pending'"):
        status_resp = self.orders_api.get_order_status(order_id)
        assert status_resp.json()["status"] == "pending"
```

## Attachments

### Attach request/response to every API call

```python
class HTTPClient:
    def _attach_to_allure(self, response: httpx.Response):
        allure.attach(
            body=str(response.request.url),
            name="Request URL",
            attachment_type=allure.attachment_type.TEXT,
        )
        if response.request.content:
            allure.attach(
                body=response.request.content.decode("utf-8"),
                name="Request Body",
                attachment_type=allure.attachment_type.JSON,
            )
        allure.attach(
            body=response.text,
            name=f"Response [{response.status_code}]",
            attachment_type=allure.attachment_type.JSON,
        )
        allure.attach(
            body=str(dict(response.headers)),
            name="Response Headers",
            attachment_type=allure.attachment_type.JSON,
        )
```

### Attach screenshots (E2E/UI tests)

```python
@allure.step("Take screenshot: {name}")
def attach_screenshot(page, name: str):
    screenshot = page.screenshot()
    allure.attach(screenshot, name=name, attachment_type=allure.attachment_type.PNG)
```

### Attach logs on failure

```python
@pytest.fixture(autouse=True)
def attach_logs_on_failure(request, caplog):
    yield
    if request.node.rep_call and request.node.rep_call.failed:
        allure.attach(
            body=caplog.text,
            name="Test Logs",
            attachment_type=allure.attachment_type.TEXT,
        )

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    rep = outcome.get_result()
    setattr(item, f"rep_{rep.when}", rep)
```

## Dynamic Allure Metadata

```python
class BaseTest:
    @pytest.fixture(autouse=True)
    def _set_allure_metadata(self, request):
        """Auto-set Allure metadata from test class/method."""
        if request.cls:
            allure.dynamic.parent_suite(request.cls.__module__)
            allure.dynamic.suite(request.cls.__name__)
        allure.dynamic.sub_suite(request.node.name)

        # Add environment info
        allure.dynamic.parameter("env", self.settings.ENV_NAME)
```

## Environment Info

```python
# conftest.py — generate environment.properties for Allure
import pytest

def pytest_sessionfinish(session, exitstatus):
    """Write environment info for Allure report."""
    allure_dir = session.config.getoption("--alluredir", default=None)
    if allure_dir:
        env_file = Path(allure_dir) / "environment.properties"
        settings = Settings()
        env_file.write_text(
            f"Base.URL={settings.BASE_URL}\n"
            f"Environment={settings.ENV_NAME}\n"
            f"Python={sys.version}\n"
            f"pytest={pytest.__version__}\n"
        )
```

## Categories (defect classification)

```json
// allure-results/categories.json
[
  {
    "name": "API Errors",
    "matchedStatuses": ["failed"],
    "messageRegex": ".*assert.*status_code.*"
  },
  {
    "name": "Schema Validation Failures",
    "matchedStatuses": ["failed"],
    "messageRegex": ".*validation error.*|.*ValidationError.*"
  },
  {
    "name": "Infrastructure Issues",
    "matchedStatuses": ["broken"],
    "messageRegex": ".*ConnectionError.*|.*TimeoutError.*"
  },
  {
    "name": "Flaky Tests",
    "matchedStatuses": ["failed"],
    "traceRegex": ".*flaky.*"
  }
]
```

## CI/CD Integration

### GitHub Actions

```yaml
name: API Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: pip install -r requirements-test.txt

      - name: Run tests
        run: pytest --alluredir=allure-results -v -n auto
        env:
          BASE_URL: ${{ secrets.TEST_BASE_URL }}
          TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}

      - name: Upload Allure results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: allure-results
          path: allure-results/

      - name: Allure Report
        if: always()
        uses: simple-elf/allure-report-action@master
        with:
          allure_results: allure-results

      - name: Deploy report to GitHub Pages
        if: always()
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: allure-report
```

## Quick Reference

| Decorator | Level | Required |
|-----------|-------|----------|
| `@allure.epic("...")` | Test class | Yes |
| `@allure.feature("...")` | Test class | Yes |
| `@allure.story("...")` | Test method | Yes |
| `@allure.severity(...)` | Test method | Yes |
| `@allure.title("...")` | Test method | Recommended |
| `@allure.step("...")` | API methods | Yes |
| `@allure.tag("...")` | Test method | Optional |
| `@allure.link("...")` | Test method | Optional |
| `allure.attach(...)` | Inside steps | For API calls |
| `allure.dynamic.*` | Runtime | For base classes |

**Remember**: A good Allure report tells a story. Epic -> Feature -> Story -> Steps. Anyone reading the report should understand what was tested, how, and why it failed.
