# Refactor Clean

Safely identify and remove dead code with test verification at every step.

## Step 1: Detect Dead Code

```bash
vulture src/                        # Unused Python code
ruff check . --select F401,F811    # Unused imports
pip-compile --dry-run              # Check dependency usage
```

## Step 2: Categorize Findings

| Tier | Examples | Action |
|------|----------|--------|
| **SAFE** | Unused utilities, test helpers, internal functions | Delete with confidence |
| **CAUTION** | API endpoints, middleware, signal handlers | Verify no dynamic imports |
| **DANGER** | Config files, entry points, type stubs | Investigate before touching |

## Step 3: Safe Deletion Loop

1. Run `pytest` -- establish baseline (all green)
2. Delete the dead code
3. Re-run `pytest` -- verify nothing broke
4. If tests fail -- revert with `git checkout -- <file>`
5. If tests pass -- commit and move to next

## Step 4: Consolidate Duplicates

- Near-duplicate functions (>80% similar) -- merge
- Redundant type definitions -- consolidate
- Wrapper functions that add no value -- inline

## Rules

- **Never delete without running tests first**
- **One deletion at a time**
- **Skip if uncertain**
- **Don't refactor while cleaning**
