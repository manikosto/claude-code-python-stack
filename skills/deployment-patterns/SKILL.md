---
name: deployment-patterns
description: Deployment workflows, CI/CD pipeline patterns, Docker containerization, health checks, rollback strategies for Python applications.
origin: ECC
---

# Deployment Patterns for Python

Production deployment workflows and CI/CD best practices.

## When to Activate

- Setting up CI/CD pipelines for Python projects
- Dockerizing Python/Django/FastAPI applications
- Planning deployment strategy
- Implementing health checks and readiness probes

## Deployment Strategies

| Strategy | Use When | Pros | Cons |
|----------|----------|------|------|
| Rolling | Standard deploys | Zero downtime | Two versions run simultaneously |
| Blue-Green | Critical services | Instant rollback | 2x infrastructure |
| Canary | Risky changes | Catches issues early | Requires traffic splitting |

## Python Dockerfile

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
RUN pip install --no-cache-dir uv
COPY requirements.txt .
RUN uv pip install --system --no-cache -r requirements.txt

FROM python:3.12-slim AS runner
WORKDIR /app
RUN useradd -r -u 1001 appuser
USER appuser
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY . .
ENV PYTHONUNBUFFERED=1
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/')" || exit 1
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

## GitHub Actions (Python)

```yaml
name: CI/CD
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
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
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: ruff check . && black . --check && mypy .
      - run: pytest --cov=src --cov-report=xml
      - run: bandit -r src/ && pip-audit

  build:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

## Health Check Endpoint (FastAPI)

```python
from fastapi import FastAPI
from datetime import datetime

app = FastAPI()

@app.get("/health")
async def health():
    return {"status": "ok", "timestamp": datetime.utcnow().isoformat()}

@app.get("/health/detailed")
async def health_detailed():
    checks = {
        "database": await check_database(),
        "redis": await check_redis(),
    }
    all_healthy = all(c["status"] == "ok" for c in checks.values())
    return {"status": "ok" if all_healthy else "degraded", "checks": checks}
```

## Production Readiness Checklist

- [ ] All tests pass (unit, integration, E2E)
- [ ] No hardcoded secrets
- [ ] Health check endpoint works
- [ ] Docker image builds reproducibly
- [ ] Environment variables validated at startup
- [ ] Resource limits set
- [ ] SSL/TLS enabled
- [ ] Alerts configured
- [ ] Log aggregation set up
- [ ] Rollback plan documented
- [ ] Dependencies scanned for CVEs
