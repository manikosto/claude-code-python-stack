---
description: Review test automation code for OOP compliance, Allure metadata, proper abstractions, and flaky patterns. Invokes qa-reviewer agent.
---

# QA Test Code Review

Invokes the **qa-reviewer** agent for comprehensive test code review.

## What This Command Does

1. **OOP Compliance** -- Verify all tests are in classes, inherit BaseTest
2. **Abstraction Check** -- API calls through service classes, not raw httpx/requests
3. **Allure Metadata** -- @epic, @feature, @story, @severity on every test
4. **Flaky Detection** -- Find time.sleep(), shared state, order-dependent tests
5. **Model Usage** -- Pydantic models for request/response, not raw dicts
6. **Test Design** -- Positive + negative + edge cases, parametrization where needed

## When to Use

- After writing new API tests
- Before merging test PRs
- Reviewing test framework changes
- Onboarding new QA engineers to the project

## Checks Run

```bash
# Verify no bare test functions
grep -rn "^def test_" tests/ --include="*.py"

# Check Allure metadata coverage
grep -rn "@allure.epic\|@allure.feature\|@allure.story" tests/ --include="*.py" | wc -l

# Check for sleep() usage
grep -rn "time.sleep" tests/ --include="*.py"

# Check for direct HTTP calls in tests
grep -rn "httpx\.\|requests\." tests/test_* --include="*.py"

# Collect tests
pytest --collect-only -q
```

## Approval Criteria

| Status | Condition |
|--------|-----------|
| Approve | No OOP violations, Allure metadata complete, no flaky patterns |
| Warning | Minor Allure gaps, missing edge cases |
| Block | Bare functions, direct HTTP calls, time.sleep(), no Allure steps |
