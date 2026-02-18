---
name: enchant
description: Type-safe API contract code generation. Use when defining service contracts in .enchant.yml DSL, generating attrs models (gen-models), server stubs (gen-server), typed httpx clients (gen-client), using EnchantChannel with AIMD concurrency and retry, running conformance tests (enchant-verify), or integrating with Cherry and Rune. Triggers on enchant, .enchant.yml, gen-models, gen-server, gen-client, EnchantChannel, service contract, code generation, enchant-dialogue.
---

# Enchant — Type-Safe Code Generation (v0.1.0)

## Quick Start

```bash
pip install enchant-cli
```

```yaml
# my-service.enchant.yml
types:
  User:
    fields:
      id: { type: integer, safety: safe }
      name: { type: string, safety: safe, validation: { min-length: 1 } }
      email: { type: string, safety: unsafe }

services:
  UserService:
    base-path: /api/users
    endpoints:
      getUser:
        http: GET /{userId}
        args:
          userId: { type: integer, param-type: path }
        returns: User
```

```bash
enchant build my-service.enchant.yml --models --server --client --output generated/
```

## Key Patterns

### Architecture
```
.enchant.yml -> enchant-spec (-> OpenAPI 3.1 internal) -> gen-models + gen-server + gen-client
                                                            |            |              |
                                                         attrs        abstract       httpx
                                                         classes      base +         client via
                                                         + cattrs     router fn      EnchantChannel
```

### Using generated client
```python
from enchant_dialogue import EnchantChannel, ChannelConfig
from generated.client.user_client import UserServiceClient

channel = EnchantChannel(ChannelConfig(
    uris=["https://api.example.com"],
    max_retries=3,
    auth_token_provider=lambda: get_token(),
))
client = UserServiceClient(channel)
user = client.get_user(user_id=42)
```

## References

- **[dsl.md](references/dsl.md)** — .enchant.yml type system, validation keywords, safety annotations, services, errors
- **[codegen.md](references/codegen.md)** — gen-models, gen-server, gen-client output formats, enchant-verify, enchant-cli
- **[dialogue.md](references/dialogue.md)** — EnchantChannel, AIMD concurrency, retry policy, node selection, ChannelConfig
- **[examples.md](references/examples.md)** — Complete service definition, server implementation, client usage, CI integration

## Grep Patterns

- `\.enchant\.yml` — Find contract definition files
- `EnchantChannel|ChannelConfig` — Find client channel setup
- `ServiceBase|register_.*_service` — Find generated server stubs
- `enchant build|enchant verify` — Find CLI invocations
