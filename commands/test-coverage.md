# Test Coverage

Analyze test coverage, identify gaps, and generate missing tests to reach 80%+ coverage.

## Step 1: Run Coverage

```bash
pytest --cov=src --cov-report=json --cov-report=term-missing
```

## Step 2: Analyze Coverage Report

1. Parse the output
2. List files **below 80% coverage**, sorted worst-first
3. For each under-covered file, identify:
   - Untested functions or methods
   - Missing branch coverage (if/else, try/except)
   - Dead code that inflates the denominator

## Step 3: Generate Missing Tests

Priority:
1. **Happy path** -- Core functionality with valid inputs
2. **Error handling** -- Invalid inputs, missing data, network failures
3. **Edge cases** -- Empty lists, None, boundary values (0, -1)
4. **Branch coverage** -- Each if/else, try/except path

### Test Generation Rules

- Place tests in `tests/` directory following project convention
- Use existing test patterns (fixtures, mocking approach)
- Mock external dependencies (database, APIs, file system)
- Each test independent -- no shared mutable state
- Name descriptively: `test_create_user_with_duplicate_email_returns_409`

## Step 4: Verify

1. Run full test suite -- all tests must pass
2. Re-run coverage -- verify improvement
3. If still below 80%, repeat Step 3

## Step 5: Report

```
Coverage Report
------------------------------
File                     Before  After
src/services/auth.py     45%     88%
src/utils/validation.py  32%     82%
------------------------------
Overall:                 67%     84%
```
