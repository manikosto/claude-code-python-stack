---
name: redis-patterns
description: Redis patterns for Python — caching, sessions, pub/sub, rate limiting, distributed locks, and integration with Django/FastAPI.
origin: custom
---

# Redis Patterns for Python

Redis usage patterns for caching, sessions, queues, and distributed systems.

## When to Activate

- Implementing caching layer
- Setting up session storage
- Building pub/sub messaging
- Implementing rate limiting
- Using distributed locks

## Connection Setup

### redis-py (Sync)

```python
import redis

pool = redis.ConnectionPool(
    host="localhost", port=6379, db=0,
    max_connections=20,
    decode_responses=True,
)
r = redis.Redis(connection_pool=pool)
```

### redis-py (Async)

```python
import redis.asyncio as aioredis

pool = aioredis.ConnectionPool.from_url(
    "redis://localhost:6379/0",
    max_connections=20,
    decode_responses=True,
)
redis_client = aioredis.Redis(connection_pool=pool)

# Usage
async def get_cached(key: str) -> str | None:
    return await redis_client.get(key)
```

## Caching Patterns

### Simple Cache

```python
import json
from datetime import timedelta

async def get_user(user_id: int) -> dict:
    cache_key = f"user:{user_id}"

    # Try cache first
    cached = await redis_client.get(cache_key)
    if cached:
        return json.loads(cached)

    # Fetch from DB
    user = await db.get_user(user_id)
    if user:
        await redis_client.setex(cache_key, timedelta(minutes=15), json.dumps(user))

    return user
```

### Cache Invalidation

```python
async def update_user(user_id: int, data: dict) -> dict:
    user = await db.update_user(user_id, data)

    # Invalidate cache
    await redis_client.delete(f"user:{user_id}")

    # Invalidate related caches
    await redis_client.delete(f"user_list:page:*")

    return user
```

### Cache-Aside with Decorator

```python
import functools
import hashlib

def cached(ttl: int = 300, prefix: str = "cache"):
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            key_data = f"{func.__name__}:{args}:{sorted(kwargs.items())}"
            cache_key = f"{prefix}:{hashlib.md5(key_data.encode()).hexdigest()}"

            cached_result = await redis_client.get(cache_key)
            if cached_result:
                return json.loads(cached_result)

            result = await func(*args, **kwargs)
            await redis_client.setex(cache_key, ttl, json.dumps(result, default=str))
            return result
        return wrapper
    return decorator

@cached(ttl=600, prefix="products")
async def get_products(category: str, page: int = 1) -> list[dict]:
    return await db.query_products(category=category, page=page)
```

## Rate Limiting

### Sliding Window

```python
import time

async def is_rate_limited(user_id: str, limit: int = 100, window: int = 60) -> bool:
    key = f"rate_limit:{user_id}"
    now = time.time()
    pipe = redis_client.pipeline()

    pipe.zremrangebyscore(key, 0, now - window)  # Remove old entries
    pipe.zadd(key, {str(now): now})              # Add current request
    pipe.zcard(key)                               # Count requests in window
    pipe.expire(key, window)                      # Set expiry

    results = await pipe.execute()
    request_count = results[2]

    return request_count > limit
```

### Token Bucket

```python
async def check_rate_limit(key: str, max_tokens: int = 10, refill_rate: float = 1.0) -> bool:
    """Token bucket rate limiter. Returns True if request is allowed."""
    now = time.time()
    pipe = redis_client.pipeline()

    pipe.hgetall(key)
    result = await pipe.execute()
    data = result[0]

    tokens = float(data.get("tokens", max_tokens))
    last_refill = float(data.get("last_refill", now))

    # Refill tokens
    elapsed = now - last_refill
    tokens = min(max_tokens, tokens + elapsed * refill_rate)

    if tokens >= 1:
        tokens -= 1
        await redis_client.hset(key, mapping={"tokens": tokens, "last_refill": now})
        await redis_client.expire(key, int(max_tokens / refill_rate) + 1)
        return True

    return False
```

## Distributed Lock

```python
import uuid

class RedisLock:
    def __init__(self, redis_client, name: str, timeout: int = 10):
        self.redis = redis_client
        self.name = f"lock:{name}"
        self.timeout = timeout
        self.token = str(uuid.uuid4())

    async def __aenter__(self):
        acquired = await self.redis.set(self.name, self.token, nx=True, ex=self.timeout)
        if not acquired:
            raise RuntimeError(f"Could not acquire lock: {self.name}")
        return self

    async def __aexit__(self, *args):
        # Only release if we own the lock
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        await self.redis.eval(lua_script, 1, self.name, self.token)

# Usage
async def process_order(order_id: int):
    async with RedisLock(redis_client, f"order:{order_id}", timeout=30):
        # Only one worker processes this order at a time
        await do_process(order_id)
```

## Pub/Sub

```python
# Publisher
async def publish_event(channel: str, event: dict):
    await redis_client.publish(channel, json.dumps(event))

# Subscriber
async def subscribe_events(channel: str):
    pubsub = redis_client.pubsub()
    await pubsub.subscribe(channel)

    async for message in pubsub.listen():
        if message["type"] == "message":
            event = json.loads(message["data"])
            await handle_event(event)
```

## Django Integration

```python
# settings.py
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": "redis://localhost:6379/0",
        "OPTIONS": {
            "pool_class": "redis.BlockingConnectionPool",
            "max_connections": 50,
        },
    }
}

SESSION_ENGINE = "django.contrib.sessions.backends.cache"
SESSION_CACHE_ALIAS = "default"
```

## Quick Reference

| Pattern | Use Case |
|---------|----------|
| `GET/SET/SETEX` | Simple key-value caching |
| `HSET/HGETALL` | Structured data (hash maps) |
| `ZADD/ZRANGEBYSCORE` | Rate limiting (sorted sets) |
| `SET NX EX` | Distributed locks |
| `PUBLISH/SUBSCRIBE` | Event broadcasting |
| `LPUSH/BRPOP` | Task queues |
| `Pipeline` | Batch operations |
| `Lua scripts` | Atomic operations |
