# Verification Command

Run comprehensive verification on current Python codebase state.

## Instructions

Execute verification in this exact order:

1. **Type Check**
   - Run `mypy .`
   - Report all errors with file:line

2. **Lint Check**
   - Run `ruff check .`
   - Run `black . --check`
   - Report warnings and errors

3. **Test Suite**
   - Run `pytest --cov=src --cov-report=term-missing`
   - Report pass/fail count
   - Report coverage percentage

4. **Security Scan**
   - Run `bandit -r src/`
   - Run `pip-audit`
   - Report vulnerabilities

5. **Debug Statement Audit**
   - Search for `print(`, `pdb`, `breakpoint()` in source files
   - Report locations

6. **Git Status**
   - Show uncommitted changes
   - Show files modified since last commit

## Output

```
VERIFICATION: [PASS/FAIL]

Types:    [OK/X errors]
Lint:     [OK/X issues]
Tests:    [X/Y passed, Z% coverage]
Security: [OK/X vulnerabilities]
Debug:    [OK/X print/pdb found]

Ready for PR: [YES/NO]
```

## Arguments

$ARGUMENTS can be:
- `quick` -- Only types + lint
- `full` -- All checks (default)
- `pre-commit` -- Types + lint + debug audit
- `pre-pr` -- Full checks plus security scan
