# httpx — API Reference

> Part of the httpx skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Quick Functions (Top-Level API)](#quick-functions-top-level-api)
- [The Response Object](#the-response-object)
- [Status Code Checking](#status-code-checking)
- [Client Usage](#client-usage)
- [httpx.Client (Synchronous)](#httpxclient-synchronous)
- [httpx.AsyncClient (Asynchronous)](#httpxasyncclient-asynchronous)
- [Base URL](#base-url)
- [Default Headers, Params, and Auth](#default-headers-params-and-auth)
- [Connection Pool Limits](#connection-pool-limits)
- [Timeout Configuration](#timeout-configuration)
- [Streaming](#streaming)
- [Async Usage](#async-usage)
- [File Uploads](#file-uploads)
- [Exception Hierarchy](#exception-hierarchy)
- [Quick Reference](#quick-reference)

---

## Quick Functions (Top-Level API)

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

---

## The Response Object

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

---

## Status Code Checking

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

---

## File Uploads

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
