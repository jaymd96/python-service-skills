# Enchant — Examples & Gotchas

> Part of the enchant skill. See [SKILL.md](../SKILL.md) for overview.

## Complete Apollo Service Definition

```yaml
# apollo-catalog.enchant.yml
types:
  Product:
    fields:
      id: { type: integer, safety: safe }
      groupId: { type: string, safety: safe, validation: { min-length: 1 } }
      artifactId: { type: string, safety: safe, validation: { min-length: 1 } }
      name: { type: string, safety: safe }
      description: { type: "optional<string>", safety: safe }

  Release:
    fields:
      id: { type: integer, safety: safe }
      version: { type: string, safety: safe }
      releaseType: { type: ReleaseType, safety: safe }
      productId: { type: integer, safety: safe }

  ReleaseType:
    values: [RELEASE, RELEASE_CANDIDATE, SNAPSHOT]

  CreateProductRequest:
    fields:
      groupId: { type: string, safety: safe, validation: { min-length: 1 } }
      artifactId: { type: string, safety: safe, validation: { min-length: 1 } }
      name: { type: string, safety: safe }

errors:
  ProductNotFound:
    code: NOT_FOUND
    safe-args:
      productId: string

services:
  CatalogService:
    base-path: /api/catalog
    endpoints:
      listProducts:
        http: GET /products
        returns: list<Product>
      getProduct:
        http: GET /products/{productId}
        args:
          productId: { type: integer, param-type: path }
        returns: Product
        errors: [ProductNotFound]
      createProduct:
        http: POST /products
        args:
          request: { type: CreateProductRequest, param-type: body }
        returns: Product
```

## Building Generated Code

```bash
enchant build apollo-catalog.enchant.yml \
  --models --server --client \
  --output apollo/generated/

# Produces:
# apollo/generated/models/catalog_types.py     (attrs classes)
# apollo/generated/server/catalog_service.py    (abstract base + router)
# apollo/generated/client/catalog_client.py     (httpx client)
```

## Implementing Server Stub

```python
from apollo.generated.server.catalog_service import CatalogServiceBase
from apollo.generated.models.catalog_types import Product, CreateProductRequest

class CatalogServiceImpl(CatalogServiceBase):
    def __init__(self, session_factory):
        self._session_factory = session_factory

    async def list_products(self) -> list[Product]:
        with self._session_factory() as session:
            return [self._to_product(r) for r in list_all(session)]

    async def get_product(self, product_id: int) -> Product:
        with self._session_factory() as session:
            row = get_by_id(session, product_id)
            if not row:
                raise ProductNotFound(product_id=str(product_id))
            return self._to_product(row)

    async def create_product(self, request: CreateProductRequest) -> Product:
        with self._session_factory() as session:
            row = create(session, request.group_id, request.artifact_id, request.name)
            return self._to_product(row)
```

## Using Generated Client

```python
from enchant_dialogue import EnchantChannel, ChannelConfig
from apollo.generated.client.catalog_client import CatalogServiceClient

channel = EnchantChannel(ChannelConfig(
    uris=["https://hub.apollo.internal:8443"],
    auth_token_provider=lambda: get_service_token(),
    max_retries=3,
))
client = CatalogServiceClient(channel)

products = client.list_products()
product = client.get_product(product_id=42)
```

## CI Integration

```bash
# In CI pipeline
enchant check definitions/*.enchant.yml                          # Validate
enchant build definitions/*.enchant.yml --models --server --client --output generated/
enchant verify definitions/*.enchant.yml --test-suite tests/conformance/  # Conformance
```

## Ecosystem Integration

```
enchant (code generation)
    ├── Generated models use attrs + cattrs (same as rune)
    ├── Generated clients use enchant-dialogue (httpx + AIMD + retry)
    ├── Generated servers implement Cherry interfaces
    └── Safety annotations flow through structlog

rune (validation/serialization)
    ├── Same attrs + cattrs foundation as enchant models
    └── Constraints can augment generated models

transmute (migrations)
    ├── Migrates schemas that enchant models depend on
    └── Dual-write proxy works with generated server stubs
```

## Gotchas

1. **Never modify generated code**: It's overwritten on every build. Put implementation in separate files
2. **Use safety annotations on every field**: PII is always `unsafe`, internal IDs are `safe`, secrets are `do-not-log`
3. **One definition file per service domain**: Keep contracts focused
4. **DSL is the only authoring surface**: OpenAPI 3.1 is internal — never expose it to developers
5. **Forward compatibility by default**: Clients tolerate unknown fields, enum values, union variants
6. **Validation in the DSL**: Push constraints into `.enchant.yml` — generated code enforces automatically
