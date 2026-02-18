# httpx — Auth & Transport

> Part of the httpx skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Basic Authentication](#basic-authentication)
- [Digest Authentication](#digest-authentication)
- [Bearer Token Authentication](#bearer-token-authentication)
- [Custom Auth Flows](#custom-auth-flows)
- [Transport Customization](#transport-customization)
- [Connection Pool Limits](#connection-pool-limits)
- [HTTP/2 Support](#http2-support)
- [Proxy Support](#proxy-support)
- [Event Hooks](#event-hooks)
- [Redirect Following](#redirect-following)

---

## Basic Authentication

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

---

## Digest Authentication

```python
import httpx

auth = httpx.DigestAuth(username="user", password="pass")
response = httpx.get("https://httpbin.org/digest-auth/auth/user/pass", auth=auth)
```

---

## Bearer Token Authentication

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

---

## Custom Auth Flows

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

## Transport Customization

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

---

## Connection Pool Limits

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

## HTTP/2 Support

HTTP/2 provides multiplexing (multiple requests over a single connection), header compression, and server push. `httpx` supports HTTP/2 via the optional `h2` dependency.

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

## Event Hooks

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

---

## Redirect Following

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
