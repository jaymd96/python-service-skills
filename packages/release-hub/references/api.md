# Release-Hub — Core API Reference

> Part of the release-hub skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [ApolloHubClient](#apollohubclient)
- [PublishRequest](#publishrequest)
- [PublishResult](#publishresult)
- [Release](#release)
- [Page](#page)
- [Version Detection](#version-detection)
- [RetryPolicy](#retrypolicy)
- [Error Types](#error-types)

## ApolloHubClient

HTTP client for Apollo Hub API. Uses `urllib` — zero external dependencies.

```python
from pants_release_hub import ApolloHubClient

client = ApolloHubClient(
    base_url="https://hub.example.com",
    auth_token="Bearer ...",       # Optional
    timeout=30,                    # Request timeout (seconds)
    retry_policy=None,             # Optional RetryPolicy
)
```

### Methods

| Method | HTTP | Description |
|--------|------|-------------|
| `publish_release(request: PublishRequest) -> PublishResult` | POST `/api/v1/releases` | Publish a release |
| `check_health() -> bool` | GET `/api/v1/health` | Health check |
| `get_product(product_id: str) -> dict \| None` | GET `/api/v1/products/{id}` | Get product info |
| `list_releases(product_id, channel=None, page=1, page_size=20) -> Page` | GET `/api/v1/products/{id}/releases` | List releases |
| `promote_release(release_id: str, target_channel: str) -> dict` | POST `/api/v1/releases/{id}/promote` | Promote to channel |
| `deprecate_release(release_id: str, reason: str = "") -> dict` | POST `/api/v1/releases/{id}/deprecate` | Deprecate release |

## PublishRequest

Frozen dataclass for publishing a release.

```python
from pants_release_hub import PublishRequest

request = PublishRequest(
    product_group="com.example",       # Required
    product_name="my-service",         # Required
    product_version="1.0.0",           # Required
    product_type="helm.v1",            # Required
    artifact_url="oci://registry/...", # Required
    manifest_yaml="...",               # Required
    channel=None,                      # Auto-detected from version
    release_type=None,                 # Auto-detected from version
    changelog=None,                    # Optional changelog
    published_by=None,                 # Optional publisher ID
    labels={},                         # Additional labels
    dry_run=False,                     # Preview without publishing
)
```

### Properties

| Property | Returns | Description |
|----------|---------|-------------|
| `product_id` | str | `"{product_group}:{product_name}"` |
| `resolved_release_type` | ReleaseType | Auto-detect if not explicitly set |
| `resolved_channel` | str | Auto-detect if not explicitly set |

### Methods

| Method | Description |
|--------|-------------|
| `to_api_payload() -> dict` | Build API request payload |

## PublishResult

Frozen dataclass for publish response.

```python
from pants_release_hub import PublishResult

# Fields
result.success: bool
result.release_id: str | None
result.message: str
result.dry_run: bool
result.api_response: dict

# Class methods
PublishResult.dry_run_result(request)   # Preview result
PublishResult.from_api_response(resp)   # Parse API response
PublishResult.failure(message)          # Create failure result
```

## Release

Frozen dataclass representing a release.

```python
release.id: str
release.product_id: str
release.version: str
release.channel: str
release.state: str
release.artifact_url: str
release.metadata: dict
```

## Page

Paginated release listing.

```python
page.items: tuple[Release, ...]
page.total: int
page.page: int       # default 1
page.page_size: int  # default 20
page.has_next: bool
```

## Version Detection

```python
from pants_release_hub import detect_version, detect_release_type, default_channel_for_release_type
from pants_release_hub import VersionFormat, ReleaseType, VersionInfo
```

### detect_version

```python
info = detect_version("1.0.0")
# info.raw = "1.0.0"
# info.format = VersionFormat.SEMVER
# info.release_type = ReleaseType.RELEASE
# info.major = 1, info.minor = 0, info.patch = 0
```

Detection order: CalVer → SLS → PEP 440 → SemVer → UNKNOWN

### VersionFormat

| Value | Example |
|-------|---------|
| `SLS` | `1.0.0`, `1.0.0-rc1`, `1.0.0-5-gabcdef` |
| `SEMVER` | `1.0.0`, `1.0.0-beta.1+build` |
| `PEP440` | `1.0.0a1`, `1.0.0.dev1`, `1.0.0.post1` |
| `CALVER` | `2024.01.15` (year >= 2000) |
| `UNKNOWN` | Anything else |

### ReleaseType

| Value | Description |
|-------|-------------|
| `RELEASE` | Stable release |
| `RELEASE_CANDIDATE` | RC build |
| `SNAPSHOT` | Development snapshot |
| `RC_SNAPSHOT` | RC + development |
| `PRERELEASE` | Alpha/beta/dev |

### Channel Mapping

```python
channel = default_channel_for_release_type(release_type)
```

| Release Type | Channel |
|-------------|---------|
| `RELEASE` | `"stable"` |
| `RELEASE_CANDIDATE` | `"beta"` |
| `SNAPSHOT` | `"dev"` |
| `RC_SNAPSHOT` | `"dev"` |
| `PRERELEASE` | `"dev"` |

## RetryPolicy

```python
from pants_release_hub import RetryPolicy, NO_RETRY

policy = RetryPolicy(
    max_attempts=3,                              # default
    base_delay_seconds=1.0,                      # default
    max_delay_seconds=30.0,                      # default
    exponential_base=2.0,                        # default
    retryable_status_codes=(429, 500, 502, 503, 504),  # default
)
policy.delay_for_attempt(attempt) -> float  # Exponential backoff

NO_RETRY  # Singleton RetryPolicy(max_attempts=1)
```

### Helper function

```python
from pants_release_hub._retry import retry

result = retry(
    func=lambda: client.publish_release(request),
    policy=policy,
    is_retryable=None,  # Optional exception predicate
)
```

## Error Types

```python
from pants_release_hub import (
    ReleaseHubError,        # Base exception
    AuthError,              # 401/403
    ValidationError,        # 400
    ConnectionError,        # Cannot reach hub
    NotFoundError,          # 404
    ConflictError,          # 409 (duplicate release)
    RetryExhaustedError,    # All retries failed
)
```

### Utility

```python
from pants_release_hub import build_artifact_url

url = build_artifact_url("registry.example.io", "my-service", "1.0.0")
# "oci://registry.example.io/my-service:1.0.0"
```
