---
description: Enforce test-driven development workflow. Write tests FIRST, then implement minimal code to pass. Ensure 80%+ coverage.
---

# TDD Command

Invokes the **tdd-guide** agent for test-driven development.

## TDD Cycle

```
RED -> GREEN -> REFACTOR -> REPEAT

RED:      Write a failing test (pytest)
GREEN:    Write minimal code to pass
REFACTOR: Improve code, keep tests passing
REPEAT:   Next feature/scenario
```

## When to Use

- Implementing new features
- Fixing bugs (write test that reproduces bug first)
- Refactoring existing code
- Building critical business logic

## Coverage Requirements

- **80% minimum** for all code
- **100% required** for financial calculations, auth, security-critical code

## Integration

- Use `/plan` first to understand what to build
- Use `/tdd` to implement with tests
- Use `/build-fix` if build errors occur
- Use `/code-review` to review implementation
