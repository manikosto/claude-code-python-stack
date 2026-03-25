---
name: django-verification
description: "Verification loop for Django projects: migrations, linting, tests with coverage, security scans, and deployment readiness checks."
origin: ECC
---

# Django Verification Loop

Run before PRs, after major changes, and pre-deploy.

## When to Activate

- Before opening a pull request for a Django project
- After major model changes, migration updates, or dependency upgrades
- Pre-deployment verification

## Verification Pipeline

```bash
# Phase 1: Code Quality
mypy . --config-file pyproject.toml
ruff check . --fix
black . --check
isort . --check-only
python manage.py check --deploy

# Phase 2: Migrations
python manage.py showmigrations
python manage.py makemigrations --check
python manage.py migrate --plan

# Phase 3: Tests + Coverage
pytest --cov=apps --cov-report=html --cov-report=term-missing --reuse-db

# Phase 4: Security
pip-audit
bandit -r . -f json -o bandit-report.json
python manage.py check --deploy

# Phase 5: Django Commands
python manage.py check
python manage.py collectstatic --noinput --clear
```

## Pre-Deployment Checklist

- [ ] All tests passing
- [ ] Coverage >= 80%
- [ ] No security vulnerabilities
- [ ] No unapplied migrations
- [ ] DEBUG = False in production settings
- [ ] SECRET_KEY properly configured
- [ ] ALLOWED_HOSTS set correctly
- [ ] Static files collected
- [ ] Logging configured
- [ ] HTTPS/SSL configured

## GitHub Actions Example

```yaml
name: Django Verification
on: [push, pull_request]
jobs:
  verify:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install -r requirements.txt
      - run: ruff check . && black . --check && mypy .
      - run: bandit -r . && pip-audit
      - env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test
          DJANGO_SECRET_KEY: test-secret-key
        run: pytest --cov=apps --cov-report=xml
```
