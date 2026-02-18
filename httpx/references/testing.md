# httpx — Testing & Mocking

> Part of the httpx skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [MockTransport](#mocktransport)
- [Async MockTransport](#async-mocktransport)
- [Using MockTransport for Unit Testing](#using-mocktransport-for-unit-testing)
- [Testing with a Realistic Mock API](#testing-with-a-realistic-mock-api)

---

## MockTransport

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

---

## Async MockTransport

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

---

## Using MockTransport for Unit Testing

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

## Testing with a Realistic Mock API

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
