# httpx — Examples & Gotchas

> Part of the httpx skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [Client Not Closed](#1-client-not-closed-use-a-context-manager)
  - [Redirect Following Is Disabled by Default](#2-redirect-following-is-disabled-by-default)
  - [Timeout Defaults](#3-timeout-defaults-5-seconds-vs-requests-none)
  - [Response Encoding Detection](#4-response-encoding-detection)
  - [Connection Pool Limits](#5-connection-pool-limits)
  - [Sync vs Async Client Mixing](#6-sync-vs-async-client-mixing)
  - [Content vs Data vs JSON](#7-content-vs-data-vs-json)
- [Migration from requests](#migration-from-requests)
- [Complete Code Examples](#complete-code-examples)
  - [Basic GET and POST](#example-1-basic-get-and-post)
  - [Client with Connection Pooling](#example-2-client-with-connection-pooling)
  - [File Uploads](#example-3-file-uploads)
  - [Streaming Large Responses](#example-4-streaming-large-responses)
  - [Async Client Usage](#example-5-async-client-usage)
  - [Custom Auth with Token Refresh](#example-6-custom-auth-with-token-refresh)
  - [Testing with MockTransport](#example-7-testing-with-mocktransport)
- [Further Reading](#further-reading)

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

## Further Reading

- [Official documentation](https://www.python-httpx.org)
- [GitHub repository](https://github.com/encode/httpx)
- [PyPI page](https://pypi.org/project/httpx/)
- [httpcore (transport layer)](https://github.com/encode/httpcore)
- [h2 (HTTP/2 implementation)](https://github.com/python-hyper/h2)
