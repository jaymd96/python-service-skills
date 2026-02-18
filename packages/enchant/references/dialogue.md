# Enchant — Client Runtime (enchant-dialogue)

> Part of the enchant skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [EnchantChannel](#enchantchannel)
- [ChannelConfig](#channelconfig)
- [AIMD Concurrency Control](#aimd-concurrency-control)
- [Retry Policy](#retry-policy)
- [Node Selection](#node-selection)
- [Safety and Logging](#safety-and-logging)
- [Distributed Tracing](#distributed-tracing)
- [Error Types](#error-types)

## EnchantChannel

Main HTTP channel that generated clients delegate to. Provides resilience, observability, and safety.

```python
from enchant_dialogue import EnchantChannel, ChannelConfig

channel = EnchantChannel(ChannelConfig(
    uris=["https://api-1.example.com", "https://api-2.example.com"],
    max_retries=3,
    auth_token_provider=lambda: get_token(),
))

# Used by generated clients internally
result = channel.execute(
    method="GET",
    path="/api/users/42",
    response_type=User,
)
```

Dependencies: `httpx`, `cattrs`, `structlog`, `opentelemetry` (optional)

## ChannelConfig

```python
config = ChannelConfig(
    uris=["https://api-1.example.com"],  # Required: target URIs
    connect_timeout=5.0,                  # Connection timeout (seconds)
    read_timeout=30.0,                    # Read timeout
    write_timeout=30.0,                   # Write timeout
    max_retries=3,                        # Max retry attempts

    # Concurrency
    initial_concurrency_limit=20,
    min_concurrency_limit=1,

    # Authentication
    auth_token="Bearer ...",              # Static token
    auth_token_provider=lambda: get_token(),  # Dynamic token (preferred)

    # HTTP settings
    user_agent="apollo-client/1.0",
    enable_http2=True,

    # TLS
    ca_bundle="/path/to/ca.pem",
    client_cert="/path/to/cert.pem",
    client_key="/path/to/key.pem",
    verify_ssl=True,

    # Live config
    uri_file="/path/to/uris.txt",        # Reload URIs from file
)
```

## AIMD Concurrency Control

Additive Increase / Multiplicative Decrease:

| Event | Action |
|-------|--------|
| Success (2xx) | `limit += 1` |
| Throttle (429) | `limit *= 0.5` |
| Unavailable (503) | `limit *= 0.5` |

- Configurable min/max limits
- FIFO queue for waiters when at capacity
- Prevents overwhelming downstream services

## Retry Policy

| Condition | Behavior |
|-----------|----------|
| Safe methods only | GET, HEAD, OPTIONS are retried |
| 429 Too Many Requests | Constant backoff + different node |
| 503 Service Unavailable | Exponential backoff + different node |
| 308 Permanent Redirect | Follow to Location header |
| 307 Temporary Redirect | Follow temporarily |
| Other errors | No retry |

Unsafe methods (POST, PUT, DELETE) are never retried.

## Node Selection

- **Round-robin** load balancing across configured URIs
- **Health tracking**: Nodes marked unhealthy on failure
- **Automatic recovery**: Unhealthy nodes retried after 30s interval
- **Least-recently-failed** fallback when all nodes unhealthy

## Safety and Logging

`SafeArg` / `UnsafeArg` type markers flow from DSL through generated models:

```python
from enchant_dialogue import SafeArg, UnsafeArg

# In generated models
origin: Annotated[str, SafeArg]           # Logged normally
passenger_name: Annotated[str, UnsafeArg]  # Redacted in production

# structlog processor respects annotations
# Production: {"origin": "LAX", "passenger_name": "<REDACTED>"}
```

## Distributed Tracing

- W3C `traceparent` header propagation
- Optional OpenTelemetry integration
- Falls back to synthetic trace IDs when no tracer configured

## Error Types

```python
from enchant_dialogue import (
    EnchantRemoteError,      # Server returned error response
    EnchantTimeoutError,     # Request timed out
    EnchantTransportError,   # Connection failed
)
```

### Metrics

Automatically collected:
- Request counts (total, successful, failed, retried)
- Duration tracking (total, average)
