# httpx

A modern, fully-featured HTTP client for Python 3, supporting both **synchronous** and **asynchronous** APIs, with **HTTP/1.1** and **HTTP/2** support.

`httpx` is a next-generation HTTP client designed as a drop-in replacement for `requests`, while adding async support, HTTP/2, and stricter defaults. It is built by [Encode](https://www.encode.io/) (the team behind Starlette and Uvicorn) and is the recommended HTTP client for modern Python applications.

---

## Latest Stable Version

| Detail | Value |
|--------|-------|
| **Package** | `httpx` |
| **Latest version** | 0.28.1 (verify at PyPI for newer releases) |
| **Python support** | Python >= 3.8 |
| **License** | BSD 3-Clause |
| **Repository** | https://github.com/encode/httpx |
| **Documentation** | https://www.python-httpx.org |

---

## Table of Contents

1. [Installation](#installation)
2. [Core API](#core-api)
3. [Client Usage](#client-usage)
4. [Authentication](#authentication)
5. [Timeout Configuration](#timeout-configuration)
6. [Streaming](#streaming)
7. [HTTP/2 Support](#http2-support)
8. [Proxy Support](#proxy-support)
9. [Testing with MockTransport](#testing-with-mocktransport)
10. [Async Usage](#async-usage)
11. [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
12. [Migration from requests](#migration-from-requests)
13. [Complete Code Examples](#complete-code-examples)

---

## Installation

```bash
# Standard installation
pip install httpx

# With HTTP/2 support
pip install httpx[http2]

# With Brotli decompression support
pip install httpx[brotli]

# With SOCKS proxy support
pip install httpx[socks]

# All optional dependencies
pip install httpx[http2,brotli,socks]
```

**Dependencies:** `httpx` depends on `httpcore` (the low-level transport layer), `certifi`, `idna`, and `sniffio`. These are installed automatically.

---

## Core API

### Quick Functions (Top-Level API)

For one-off requests, `httpx` provides module-level convenience functions. Each creates a short-lived connection, sends the request, and returns a `Response`.

```python
import httpx

# GET request
response = httpx.get("https://httpbin.org/get")

# GET with query parameters
response = httpx.get("https://httpbin.org/get", params={"key": "value", "page": 2})

# POST with JSON body
response = httpx.post("https://httpbin.org/post", json={"username": "alice", "active": True})

# POST with form data
response = httpx.post("https://httpbin.org/post", data={"username": "alice"})

# PUT
response = httpx.put("https://httpbin.org/put", json={"id": 1, "name": "updated"})

# PATCH
response = httpx.patch("https://httpbin.org/patch", json={"name": "patched"})

# DELETE
response = httpx.delete("https://httpbin.org/delete")

# HEAD (response body is empty; only headers returned)
response = httpx.head("https://httpbin.org/get")

# OPTIONS
response = httpx.options("https://httpbin.org/get")
```

All top-level functions accept the same keyword arguments:

| Parameter | Type | Description |
|-----------|------|-------------|
| `params` | `dict`, `list[tuple]`, or `str` | URL query parameters |
| `headers` | `dict` | HTTP headers |
| `cookies` | `dict` or `Cookies` | Request cookies |
| `content` | `bytes`, `str`, or iterable | Raw request body |
| `data` | `dict` | Form-encoded request body |
| `files` | `dict` | Multipart file uploads |
| `json` | Any JSON-serializable | JSON-encoded request body |
| `timeout` | `float`, `Timeout`, or `None` | Request timeout (default: 5 seconds) |
| `auth` | `tuple`, `Auth`, or callable | Authentication credentials |
| `follow_redirects` | `bool` | Whether to follow redirects (default: `False`) |
| `verify` | `bool` or `str` | SSL verification (default: `True`) |

### The Response Object

Every request returns an `httpx.Response` object with a rich interface:

```python
import httpx

response = httpx.get("https://httpbin.org/get")

# Status
response.status_code          # 200
response.reason_phrase         # 'OK'
response.is_success            # True (2xx status)
response.is_redirect           # False
response.is_client_error       # False (4xx status)
response.is_server_error       # False (5xx status)

# Body
response.text                  # Response body as decoded string
response.content               # Response body as raw bytes
response.json()                # Parse body as JSON -> dict/list
response.read()                # Read and return body bytes (for streaming)

# Headers
response.headers               # Headers (case-insensitive dict-like)
response.headers["content-type"]  # 'application/json'

# URL (may differ from request URL after redirects)
response.url                   # URL('https://httpbin.org/get')

# Cookies
response.cookies               # Cookies set by the response

# Timing
response.elapsed               # datetime.timedelta of request duration

# Encoding
response.encoding              # Detected or declared encoding (e.g. 'utf-8')

# HTTP version
response.http_version          # 'HTTP/1.1' or 'HTTP/2'

# The original request
response.request               # The httpx.Request that generated this response

# Raise an exception for 4xx/5xx responses
response.raise_for_status()    # Raises httpx.HTTPStatusError if not 2xx
```

### Status Code Checking

```python
import httpx

response = httpx.get("https://httpbin.org/status/404")

# Boolean check
if response.status_code == 200:
    print("OK")

# Category checks
response.is_success        # True for 2xx
response.is_redirect       # True for 3xx
response.is_client_error   # True for 4xx
response.is_server_error   # True for 5xx

# Raise on error status (recommended pattern)
try:
    response.raise_for_status()
except httpx.HTTPStatusError as e:
    print(f"Error: {e.response.status_code} for URL {e.request.url}")
```

---

## Client Usage

### Why Use a Client?

The top-level functions (`httpx.get()`, etc.) create a new connection for every request. For applications making multiple requests, use `httpx.Client` (sync) or `httpx.AsyncClient` (async) to get:

- **Connection pooling** -- reuses TCP connections across requests
- **Keep-alive** -- holds connections open for subsequent requests
- **Cookie persistence** -- cookies flow between requests automatically
- **Shared configuration** -- set base URL, headers, auth, timeouts once

### httpx.Client (Synchronous)

```python
import httpx

# Always use as a context manager to ensure connections are properly closed
with httpx.Client() as client:
    r1 = client.get("https://httpbin.org/get")
    r2 = client.post("https://httpbin.org/post", json={"key": "value"})
    r3 = client.get("https://httpbin.org/get")
    # All three requests reuse the same connection pool
```

### httpx.AsyncClient (Asynchronous)

```python
import httpx
import asyncio

async def main():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://httpbin.org/get")
        print(response.json())

asyncio.run(main())
```

### Base URL

Set a base URL to avoid repeating the domain for every request:

```python
with httpx.Client(base_url="https://api.example.com/v2") as client:
    # Requests are resolved relative to the base URL
    r = client.get("/users")          # -> https://api.example.com/v2/users
    r = client.get("/users/42")       # -> https://api.example.com/v2/users/42
    r = client.post("/users", json={  # -> https://api.example.com/v2/users
        "name": "Alice"
    })
```

### Default Headers, Params, and Auth

```python
with httpx.Client(
    base_url="https://api.example.com",
    headers={"User-Agent": "my-app/1.0", "Accept": "application/json"},
    params={"api_key": "secret123"},
    auth=("username", "password"),
    timeout=30.0,
) as client:
    # Every request inherits these defaults
    r = client.get("/data")
    # Equivalent to: GET https://api.example.com/data?api_key=secret123
    #   with headers User-Agent: my-app/1.0, Accept: application/json
    #   with basic auth

    # Per-request values merge with (or override) client defaults
    r = client.get("/data", params={"extra": "param"})
    # -> ?api_key=secret123&extra=param  (merged)

    r = client.get("/data", headers={"Accept": "text/plain"})
    # Accept header overridden for this request only
```

### Event Hooks

Event hooks let you attach callbacks to every request and/or response. They are useful for logging, metrics, injecting headers, or retrying.

```python
import httpx

def log_request(request: httpx.Request):
    print(f"-> {request.method} {request.url}")

def log_response(response: httpx.Response):
    print(f"<- {response.status_code} {response.url} ({response.elapsed.total_seconds():.2f}s)")

def raise_on_4xx_5xx(response: httpx.Response):
    response.raise_for_status()

with httpx.Client(
    event_hooks={
        "request": [log_request],
        "response": [log_response, raise_on_4xx_5xx],
    }
) as client:
    r = client.get("https://httpbin.org/get")
    # -> GET https://httpbin.org/get
    # <- 200 https://httpbin.org/get (0.25s)
```

**Async event hooks** work with `AsyncClient`:

```python
import httpx

async def log_request(request: httpx.Request):
    print(f"-> {request.method} {request.url}")

async def log_response(response: httpx.Response):
    print(f"<- {response.status_code}")

async with httpx.AsyncClient(
    event_hooks={"request": [log_request], "response": [log_response]}
) as client:
    await client.get("https://httpbin.org/get")
```

### Transport Customization

The transport layer controls how HTTP requests are physically sent. You can replace it entirely for testing, retries, or custom behavior.

```python
import httpx

# Use a custom transport with connection pool limits
transport = httpx.HTTPTransport(
    retries=3,                    # Retry on connection failures
    local_address="0.0.0.0",     # Bind to specific local address
)

with httpx.Client(transport=transport) as client:
    r = client.get("https://httpbin.org/get")
```

### Connection Pool Limits

```python
import httpx

# Control pool size
limits = httpx.Limits(
    max_connections=100,          # Max total connections in the pool
    max_keepalive_connections=20, # Max idle keep-alive connections
)

with httpx.Client(limits=limits) as client:
    r = client.get("https://httpbin.org/get")
```

---

## Authentication

### Basic Authentication

```python
import httpx

# Tuple shorthand for HTTP Basic Auth
response = httpx.get(
    "https://httpbin.org/basic-auth/user/pass",
    auth=("user", "pass"),
)

# Explicit BasicAuth object
auth = httpx.BasicAuth(username="user", password="pass")
response = httpx.get("https://httpbin.org/basic-auth/user/pass", auth=auth)

# On a client
with httpx.Client(auth=("user", "pass")) as client:
    r = client.get("https://httpbin.org/basic-auth/user/pass")
```

### Digest Authentication

```python
import httpx

auth = httpx.DigestAuth(username="user", password="pass")
response = httpx.get("https://httpbin.org/digest-auth/auth/user/pass", auth=auth)
```

### Bearer Token Authentication

`httpx` does not have a built-in `BearerAuth` class, but you can use headers directly or build a custom auth class:

```python
import httpx

# Simple approach: pass token as a header
response = httpx.get(
    "https://api.example.com/data",
    headers={"Authorization": "Bearer eyJhbGciOiJIUzI1NiIs..."},
)

# On a client
with httpx.Client(
    headers={"Authorization": "Bearer eyJhbGciOiJIUzI1NiIs..."}
) as client:
    r = client.get("https://api.example.com/data")
```

### Custom Auth Flows

For complex authentication (OAuth2, token refresh, request signing), subclass `httpx.Auth`:

```python
import httpx
from typing import Generator

class BearerAuth(httpx.Auth):
    """Custom auth that adds a Bearer token to every request."""

    def __init__(self, token: str):
        self.token = token

    def auth_flow(self, request: httpx.Request) -> Generator[httpx.Request, httpx.Response, None]:
        request.headers["Authorization"] = f"Bearer {self.token}"
        yield request


class TokenRefreshAuth(httpx.Auth):
    """Auth that automatically refreshes an expired token."""

    def __init__(self, access_token: str, refresh_token: str, token_url: str):
        self.access_token = access_token
        self.refresh_token = refresh_token
        self.token_url = token_url

    def auth_flow(self, request: httpx.Request) -> Generator[httpx.Request, httpx.Response, None]:
        # Try the request with the current token
        request.headers["Authorization"] = f"Bearer {self.access_token}"
        response = yield request

        # If we get a 401, refresh and retry
        if response.status_code == 401:
            # Build a token refresh request
            refresh_request = httpx.Request(
                "POST",
                self.token_url,
                data={"grant_type": "refresh_token", "refresh_token": self.refresh_token},
            )
            refresh_response = yield refresh_request

            # Parse the new token
            data = refresh_response.json()
            self.access_token = data["access_token"]
            self.refresh_token = data.get("refresh_token", self.refresh_token)

            # Retry the original request with the new token
            request.headers["Authorization"] = f"Bearer {self.access_token}"
            yield request


# Usage
auth = TokenRefreshAuth(
    access_token="expired-token",
    refresh_token="my-refresh-token",
    token_url="https://auth.example.com/token",
)
with httpx.Client(auth=auth) as client:
    r = client.get("https://api.example.com/protected")
```

---

## Timeout Configuration

`httpx` has a **default timeout of 5 seconds** (unlike `requests`, which has no default timeout). You can configure granular timeouts for different phases of the request.

### Simple Timeout

```python
import httpx

# Single value applies to all phases (connect, read, write, pool)
response = httpx.get("https://httpbin.org/delay/2", timeout=10.0)

# Disable timeout entirely (not recommended for production)
response = httpx.get("https://httpbin.org/delay/2", timeout=None)
```

### Granular Timeout with httpx.Timeout

```python
import httpx

timeout = httpx.Timeout(
    connect=5.0,    # Max time to establish a connection
    read=30.0,      # Max time to receive the response body
    write=10.0,     # Max time to send the request body
    pool=5.0,       # Max time to acquire a connection from the pool
)

# On a single request
response = httpx.get("https://httpbin.org/delay/2", timeout=timeout)

# On a client (applies to all requests)
with httpx.Client(timeout=timeout) as client:
    r = client.get("https://httpbin.org/delay/2")

# Override per request
with httpx.Client(timeout=timeout) as client:
    # This specific request gets a longer read timeout
    r = client.get(
        "https://httpbin.org/delay/25",
        timeout=httpx.Timeout(connect=5.0, read=60.0, write=10.0, pool=5.0),
    )
```

### Timeout Exceptions

```python
import httpx

try:
    response = httpx.get("https://httpbin.org/delay/10", timeout=2.0)
except httpx.ConnectTimeout:
    print("Connection establishment timed out")
except httpx.ReadTimeout:
    print("Reading the response timed out")
except httpx.WriteTimeout:
    print("Sending the request body timed out")
except httpx.PoolTimeout:
    print("Timed out waiting for a connection from the pool")
except httpx.TimeoutException:
    print("Some timeout occurred (base class for all timeout errors)")
```

---

## Streaming

### Streaming Responses

For large downloads, use streaming to avoid loading the entire response body into memory.

```python
import httpx

# Synchronous streaming
with httpx.stream("GET", "https://example.com/large-file.zip") as response:
    response.raise_for_status()
    total = int(response.headers.get("content-length", 0))
    downloaded = 0
    with open("large-file.zip", "wb") as f:
        for chunk in response.iter_bytes(chunk_size=8192):
            f.write(chunk)
            downloaded += len(chunk)
            if total:
                print(f"\rProgress: {downloaded / total:.1%}", end="")

# Stream text line-by-line
with httpx.stream("GET", "https://example.com/log.txt") as response:
    for line in response.iter_lines():
        print(line)

# Stream raw bytes without decoding
with httpx.stream("GET", "https://example.com/data.bin") as response:
    for chunk in response.iter_raw():
        process(chunk)
```

### Streaming with a Client

```python
import httpx

with httpx.Client() as client:
    with client.stream("GET", "https://example.com/large-file.zip") as response:
        for chunk in response.iter_bytes():
            process(chunk)
```

### Async Streaming

```python
import httpx
import asyncio

async def download_file(url: str, path: str):
    async with httpx.AsyncClient() as client:
        async with client.stream("GET", url) as response:
            response.raise_for_status()
            with open(path, "wb") as f:
                async for chunk in response.aiter_bytes(chunk_size=8192):
                    f.write(chunk)

asyncio.run(download_file("https://example.com/file.zip", "file.zip"))
```

### Streaming Request Bodies

You can also stream the request body for large uploads:

```python
import httpx

def upload_chunks():
    """Generator that yields chunks of data to upload."""
    for i in range(100):
        yield f"chunk-{i}\n".encode()

response = httpx.post("https://httpbin.org/post", content=upload_chunks())
```

---

## HTTP/2 Support

HTTP/2 provides multiplexing (multiple requests over a single connection), header compression, and server push. `httpx` supports HTTP/2 via the optional `h2` dependency.

### Setup

```bash
pip install httpx[http2]
```

### Usage

HTTP/2 must be explicitly enabled on the client:

```python
import httpx

# HTTP/2 enabled
with httpx.Client(http2=True) as client:
    response = client.get("https://httpbin.org/get")
    print(response.http_version)  # 'HTTP/2' if server supports it, else 'HTTP/1.1'

# Async HTTP/2
async with httpx.AsyncClient(http2=True) as client:
    response = await client.get("https://httpbin.org/get")
    print(response.http_version)
```

**Notes:**
- HTTP/2 is negotiated via ALPN during the TLS handshake. If the server does not support HTTP/2, `httpx` falls back to HTTP/1.1 automatically.
- HTTP/2 is only supported over HTTPS (not plain HTTP).
- The top-level functions (`httpx.get()`, etc.) do not support HTTP/2 since they do not use a persistent client.

---

## Proxy Support

### HTTP Proxies

```python
import httpx

# Single proxy for all traffic
with httpx.Client(proxy="http://proxy.example.com:8080") as client:
    r = client.get("https://httpbin.org/get")

# Proxy with authentication
with httpx.Client(proxy="http://user:pass@proxy.example.com:8080") as client:
    r = client.get("https://httpbin.org/get")
```

### SOCKS Proxies

SOCKS proxy support requires the `socksio` package:

```bash
pip install httpx[socks]
```

```python
import httpx

# SOCKS5 proxy
with httpx.Client(proxy="socks5://proxy.example.com:1080") as client:
    r = client.get("https://httpbin.org/get")

# SOCKS5 with authentication
with httpx.Client(proxy="socks5://user:pass@proxy.example.com:1080") as client:
    r = client.get("https://httpbin.org/get")
```

### Async Proxy Usage

```python
import httpx

async with httpx.AsyncClient(proxy="http://proxy.example.com:8080") as client:
    response = await client.get("https://httpbin.org/get")
```

---

## Testing with MockTransport

`httpx` provides `MockTransport` for writing tests that do not make real HTTP requests. You supply a handler function that receives a request and returns a response.

```python
import httpx

def mock_handler(request: httpx.Request) -> httpx.Response:
    """A mock transport handler that simulates an API."""
    if request.url.path == "/users" and request.method == "GET":
        return httpx.Response(
            200,
            json=[
                {"id": 1, "name": "Alice"},
                {"id": 2, "name": "Bob"},
            ],
        )
    elif request.url.path == "/users" and request.method == "POST":
        body = request.content.decode()
        return httpx.Response(201, json={"id": 3, "name": "Carol"})
    elif request.url.path == "/error":
        return httpx.Response(500, text="Internal Server Error")
    else:
        return httpx.Response(404, json={"error": "not found"})


# Create a client with the mock transport
transport = httpx.MockTransport(mock_handler)

with httpx.Client(transport=transport, base_url="https://api.test") as client:
    # Test GET /users
    response = client.get("/users")
    assert response.status_code == 200
    assert len(response.json()) == 2

    # Test POST /users
    response = client.post("/users", json={"name": "Carol"})
    assert response.status_code == 201

    # Test 404
    response = client.get("/nonexistent")
    assert response.status_code == 404

    # Test error
    response = client.get("/error")
    assert response.status_code == 500
```

### Async MockTransport

```python
import httpx
import pytest

async def async_mock_handler(request: httpx.Request) -> httpx.Response:
    if request.url.path == "/data":
        return httpx.Response(200, json={"result": "ok"})
    return httpx.Response(404)

@pytest.mark.anyio
async def test_async_api():
    transport = httpx.MockTransport(async_mock_handler)
    async with httpx.AsyncClient(transport=transport, base_url="https://api.test") as client:
        response = await client.get("/data")
        assert response.status_code == 200
        assert response.json() == {"result": "ok"}
```

### Using MockTransport for Unit Testing

A common pattern is to inject the client into your application code so it can be replaced in tests:

```python
import httpx
from dataclasses import dataclass


@dataclass
class ApiClient:
    """Application-level API client that wraps httpx."""
    client: httpx.Client

    def get_user(self, user_id: int) -> dict:
        response = self.client.get(f"/users/{user_id}")
        response.raise_for_status()
        return response.json()


# Production usage
def create_real_client() -> ApiClient:
    client = httpx.Client(base_url="https://api.example.com")
    return ApiClient(client=client)


# Test usage
def test_get_user():
    def handler(request: httpx.Request) -> httpx.Response:
        assert request.url.path == "/users/42"
        return httpx.Response(200, json={"id": 42, "name": "Alice"})

    transport = httpx.MockTransport(handler)
    client = httpx.Client(transport=transport, base_url="https://api.test")
    api = ApiClient(client=client)

    user = api.get_user(42)
    assert user == {"id": 42, "name": "Alice"}

    client.close()
```

---

## Async Usage

### Full Async Patterns with AsyncClient

```python
import httpx
import asyncio

async def fetch_url(client: httpx.AsyncClient, url: str) -> dict:
    response = await client.get(url)
    response.raise_for_status()
    return response.json()

async def main():
    async with httpx.AsyncClient() as client:
        # Sequential requests
        data1 = await fetch_url(client, "https://httpbin.org/get")
        data2 = await fetch_url(client, "https://httpbin.org/get")

        # Concurrent requests with asyncio.gather
        results = await asyncio.gather(
            fetch_url(client, "https://httpbin.org/get"),
            fetch_url(client, "https://httpbin.org/get"),
            fetch_url(client, "https://httpbin.org/get"),
        )
        print(f"Fetched {len(results)} responses concurrently")

asyncio.run(main())
```

### Concurrent Requests with Rate Limiting

```python
import httpx
import asyncio

async def fetch_with_semaphore(
    client: httpx.AsyncClient,
    url: str,
    semaphore: asyncio.Semaphore,
) -> httpx.Response:
    async with semaphore:
        return await client.get(url)

async def main():
    urls = [f"https://httpbin.org/get?id={i}" for i in range(50)]
    semaphore = asyncio.Semaphore(10)  # Max 10 concurrent requests

    async with httpx.AsyncClient() as client:
        tasks = [fetch_with_semaphore(client, url, semaphore) for url in urls]
        responses = await asyncio.gather(*tasks)
        print(f"Fetched {len(responses)} URLs")

asyncio.run(main())
```

### Async Streaming

```python
import httpx
import asyncio

async def stream_download(url: str, filepath: str):
    async with httpx.AsyncClient() as client:
        async with client.stream("GET", url) as response:
            response.raise_for_status()
            with open(filepath, "wb") as f:
                async for chunk in response.aiter_bytes(chunk_size=8192):
                    f.write(chunk)

asyncio.run(stream_download("https://example.com/file.zip", "file.zip"))
```

### Async Event Hooks

```python
import httpx

async def log_request(request: httpx.Request):
    print(f"-> {request.method} {request.url}")

async def log_response(response: httpx.Response):
    request = response.request
    print(f"<- {response.status_code} {request.method} {request.url}")

async def main():
    async with httpx.AsyncClient(
        event_hooks={
            "request": [log_request],
            "response": [log_response],
        }
    ) as client:
        await client.get("https://httpbin.org/get")

import asyncio
asyncio.run(main())
```

---

## Gotchas and Common Mistakes

### 1. Client Not Closed (Use a Context Manager)

If you instantiate a `Client` or `AsyncClient` without closing it, connections leak and you get resource warnings.

```python
# BAD: client is never closed -- leaked connections
client = httpx.Client()
response = client.get("https://httpbin.org/get")
# ... client.close() never called

# GOOD: context manager ensures cleanup
with httpx.Client() as client:
    response = client.get("https://httpbin.org/get")
# client.close() is called automatically

# GOOD: explicit close (when context manager is not possible)
client = httpx.Client()
try:
    response = client.get("https://httpbin.org/get")
finally:
    client.close()
```

### 2. Redirect Following Is Disabled by Default

Unlike `requests` (which follows redirects by default), `httpx` does **not** follow redirects unless you explicitly opt in. This is a deliberate safety feature to prevent surprising behavior.

```python
import httpx

# This will return a 301/302 response, NOT the final destination
response = httpx.get("https://httpbin.org/redirect/3")
print(response.status_code)   # 302 (not 200!)
print(response.is_redirect)   # True

# Enable redirect following
response = httpx.get("https://httpbin.org/redirect/3", follow_redirects=True)
print(response.status_code)   # 200

# On a client (applies to all requests)
with httpx.Client(follow_redirects=True) as client:
    response = client.get("https://httpbin.org/redirect/3")
    print(response.status_code)  # 200
```

### 3. Timeout Defaults (5 Seconds vs requests' None)

`httpx` uses a **5-second default timeout** for all phases (connect, read, write, pool). `requests` has **no default timeout**, which can cause applications to hang indefinitely.

```python
import httpx

# This will raise httpx.ReadTimeout if the server takes > 5 seconds
# response = httpx.get("https://httpbin.org/delay/10")  # ReadTimeout!

# Increase timeout for slow endpoints
response = httpx.get("https://httpbin.org/delay/10", timeout=30.0)

# Disable timeout (use with caution)
response = httpx.get("https://httpbin.org/delay/10", timeout=None)
```

### 4. Response Encoding Detection

`httpx` uses a different encoding detection strategy than `requests`:

- `requests` uses `chardet` (or `charset_normalizer`) to guess encoding from the response body.
- `httpx` uses the charset declared in the `Content-Type` header. If none is declared, it defaults to `utf-8` (per the HTTP spec for JSON and most modern APIs).

```python
import httpx

response = httpx.get("https://example.com")

# Check what encoding httpx detected
print(response.encoding)  # e.g., 'utf-8'

# Override encoding if needed
response.encoding = "latin-1"
text = response.text  # Re-decoded with latin-1
```

### 5. Connection Pool Limits

The default connection pool limits are generous but may need tuning for high-concurrency applications:

```python
import httpx

# Default limits: 100 max connections, 20 max keepalive connections
# Increase for high-throughput apps
limits = httpx.Limits(
    max_connections=200,
    max_keepalive_connections=50,
)

with httpx.Client(limits=limits) as client:
    # ...
    pass
```

If all connections in the pool are busy, new requests will wait up to the `pool` timeout (default 5 seconds) before raising `httpx.PoolTimeout`.

### 6. Sync vs Async Client Mixing

Do not use `httpx.Client` inside async code, or `httpx.AsyncClient` in sync code. They are not interchangeable.

```python
import httpx
import asyncio

# BAD: using sync Client in async code -- blocks the event loop
async def bad_example():
    with httpx.Client() as client:       # WRONG: blocks!
        response = client.get("https://httpbin.org/get")

# GOOD: use AsyncClient in async code
async def good_example():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://httpbin.org/get")

# BAD: using AsyncClient in sync code -- requires an event loop
def also_bad():
    async with httpx.AsyncClient() as client:  # SyntaxError or RuntimeError
        pass

# GOOD: use Client in sync code
def also_good():
    with httpx.Client() as client:
        response = client.get("https://httpbin.org/get")
```

### 7. Content vs Data vs JSON

Only one of `content`, `data`, `json`, or `files` should be used per request. Using multiple is ambiguous and will raise an error.

```python
import httpx

# Raw bytes/string body
httpx.post("https://httpbin.org/post", content=b"raw bytes")
httpx.post("https://httpbin.org/post", content="raw string")

# Form-encoded body (application/x-www-form-urlencoded)
httpx.post("https://httpbin.org/post", data={"field": "value"})

# JSON body (application/json) -- auto-serialized and Content-Type set
httpx.post("https://httpbin.org/post", json={"key": "value"})

# File upload (multipart/form-data)
httpx.post("https://httpbin.org/post", files={"file": open("report.pdf", "rb")})

# BAD: don't combine these
# httpx.post(url, data={"x": 1}, json={"y": 2})  # Error!
```

---

## Migration from requests

`httpx` is intentionally API-compatible with `requests` for the most common cases. Here are the key differences:

| Feature | `requests` | `httpx` |
|---------|-----------|---------|
| **Redirect following** | Enabled by default | **Disabled by default**; use `follow_redirects=True` |
| **Default timeout** | `None` (no timeout) | **5 seconds** for all phases |
| **Async support** | None (requires `aiohttp`) | Built-in `AsyncClient` |
| **HTTP/2** | Not supported | Supported via `httpx[http2]` |
| **Connection pool** | `requests.Session()` | `httpx.Client()` |
| **Session** | `requests.Session` | `httpx.Client` (no `Session` alias) |
| **Response encoding** | Guesses from body via chardet | Uses `Content-Type` header, defaults to `utf-8` |
| **Streaming** | `response.iter_content()` | `response.iter_bytes()`, `response.iter_text()`, `response.iter_lines()` |
| **Proxy config** | `proxies={"http": ..., "https": ...}` | `proxy="http://..."` (single proxy URL) |
| **Certificate verify** | `verify=True` | `verify=True` (same) |
| **Auth tuple** | `(user, pass)` | `(user, pass)` (same) |
| **Hooks** | `hooks={"response": [...]}` | `event_hooks={"request": [...], "response": [...]}` |
| **Prepared requests** | `req = Request(); session.send(req)` | `req = client.build_request(); client.send(req)` |

### Quick Migration Checklist

```python
# requests
import requests

s = requests.Session()
s.headers.update({"Authorization": "Bearer token"})
r = s.get("https://api.example.com/data")      # follows redirects
r.raise_for_status()
data = r.json()

# httpx equivalent
import httpx

with httpx.Client(
    headers={"Authorization": "Bearer token"},
    follow_redirects=True,                       # <-- must opt in
) as client:
    r = client.get("https://api.example.com/data")
    r.raise_for_status()
    data = r.json()
```

---

## Complete Code Examples

### Example 1: Basic GET and POST

```python
import httpx

# Simple GET
response = httpx.get("https://httpbin.org/get", params={"search": "python"})
print(response.status_code)  # 200
print(response.json()["args"])  # {'search': 'python'}

# Simple POST with JSON
response = httpx.post(
    "https://httpbin.org/post",
    json={"name": "Alice", "email": "alice@example.com"},
)
print(response.status_code)  # 200
print(response.json()["json"])  # {'name': 'Alice', 'email': 'alice@example.com'}

# POST with form data
response = httpx.post(
    "https://httpbin.org/post",
    data={"username": "alice", "password": "secret"},
)
print(response.json()["form"])  # {'username': 'alice', 'password': 'secret'}
```

### Example 2: Client with Connection Pooling

```python
import httpx

with httpx.Client(
    base_url="https://jsonplaceholder.typicode.com",
    headers={"Accept": "application/json"},
    timeout=10.0,
    follow_redirects=True,
) as client:
    # All requests reuse the connection pool
    users = client.get("/users").json()
    print(f"Found {len(users)} users")

    first_user = client.get(f"/users/{users[0]['id']}").json()
    print(f"First user: {first_user['name']}")

    posts = client.get("/posts", params={"userId": first_user["id"]}).json()
    print(f"User has {len(posts)} posts")

    # Create a new post
    new_post = client.post("/posts", json={
        "title": "New Post",
        "body": "This is the body.",
        "userId": first_user["id"],
    })
    new_post.raise_for_status()
    print(f"Created post with id: {new_post.json()['id']}")
```

### Example 3: File Uploads

```python
import httpx

# Upload a single file
with open("report.pdf", "rb") as f:
    response = httpx.post(
        "https://httpbin.org/post",
        files={"file": ("report.pdf", f, "application/pdf")},
    )
print(response.status_code)

# Upload multiple files
with open("photo1.jpg", "rb") as f1, open("photo2.jpg", "rb") as f2:
    response = httpx.post(
        "https://httpbin.org/post",
        files=[
            ("images", ("photo1.jpg", f1, "image/jpeg")),
            ("images", ("photo2.jpg", f2, "image/jpeg")),
        ],
    )

# Upload file with additional form fields
with open("data.csv", "rb") as f:
    response = httpx.post(
        "https://httpbin.org/post",
        data={"description": "Monthly report", "format": "csv"},
        files={"file": ("data.csv", f, "text/csv")},
    )

# Upload from bytes (no file on disk needed)
response = httpx.post(
    "https://httpbin.org/post",
    files={"file": ("hello.txt", b"Hello, World!", "text/plain")},
)
```

### Example 4: Streaming Large Responses

```python
import httpx
import hashlib

def download_with_progress(url: str, filepath: str) -> str:
    """Download a file with progress reporting and checksum verification."""
    sha256 = hashlib.sha256()

    with httpx.stream("GET", url, follow_redirects=True) as response:
        response.raise_for_status()
        total = int(response.headers.get("content-length", 0))
        downloaded = 0

        with open(filepath, "wb") as f:
            for chunk in response.iter_bytes(chunk_size=65536):
                f.write(chunk)
                sha256.update(chunk)
                downloaded += len(chunk)
                if total:
                    pct = downloaded / total * 100
                    print(f"\rDownloading: {pct:.1f}% ({downloaded}/{total})", end="")

    print()  # newline after progress
    return sha256.hexdigest()

# Usage
# checksum = download_with_progress("https://example.com/large-file.zip", "large-file.zip")
# print(f"SHA256: {checksum}")
```

### Example 5: Async Client Usage

```python
import httpx
import asyncio
from typing import Any

async def fetch_all_pages(
    client: httpx.AsyncClient,
    base_url: str,
    max_pages: int = 10,
) -> list[dict[str, Any]]:
    """Fetch multiple pages of a paginated API concurrently."""
    # First, get page 1 to discover total pages
    first = await client.get(base_url, params={"page": 1})
    first.raise_for_status()
    results = [first.json()]

    # Fetch remaining pages concurrently
    total_pages = min(first.json().get("total_pages", 1), max_pages)
    if total_pages > 1:
        tasks = [
            client.get(base_url, params={"page": page})
            for page in range(2, total_pages + 1)
        ]
        responses = await asyncio.gather(*tasks)
        for resp in responses:
            resp.raise_for_status()
            results.append(resp.json())

    return results

async def main():
    async with httpx.AsyncClient(
        base_url="https://api.example.com",
        headers={"Authorization": "Bearer my-token"},
        timeout=30.0,
        follow_redirects=True,
    ) as client:
        pages = await fetch_all_pages(client, "/api/items")
        all_items = [item for page in pages for item in page.get("items", [])]
        print(f"Fetched {len(all_items)} items across {len(pages)} pages")

# asyncio.run(main())
```

### Example 6: Custom Auth with Token Refresh

```python
import httpx
import time
from typing import Generator

class OAuth2ClientCredentials(httpx.Auth):
    """OAuth2 client credentials flow with automatic token caching and refresh."""

    def __init__(self, token_url: str, client_id: str, client_secret: str):
        self.token_url = token_url
        self.client_id = client_id
        self.client_secret = client_secret
        self._access_token: str | None = None
        self._expires_at: float = 0

    @property
    def _token_is_valid(self) -> bool:
        return self._access_token is not None and time.time() < self._expires_at

    def auth_flow(
        self, request: httpx.Request
    ) -> Generator[httpx.Request, httpx.Response, None]:
        if not self._token_is_valid:
            # Fetch a new token
            token_request = httpx.Request(
                "POST",
                self.token_url,
                data={
                    "grant_type": "client_credentials",
                    "client_id": self.client_id,
                    "client_secret": self.client_secret,
                },
            )
            token_response = yield token_request
            token_data = token_response.json()
            self._access_token = token_data["access_token"]
            self._expires_at = time.time() + token_data.get("expires_in", 3600) - 30

        request.headers["Authorization"] = f"Bearer {self._access_token}"
        response = yield request

        # If token was rejected, clear cache and retry once
        if response.status_code == 401:
            self._access_token = None
            self._expires_at = 0

            token_request = httpx.Request(
                "POST",
                self.token_url,
                data={
                    "grant_type": "client_credentials",
                    "client_id": self.client_id,
                    "client_secret": self.client_secret,
                },
            )
            token_response = yield token_request
            token_data = token_response.json()
            self._access_token = token_data["access_token"]
            self._expires_at = time.time() + token_data.get("expires_in", 3600) - 30

            request.headers["Authorization"] = f"Bearer {self._access_token}"
            yield request


# Usage
auth = OAuth2ClientCredentials(
    token_url="https://auth.example.com/oauth/token",
    client_id="my-client-id",
    client_secret="my-client-secret",
)

with httpx.Client(auth=auth) as client:
    # Token is fetched automatically on the first request
    response = client.get("https://api.example.com/protected/resource")
    print(response.json())
```

### Example 7: Testing with MockTransport

```python
import httpx
import json

# Define a realistic mock API
def create_mock_api() -> httpx.MockTransport:
    """Create a mock transport simulating a REST API."""
    users_db = {
        1: {"id": 1, "name": "Alice", "email": "alice@example.com"},
        2: {"id": 2, "name": "Bob", "email": "bob@example.com"},
    }
    next_id = 3

    def handler(request: httpx.Request) -> httpx.Response:
        nonlocal next_id
        path = request.url.path
        method = request.method

        # GET /users
        if path == "/users" and method == "GET":
            return httpx.Response(200, json=list(users_db.values()))

        # GET /users/:id
        if path.startswith("/users/") and method == "GET":
            user_id = int(path.split("/")[-1])
            if user_id in users_db:
                return httpx.Response(200, json=users_db[user_id])
            return httpx.Response(404, json={"error": "User not found"})

        # POST /users
        if path == "/users" and method == "POST":
            body = json.loads(request.content)
            new_user = {"id": next_id, **body}
            users_db[next_id] = new_user
            next_id += 1
            return httpx.Response(201, json=new_user)

        # DELETE /users/:id
        if path.startswith("/users/") and method == "DELETE":
            user_id = int(path.split("/")[-1])
            if user_id in users_db:
                del users_db[user_id]
                return httpx.Response(204)
            return httpx.Response(404, json={"error": "User not found"})

        return httpx.Response(404, json={"error": "Not found"})

    return httpx.MockTransport(handler)


def test_user_crud():
    """Test full CRUD operations against the mock API."""
    transport = create_mock_api()

    with httpx.Client(transport=transport, base_url="https://api.test") as client:
        # List users
        response = client.get("/users")
        assert response.status_code == 200
        users = response.json()
        assert len(users) == 2

        # Get single user
        response = client.get("/users/1")
        assert response.status_code == 200
        assert response.json()["name"] == "Alice"

        # Create user
        response = client.post("/users", json={"name": "Carol", "email": "carol@test.com"})
        assert response.status_code == 201
        carol = response.json()
        assert carol["id"] == 3
        assert carol["name"] == "Carol"

        # Verify user was added
        response = client.get("/users")
        assert len(response.json()) == 3

        # Delete user
        response = client.delete(f"/users/{carol['id']}")
        assert response.status_code == 204

        # Verify user was removed
        response = client.get(f"/users/{carol['id']}")
        assert response.status_code == 404

    print("All CRUD tests passed")


test_user_crud()
```

---

## Exception Hierarchy

`httpx` provides a clear exception hierarchy for handling different error types:

```
httpx.HTTPError
    httpx.HTTPStatusError          # Raised by response.raise_for_status()
    httpx.RequestError             # Base for all request-sending errors
        httpx.TransportError
            httpx.TimeoutException
                httpx.ConnectTimeout
                httpx.ReadTimeout
                httpx.WriteTimeout
                httpx.PoolTimeout
            httpx.NetworkError
                httpx.ConnectError
                httpx.ReadError
                httpx.WriteError
                httpx.CloseError
            httpx.ProtocolError
                httpx.LocalProtocolError
                httpx.RemoteProtocolError
        httpx.DecodingError
        httpx.TooManyRedirects
```

**Recommended error handling pattern:**

```python
import httpx

try:
    with httpx.Client() as client:
        response = client.get("https://api.example.com/data", follow_redirects=True)
        response.raise_for_status()
        data = response.json()
except httpx.HTTPStatusError as e:
    print(f"HTTP error {e.response.status_code}: {e.response.text}")
except httpx.ConnectTimeout:
    print("Could not connect to server (connection timed out)")
except httpx.ReadTimeout:
    print("Server did not send response in time (read timed out)")
except httpx.TimeoutException:
    print("Request timed out")
except httpx.NetworkError:
    print("Network error (DNS failure, refused connection, etc.)")
except httpx.TooManyRedirects:
    print("Too many redirects")
except httpx.HTTPError as e:
    print(f"HTTP error: {e}")
```

---

## Quick Reference

| Task | Code |
|------|------|
| Simple GET | `httpx.get(url)` |
| GET with params | `httpx.get(url, params={"key": "val"})` |
| POST JSON | `httpx.post(url, json={"key": "val"})` |
| POST form | `httpx.post(url, data={"key": "val"})` |
| File upload | `httpx.post(url, files={"f": open("f.txt","rb")})` |
| Set headers | `httpx.get(url, headers={"X-Key": "val"})` |
| Basic auth | `httpx.get(url, auth=("user", "pass"))` |
| Follow redirects | `httpx.get(url, follow_redirects=True)` |
| Set timeout | `httpx.get(url, timeout=30.0)` |
| Disable timeout | `httpx.get(url, timeout=None)` |
| Sync client | `with httpx.Client() as c: c.get(url)` |
| Async client | `async with httpx.AsyncClient() as c: await c.get(url)` |
| Stream response | `with httpx.stream("GET", url) as r: ...` |
| Enable HTTP/2 | `httpx.Client(http2=True)` |
| Use proxy | `httpx.Client(proxy="http://proxy:8080")` |
| Raise on error | `response.raise_for_status()` |
| Read JSON | `response.json()` |
| Read text | `response.text` |
| Read bytes | `response.content` |
| Check status | `response.status_code` / `response.is_success` |

---

## Further Reading

- [Official documentation](https://www.python-httpx.org)
- [GitHub repository](https://github.com/encode/httpx)
- [PyPI page](https://pypi.org/project/httpx/)
- [httpcore (transport layer)](https://github.com/encode/httpcore)
- [h2 (HTTP/2 implementation)](https://github.com/python-hyper/h2)
