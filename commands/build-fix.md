# Build and Fix

Incrementally fix build and type errors with minimal, safe changes.

## Step 1: Detect Build System

| Indicator | Build Command |
|-----------|---------------|
| `pyproject.toml` | `python -m compileall -q .` or `mypy .` |
| `setup.py` / `setup.cfg` | `python setup.py check` |
| `Makefile` | `make build` or `make check` |
| `tox.ini` | `tox` |

## Step 2: Parse and Group Errors

1. Run `mypy .` and `ruff check .` to capture all errors
2. Group errors by file path
3. Sort by dependency order (fix imports/types before logic errors)

## Step 3: Fix Loop (One Error at a Time)

1. **Read the file** -- See error context
2. **Diagnose** -- Missing import, wrong type, syntax error
3. **Fix minimally** -- Smallest change that resolves the error
4. **Re-run checks** -- Verify error is gone
5. **Move to next** -- Continue with remaining errors

## Step 4: Guardrails

Stop and ask the user if:
- A fix introduces more errors than it resolves
- The same error persists after 3 attempts
- The fix requires architectural changes
- Build errors stem from missing dependencies

Fix one error at a time for safety. Prefer minimal diffs over refactoring.
