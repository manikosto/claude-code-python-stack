---
name: celery-patterns
description: Celery patterns for distributed task queues — task definitions, retry strategies, scheduling, chains/groups, monitoring, and production configuration with Redis/RabbitMQ.
origin: custom
---

# Celery Patterns

Distributed task queue patterns for Python applications.

## When to Activate

- Offloading long-running tasks (email, PDF generation, data processing)
- Scheduling periodic tasks (cron-like)
- Building data processing pipelines
- Implementing retry logic for external API calls
- Setting up task monitoring and alerting

## Setup

### Configuration

```python
# config/celery.py
from celery import Celery
from app.config import settings

celery_app = Celery(
    "worker",
    broker=settings.REDIS_URL,
    backend=settings.REDIS_URL,
)

celery_app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="UTC",
    enable_utc=True,
    task_track_started=True,
    task_time_limit=300,           # Hard limit: 5 minutes
    task_soft_time_limit=240,      # Soft limit: 4 minutes
    worker_prefetch_multiplier=1,  # Disable prefetching for fair scheduling
    worker_max_tasks_per_child=1000,  # Restart worker after 1000 tasks (memory leak protection)
    task_acks_late=True,           # Acknowledge after execution (at-least-once)
    task_reject_on_worker_lost=True,
    result_expires=3600,           # Results expire after 1 hour
)

# Auto-discover tasks in all apps
celery_app.autodiscover_tasks(["app.tasks"])
```

### Django Integration

```python
# config/celery.py (Django)
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.production")

app = Celery("myproject")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()

# config/__init__.py
from .celery import app as celery_app
__all__ = ("celery_app",)
```

## Task Definitions

### Basic Task

```python
from celery import shared_task

@shared_task
def send_email(to: str, subject: str, body: str) -> dict:
    """Send email task."""
    result = email_service.send(to=to, subject=subject, body=body)
    return {"status": "sent", "message_id": result.id}

# Call the task
send_email.delay("user@example.com", "Welcome!", "Hello there")

# Or with more control
send_email.apply_async(
    args=["user@example.com", "Welcome!", "Hello there"],
    countdown=60,          # Delay execution by 60 seconds
    expires=3600,          # Task expires after 1 hour if not started
    queue="emails",        # Route to specific queue
)
```

### Task with Retry

```python
@shared_task(
    bind=True,
    max_retries=3,
    default_retry_delay=60,  # 60 seconds between retries
    autoretry_for=(ConnectionError, TimeoutError),
    retry_backoff=True,      # Exponential backoff
    retry_backoff_max=600,   # Max 10 minutes between retries
    retry_jitter=True,       # Add random jitter
)
def call_external_api(self, url: str, payload: dict) -> dict:
    """Call external API with automatic retry on failure."""
    try:
        response = httpx.post(url, json=payload, timeout=30)
        response.raise_for_status()
        return response.json()
    except httpx.HTTPStatusError as exc:
        if exc.response.status_code >= 500:
            raise self.retry(exc=exc)
        raise  # Don't retry 4xx errors
```

### Task with Custom Retry

```python
@shared_task(bind=True, max_retries=5)
def process_payment(self, order_id: int) -> dict:
    """Process payment with custom retry logic."""
    try:
        order = Order.objects.get(id=order_id)
        result = payment_gateway.charge(order)
        order.status = "paid"
        order.save()
        return {"order_id": order_id, "status": "paid"}
    except PaymentDeclinedError:
        order.status = "declined"
        order.save()
        return {"order_id": order_id, "status": "declined"}
    except PaymentGatewayError as exc:
        raise self.retry(
            exc=exc,
            countdown=2 ** self.request.retries * 30,  # 30s, 60s, 120s, 240s, 480s
        )
```

### Task with Progress Tracking

```python
@shared_task(bind=True)
def process_large_file(self, file_path: str) -> dict:
    """Process file with progress updates."""
    total_lines = count_lines(file_path)
    processed = 0

    with open(file_path) as f:
        for line in f:
            process_line(line)
            processed += 1

            if processed % 1000 == 0:
                self.update_state(
                    state="PROGRESS",
                    meta={"current": processed, "total": total_lines, "percent": processed / total_lines * 100},
                )

    return {"processed": processed, "total": total_lines}

# Check progress
result = process_large_file.AsyncResult(task_id)
if result.state == "PROGRESS":
    print(f"Progress: {result.info['percent']:.1f}%")
```

## Task Chains and Groups

### Chain (Sequential)

```python
from celery import chain

# Tasks execute sequentially, result passes to next
workflow = chain(
    fetch_data.s(url),
    transform_data.s(),
    save_to_database.s(),
    send_notification.s(),
)
result = workflow.apply_async()
```

### Group (Parallel)

```python
from celery import group

# Tasks execute in parallel
job = group([
    process_chunk.s(chunk) for chunk in data_chunks
])
result = job.apply_async()

# Wait for all results
results = result.get(timeout=300)
```

### Chord (Group + Callback)

```python
from celery import chord

# Run group in parallel, then call callback with results
workflow = chord(
    [process_chunk.s(chunk) for chunk in data_chunks],
    aggregate_results.s(),
)
result = workflow.apply_async()
```

### Canvas Example

```python
from celery import chain, group, chord

# Complex workflow
workflow = chain(
    fetch_raw_data.s(source_id),
    group([
        process_text.s(),
        process_images.s(),
        process_metadata.s(),
    ]),
    merge_results.s(),
    save_and_notify.s(user_id),
)
```

## Periodic Tasks (Beat)

```python
from celery.schedules import crontab

celery_app.conf.beat_schedule = {
    "cleanup-expired-sessions": {
        "task": "app.tasks.cleanup_expired_sessions",
        "schedule": crontab(hour=2, minute=0),  # Daily at 2 AM
    },
    "send-daily-report": {
        "task": "app.tasks.send_daily_report",
        "schedule": crontab(hour=8, minute=0, day_of_week="mon-fri"),
    },
    "check-payment-status": {
        "task": "app.tasks.check_pending_payments",
        "schedule": 300.0,  # Every 5 minutes
    },
    "monthly-invoice": {
        "task": "app.tasks.generate_invoices",
        "schedule": crontab(day_of_month=1, hour=0, minute=0),
    },
}
```

## Task Routing

```python
celery_app.conf.task_routes = {
    "app.tasks.send_email": {"queue": "emails"},
    "app.tasks.process_payment": {"queue": "payments", "priority": 9},
    "app.tasks.generate_report": {"queue": "reports"},
    "app.tasks.*": {"queue": "default"},
}

# Start workers for specific queues
# celery -A config worker -Q emails -c 4
# celery -A config worker -Q payments -c 2
# celery -A config worker -Q default,reports -c 8
```

## Error Handling

```python
from celery import shared_task
from celery.exceptions import SoftTimeLimitExceeded

@shared_task(bind=True, soft_time_limit=120, time_limit=150)
def long_running_task(self, data: dict) -> dict:
    try:
        result = process(data)
        return result
    except SoftTimeLimitExceeded:
        # Graceful shutdown: save progress, cleanup
        save_partial_progress(self.request.id, data)
        return {"status": "timeout", "partial": True}

# Global error handler
@celery_app.task(bind=True)
def on_task_failure(self, exc, task_id, args, kwargs, einfo):
    """Handle task failures globally."""
    logger.error(f"Task {task_id} failed: {exc}", exc_info=einfo)
    notify_ops_team(task_id, str(exc))
```

## Testing

```python
import pytest
from unittest.mock import patch

# Test task logic directly (no Celery overhead)
def test_send_email():
    with patch("app.tasks.email_service.send") as mock_send:
        mock_send.return_value.id = "msg-123"
        result = send_email("test@example.com", "Subject", "Body")
        assert result["status"] == "sent"
        mock_send.assert_called_once()

# Test with eager mode
@pytest.fixture(autouse=True)
def celery_eager(settings):
    settings.CELERY_TASK_ALWAYS_EAGER = True
    settings.CELERY_TASK_EAGER_PROPAGATES = True

def test_task_chain():
    result = chain(fetch_data.s("url"), transform_data.s()).apply()
    assert result.successful()
```

## Monitoring

```bash
# Flower (web UI)
pip install flower
celery -A config flower --port=5555

# CLI monitoring
celery -A config inspect active          # Active tasks
celery -A config inspect reserved        # Queued tasks
celery -A config inspect stats           # Worker stats
celery -A config inspect ping            # Worker health
celery -A config purge                   # Clear queues (DANGEROUS)
```

## Production Commands

```bash
# Start worker
celery -A config worker --loglevel=info --concurrency=8

# Start beat scheduler
celery -A config beat --loglevel=info

# Combined (dev only)
celery -A config worker --beat --loglevel=info

# With supervisor (production)
# /etc/supervisor/conf.d/celery.conf
[program:celery-worker]
command=celery -A config worker -l info -c 8
directory=/app
user=appuser
autostart=true
autorestart=true
stopwaitsecs=600
```

## Quick Reference

| Pattern | Usage |
|---------|-------|
| `task.delay(*args)` | Fire and forget |
| `task.apply_async(kwargs=...)` | Full control |
| `chain(a.s(), b.s())` | Sequential execution |
| `group([a.s(x) for x in items])` | Parallel execution |
| `chord(group, callback)` | Parallel + aggregate |
| `bind=True` | Access `self` for retries |
| `autoretry_for=(Error,)` | Auto-retry on exceptions |
| `retry_backoff=True` | Exponential backoff |
| `self.update_state()` | Progress tracking |
| `beat_schedule` | Periodic tasks |

**Remember**: Tasks should be idempotent (safe to run multiple times with same input). Use `task_acks_late=True` for at-least-once delivery. Never pass large objects as task arguments -- pass IDs and fetch from DB.
