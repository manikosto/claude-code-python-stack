# Code Review

Comprehensive security and quality review of uncommitted changes:

1. Get changed files: `git diff --name-only HEAD`

2. For each changed file, check for:

**Security Issues (CRITICAL):**
- Hardcoded credentials, API keys, tokens
- SQL injection (f-strings in queries)
- Missing input validation
- Unsafe eval/exec/pickle
- Path traversal risks

**Code Quality (HIGH):**
- Functions > 50 lines
- Files > 800 lines
- Nesting depth > 4 levels
- Missing error handling (bare except)
- print() statements (use logging)
- Missing type hints on public functions
- Mutable default arguments

**Best Practices (MEDIUM):**
- Missing tests for new code
- PEP 8 violations
- `value == None` instead of `value is None`
- `type() ==` instead of `isinstance()`

3. Generate report with severity, file location, issue, and fix suggestion

4. Block commit if CRITICAL or HIGH issues found
