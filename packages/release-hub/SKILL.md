---
name: release-hub
description: Apollo Hub publishing client with zero external dependencies. Use when publishing releases to Apollo Hub, detecting version formats (SLS, SemVer, PEP 440, CalVer), auto-mapping release channels, or managing release lifecycle. Triggers on release-hub, ApolloHubClient, publish release, version detection, release channel, Apollo Hub.
---

# release-hub — Apollo Hub Client (v0.1.0)

## Quick Start

```bash
pip install jaymd96-pants-release-hub
```

```python
from pants_release_hub import ApolloHubClient, PublishRequest

client = ApolloHubClient(base_url="https://hub.example.com", auth_token="...")
result = client.publish_release(PublishRequest(
    product_group="com.example", product_name="my-service", version="1.0.0",
    artifact_url="registry.example.io/my-service:1.0.0",
))
```

## Key Patterns

### Version auto-detection
```python
from pants_release_hub import detect_version, default_channel_for_release_type

info = detect_version("1.0.0")      # format=SEMVER, release_type=RELEASE
channel = default_channel_for_release_type(info.release_type)  # "stable"
# 1.0.0-rc1 -> beta, snapshot -> dev, CalVer -> stable
```

### Release management
```python
client.get_product("com.example", "my-service")
client.list_releases("com.example", "my-service")
client.promote_release(release_id, channel="stable")
client.deprecate_release(release_id)
```

## References

- **[api.md](references/api.md)** — ApolloHubClient, PublishRequest/Result, version detection, error types
- **[examples.md](references/examples.md)** — Publishing workflow, version format table, gotchas
