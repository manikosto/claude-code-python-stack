---
name: async-http-patterns
description: Async HTTP client patterns for Python — httpx, aiohttp, retry strategies, connection pooling, streaming, and testing with respx.
origin: custom
---

# Async HTTP Client Patterns

Patterns for making HTTP requests in async Python applications.

## When to Activate

- Making HTTP requests to external APIs
- Implementing API clients/wrappers
- Handling retries and timeouts
- Streaming large responses
- Testing HTTP interactions

## httpx (Recommended)

### Basic Usage

```python
import httpx

# Sync
response = httpx.get("https://api.example.com/users")
response.raise_for_status()
data = response.json()

# Async
async with httpx.AsyncClient() as client:
    response = await client.get("https://api.example.com/users")
    response.raise_for_status()
    data = response.json()
```

### Reusable Client with Connection Pooling

```python
import httpx
from contextlib import asynccontextmanager

class APIClient:
    def __init__(self, base_url: str, api_key: str):
        self.client = httpx.AsyncClient(
            base_url=base_url,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=httpx.Timeout(30.0, connect=5.0),
            limits=httpx.Limits(max_connections=100, max_keepalive_connections=20),
        )

    async def get_user(self, user_id: int) -> dict:
        response = await self.client.get(f"/users/{user_id}")
        response.raise_for_status()
        return response.json()

    async def create_user(self, data: dict) -> dict:
        response = await self.client.post("/users", json=data)
        response.raise_for_status()
        return response.json()

    async def close(self):
        await self.client.aclose()
```

### Retry with Exponential Backoff

```python
import asyncio
import httpx
from typing import TypeVar

T = TypeVar("T")

async def retry_request(
    client: httpx.AsyncClient,
    method: str,
    url: str,
    max_retries: int = 3,
    base_delay: float = 1.0,
    **kwargs,
) -> httpx.Response:
    last_exc = None
    for attempt in range(max_retries + 1):
        try:
            response = await client.request(method, url, **kwargs)
            if response.status_code < 500:
                return response
            # Retry on 5xx
            last_exc = httpx.HTTPStatusError(
                f"Server error: {response.status_code}",
                request=response.request,
                response=response,
            )
        except (httpx.ConnectError, httpx.ReadTimeout) as exc:
            last_exc = exc

        if attempt < max_retries:
            delay = base_delay * (2 ** attempt)
            await asyncio.sleep(delay)

    raise last_exc
```

### Streaming Large Responses

```python
async def download_file(url: str, output_path: str):
    async with httpx.AsyncClient() as client:
        async with client.stream("GET", url) as response:
            response.raise_for_status()
            with open(output_path, "wb") as f:
                async for chunk in response.aiter_bytes(chunk_size=8192):
                    f.write(chunk)
```

### Concurrent Requests

```python
import asyncio
import httpx

async def fetch_all(urls: list[str]) -> list[dict]:
    async with httpx.AsyncClient() as client:
        tasks = [client.get(url) for url in urls]
        responses = await asyncio.gather(*tasks, return_exceptions=True)

        results = []
        for url, response in zip(urls, responses):
            if isinstance(response, Exception):
                results.append({"url": url, "error": str(response)})
            else:
                results.append({"url": url, "data": response.json()})
        return results
```

### Rate-Limited Client

```python
import asyncio

class RateLimitedClient:
    def __init__(self, client: httpx.AsyncClient, requests_per_second: float = 10):
        self.client = client
        self.semaphore = asyncio.Semaphore(int(requests_per_second))
        self.delay = 1.0 / requests_per_second

    async def get(self, url: str, **kwargs) -> httpx.Response:
        async with self.semaphore:
            response = await self.client.get(url, **kwargs)
            await asyncio.sleep(self.delay)
            return response
```

## aiohttp

```python
import aiohttp

async def fetch_with_aiohttp(url: str) -> dict:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            response.raise_for_status()
            return await response.json()

# With connection pooling
connector = aiohttp.TCPConnector(limit=100, limit_per_host=20)
session = aiohttp.ClientSession(connector=connector)
```

## Testing with respx

```python
import respx
import httpx
import pytest

@respx.mock
@pytest.mark.asyncio
async def test_get_user():
    respx.get("https://api.example.com/users/1").mock(
        return_value=httpx.Response(200, json={"id": 1, "name": "Alice"})
    )

    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/users/1")
        assert response.json()["name"] == "Alice"

@respx.mock
@pytest.mark.asyncio
async def test_api_error():
    respx.get("https://api.example.com/users/999").mock(
        return_value=httpx.Response(404, json={"error": "Not found"})
    )

    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/users/999")
        assert response.status_code == 404
```

## Quick Reference

| Feature | httpx | aiohttp |
|---------|-------|---------|
| Sync + Async | Yes | Async only |
| HTTP/2 | Yes | No |
| Connection pooling | Built-in | Built-in |
| Streaming | `aiter_bytes()` | `content.read()` |
| Testing | respx | aioresponses |
| FastAPI testing | AsyncClient + ASGITransport | N/A |
