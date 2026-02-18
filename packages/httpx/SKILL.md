---
name: httpx
description: Modern HTTP client for Python with sync and async support. Use when making HTTP requests, configuring clients with connection pooling, handling authentication, streaming responses, or migrating from requests library. Triggers on HTTP client, httpx, API requests, async HTTP, REST calls, HTTP/2.
---

# httpx — Modern HTTP Client (v0.28.1)

## Quick Start

```bash
pip install httpx
pip install "httpx[http2]"  # for HTTP/2 support
```

```python
import httpx

r = httpx.get("https://api.example.com/users")
r.status_code   # 200
r.json()         # parsed JSON
r.raise_for_status()  # raises on 4xx/5xx
```

## Key Patterns

### Client with connection pooling (always use for multiple requests)
```python
with httpx.Client(base_url="https://api.example.com", follow_redirects=True) as client:
    r = client.get("/users", params={"page": 1})
    r = client.post("/users", json={"name": "Alice"})
```

### Async client
```python
async with httpx.AsyncClient() as client:
    r = await client.get("https://api.example.com/users")
```

### Testing with MockTransport
```python
def handler(request):
    return httpx.Response(200, json={"ok": True})

client = httpx.Client(transport=httpx.MockTransport(handler))
```

## References

- **[api.md](references/api.md)** — Client/AsyncClient, request methods, headers, params, timeouts, response
- **[auth-and-transport.md](references/auth-and-transport.md)** — Authentication, transport/proxies, HTTP/2, SSL, cookies, event hooks
- **[testing.md](references/testing.md)** — MockTransport, ASGI/WSGI transport, response mocking
- **[examples.md](references/examples.md)** — Complete examples, gotchas, migration from requests

## Grep Patterns

- `httpx\.Client|httpx\.AsyncClient` — Find client instances
- `httpx\.get|httpx\.post` — Find one-off requests
- `MockTransport|ASGITransport` — Find test transport usage
