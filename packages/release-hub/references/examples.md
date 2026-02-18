# Release-Hub — Examples & Gotchas

> Part of the release-hub skill. See [SKILL.md](../SKILL.md) for overview.

## Publish a Release

```python
from pants_release_hub import ApolloHubClient, PublishRequest, build_artifact_url

client = ApolloHubClient(
    base_url="https://hub.example.com",
    auth_token="Bearer my-token",
)

request = PublishRequest(
    product_group="com.example",
    product_name="catalog-service",
    product_version="1.0.0",
    product_type="helm.v1",
    artifact_url=build_artifact_url("registry.example.io", "catalog-service", "1.0.0"),
    manifest_yaml=open("deployment/manifest.yml").read(),
    published_by="ci-pipeline",
)

result = client.publish_release(request)
if result.success:
    print(f"Published: {result.release_id}")
else:
    print(f"Failed: {result.message}")
```

## Dry Run

```python
request = PublishRequest(
    product_group="com.example",
    product_name="my-service",
    product_version="1.0.0",
    product_type="helm.v1",
    artifact_url="oci://registry.example.io/my-service:1.0.0",
    manifest_yaml="...",
    dry_run=True,
)
result = client.publish_release(request)
# result.dry_run == True, no actual publish
```

## Version Detection

```python
from pants_release_hub import detect_version, default_channel_for_release_type

# Stable release
info = detect_version("1.0.0")
# format=SEMVER, release_type=RELEASE
channel = default_channel_for_release_type(info.release_type)  # "stable"

# Release candidate
info = detect_version("1.0.0-rc1")
# format=SLS, release_type=RELEASE_CANDIDATE → "beta"

# Snapshot
info = detect_version("1.0.0-5-gabcdef")
# format=SLS, release_type=SNAPSHOT → "dev"

# CalVer
info = detect_version("2024.01.15")
# format=CALVER, release_type=RELEASE → "stable"

# PEP 440
info = detect_version("1.0.0a1")
# format=PEP440, release_type=PRERELEASE → "dev"
```

## Release Management

```python
# List releases
page = client.list_releases("com.example:catalog-service", channel="stable")
for release in page.items:
    print(f"{release.version} ({release.channel}) - {release.state}")

# Paginate
while page.has_next:
    page = client.list_releases(
        "com.example:catalog-service",
        page=page.page + 1,
    )

# Promote release
client.promote_release(release_id="abc123", target_channel="stable")

# Deprecate release
client.deprecate_release(release_id="abc123", reason="Security vulnerability CVE-2024-XXX")

# Get product info
product = client.get_product("com.example:catalog-service")
```

## Custom Retry Policy

```python
from pants_release_hub import ApolloHubClient, RetryPolicy, NO_RETRY

# Aggressive retry for CI
ci_policy = RetryPolicy(
    max_attempts=5,
    base_delay_seconds=2.0,
    max_delay_seconds=60.0,
    retryable_status_codes=(429, 500, 502, 503, 504),
)
client = ApolloHubClient(
    base_url="https://hub.example.com",
    auth_token="...",
    retry_policy=ci_policy,
)

# No retry for testing
client = ApolloHubClient(
    base_url="http://localhost:8000",
    retry_policy=NO_RETRY,
)
```

## Error Handling

```python
from pants_release_hub import (
    ApolloHubClient, PublishRequest,
    ConflictError, AuthError, ValidationError,
    ConnectionError, RetryExhaustedError,
)

try:
    result = client.publish_release(request)
except ConflictError:
    print("Release already exists (409)")
except AuthError:
    print("Authentication failed (401/403)")
except ValidationError as e:
    print(f"Invalid request (400): {e}")
except ConnectionError:
    print("Cannot reach Apollo Hub")
except RetryExhaustedError:
    print("All retry attempts exhausted")
```

## Gotchas

1. **Zero external dependencies**: Uses only `urllib` and `json` from the standard library. No `requests` or `httpx`
2. **Channel auto-detection**: If `channel` is not set on `PublishRequest`, it's auto-detected from version format. Set it explicitly to override
3. **`product_id` format**: The `product_id` used in API calls is `"{product_group}:{product_name}"` — colon-separated, not slash-separated
4. **Auth token format**: Pass the full token string. The client prepends `Bearer ` in the Authorization header. Don't include `Bearer ` in the token value
5. **ConflictError on duplicates**: Publishing the same version twice returns 409. Check for existing releases before publishing or handle `ConflictError`
6. **Detection order matters**: `detect_version` tries CalVer → SLS → PEP 440 → SemVer → UNKNOWN. A version like `2024.1.0` matches CalVer before SemVer
7. **Retry only on server errors**: Default retryable codes are `429, 500, 502, 503, 504`. Client errors (400, 401, 403, 404) are never retried
