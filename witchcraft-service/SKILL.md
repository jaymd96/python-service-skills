---
name: witchcraft-service
description: Workflow for building production Python services using the Witchcraft stack. Covers defining API contracts with Enchant DSL, generating models/server/client code, implementing the server on Cherry, validating data with Rune, and adding schema migrations with Transmute. Use when building a new service end-to-end or integrating these tools together. Triggers on witchcraft service, build service, new service, enchant cherry rune, service workflow, API contract, contract-first.
---

# Witchcraft Service — Contract-First Service Development

## Workflow Overview

```
1. Define contract (.enchant.yml)
   → 2. Generate code (models + server stub + client)
     → 3. Implement server (Cherry + Rune validation)
       → 4. Add schema migrations (Transmute)
         → 5. Wire up client as a Python package
```

## Step 1: Define the API Contract

```yaml
# catalog-service.enchant.yml
types:
  Product:
    fields:
      id: { type: integer, safety: safe }
      name: { type: string, safety: safe, validation: { min-length: 1 } }
      description: { type: "optional<string>", safety: safe }

  CreateProductRequest:
    fields:
      name: { type: string, safety: safe, validation: { min-length: 1 } }

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

## Step 2: Generate Code

```bash
enchant build catalog-service.enchant.yml \
  --models --server --client \
  --output catalog/generated/
```

## Step 3: Implement on Cherry

```python
from py_witchcraft_server.server.server import WitchcraftServer
from catalog.generated.server.catalog_service import CatalogServiceBase
from catalog.generated.models.catalog_types import Product, CreateProductRequest

class CatalogServiceImpl(CatalogServiceBase):
    async def list_products(self) -> list[Product]:
        return self._repo.list_all()

    async def get_product(self, product_id: int) -> Product:
        product = self._repo.get(product_id)
        if not product:
            raise ProductNotFound(product_id=str(product_id))
        return product

    async def create_product(self, request: CreateProductRequest) -> Product:
        return self._repo.create(request.name)

def init_app(ctx, info):
    service = CatalogServiceImpl(repo=ProductRepo(db))
    service.register(info.router)
    ctx.health_reporter.initialize_health_component("DATABASE").healthy()

server = WitchcraftServer().with_init_func(init_app).with_self_signed_certificate()
server.start()
```

## Step 4: Add Rune Validation

```python
from rune import inscribed, field
from typing import Annotated
from rune.constraints import MinLen, Gt

@inscribed
class CreateProductRequest:
    name: Annotated[str, MinLen(1)]

@inscribed
class Product:
    id: Annotated[int, Gt(0)]
    name: str
    description: str | None = None
```

## Key Integration Points

- **Enchant models use attrs/cattrs** — same foundation as Rune's `@inscribed`
- **Generated servers implement Cherry interfaces** — `CatalogServiceBase` registers routes on Cherry's `Router`
- **Generated clients use enchant-dialogue** — `EnchantChannel` with AIMD concurrency, retry, and safety logging
- **Safety annotations flow through** — `safe`/`unsafe` in `.enchant.yml` → structlog redaction in Cherry
- **Transmute migrates schemas** that Enchant models depend on, with dual-write proxy for zero-downtime

## References

- **[integration.md](references/integration.md)** — How Rune, Cherry, Enchant, and Transmute connect, shared foundations, data flow
- **[walkthrough.md](references/walkthrough.md)** — Complete step-by-step example building a service from scratch
- **[patterns.md](references/patterns.md)** — Common patterns, testing strategies, client packaging, gotchas
