# Witchcraft Service — Complete Walkthrough

> Part of the witchcraft-service workflow. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [Project Structure](#project-structure)
- [Step 1: Define the Contract](#step-1-define-the-contract)
- [Step 2: Generate Code](#step-2-generate-code)
- [Step 3: Implement the Server](#step-3-implement-the-server)
- [Step 4: Add Validation](#step-4-add-validation)
- [Step 5: Add Health Checks](#step-5-add-health-checks)
- [Step 6: Add Schema Migration](#step-6-add-schema-migration)
- [Step 7: Wire Up the Client](#step-7-wire-up-the-client)
- [Step 8: Test Everything](#step-8-test-everything)

## Project Structure

```
catalog-service/
  catalog-service.enchant.yml        # API contract
  catalog/
    generated/                       # Enchant output (never edit)
      models/catalog_types.py
      server/catalog_service.py
      client/catalog_client.py
    service/
      __init__.py
      impl.py                        # Service implementation
      models.py                      # Rune-validated domain models
      repo.py                        # Repository layer
    migrations/
      __init__.py
      v001_add_description.py        # Transmute migration
      registry.py
    app.py                           # Cherry server entry point
  tests/
    test_service.py
    test_models.py
  BUILD                              # Pants sls_service target
```

## Step 1: Define the Contract

```yaml
# catalog-service.enchant.yml
types:
  Product:
    fields:
      id: { type: integer, safety: safe }
      groupId: { type: string, safety: safe, validation: { min-length: 1 } }
      artifactId: { type: string, safety: safe, validation: { min-length: 1 } }
      name: { type: string, safety: safe }
      description: { type: "optional<string>", safety: safe }

  CreateProductRequest:
    fields:
      groupId: { type: string, safety: safe, validation: { min-length: 1 } }
      artifactId: { type: string, safety: safe, validation: { min-length: 1 } }
      name: { type: string, safety: safe }

  UpdateProductRequest:
    fields:
      name: { type: "optional<string>", safety: safe }
      description: { type: "optional<string>", safety: safe }

errors:
  ProductNotFound:
    code: NOT_FOUND
    safe-args:
      productId: string
  ProductAlreadyExists:
    code: CONFLICT
    safe-args:
      groupId: string
      artifactId: string

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
        errors: [ProductAlreadyExists]
      updateProduct:
        http: PUT /products/{productId}
        args:
          productId: { type: integer, param-type: path }
          request: { type: UpdateProductRequest, param-type: body }
        returns: Product
        errors: [ProductNotFound]
```

## Step 2: Generate Code

```bash
# Generate all three outputs
enchant build catalog-service.enchant.yml \
  --models --server --client \
  --output catalog/generated/

# Validate the definition
enchant check catalog-service.enchant.yml

# What gets generated:
# catalog/generated/models/catalog_types.py   → attrs classes for Product, CreateProductRequest, etc.
# catalog/generated/server/catalog_service.py  → CatalogServiceBase abstract class + Cherry router
# catalog/generated/client/catalog_client.py   → CatalogServiceClient using EnchantChannel
```

## Step 3: Implement the Server

```python
# catalog/app.py
from py_witchcraft_server.server.server import WitchcraftServer
from py_witchcraft_server.server.params import InitContext, RouterInfo
from py_witchcraft_server.config.install import InstallConfig
from catalog.service.impl import CatalogServiceImpl
from catalog.service.repo import ProductRepo

def init_app(ctx: InitContext, info: RouterInfo):
    # Initialize database
    repo = ProductRepo(db_url=ctx.install_config.extensions.get("db_url"))

    # Register service implementation
    service = CatalogServiceImpl(repo=repo)
    service.register(info.router)

    # Health checks
    db_health = ctx.health_reporter.initialize_health_component("DATABASE")
    db_health.healthy()

    def cleanup():
        repo.close()
    return cleanup

server = (
    WitchcraftServer()
    .with_init_func(init_app)
    .with_install_config_from_file("var/conf/install.yml")
    .with_runtime_config_from_file("var/conf/runtime.yml")
)
server.start()
```

```python
# catalog/service/impl.py
from catalog.generated.server.catalog_service import CatalogServiceBase
from catalog.generated.models.catalog_types import (
    Product, CreateProductRequest, UpdateProductRequest,
    ProductNotFound, ProductAlreadyExists,
)
from catalog.service.repo import ProductRepo
from catalog.service.models import ValidatedCreateProduct

class CatalogServiceImpl(CatalogServiceBase):
    def __init__(self, repo: ProductRepo):
        super().__init__()
        self._repo = repo

    async def list_products(self) -> list[Product]:
        return self._repo.list_all()

    async def get_product(self, product_id: int) -> Product:
        product = self._repo.get(product_id)
        if not product:
            raise ProductNotFound(product_id=str(product_id))
        return product

    async def create_product(self, request: CreateProductRequest) -> Product:
        # Validate with Rune
        validated = ValidatedCreateProduct.parse({
            "group_id": request.group_id,
            "artifact_id": request.artifact_id,
            "name": request.name,
        })
        existing = self._repo.find_by_coordinates(
            validated.group_id, validated.artifact_id
        )
        if existing:
            raise ProductAlreadyExists(
                group_id=validated.group_id,
                artifact_id=validated.artifact_id,
            )
        return self._repo.create(validated)

    async def update_product(
        self, product_id: int, request: UpdateProductRequest
    ) -> Product:
        product = self._repo.get(product_id)
        if not product:
            raise ProductNotFound(product_id=str(product_id))
        return self._repo.update(product_id, request)
```

## Step 4: Add Validation

```python
# catalog/service/models.py
from rune import inscribed, field
from typing import Annotated
from rune.constraints import MinLen, MaxLen, Matches

ProductCoordinate = Annotated[str, MinLen(1), MaxLen(255), Matches(r"^[a-z0-9.-]+$")]

@inscribed
class ValidatedCreateProduct:
    group_id: ProductCoordinate
    artifact_id: ProductCoordinate
    name: Annotated[str, MinLen(1), MaxLen(255)]

@inscribed
class ValidatedUpdateProduct:
    name: Annotated[str | None, MaxLen(255)] = None
    description: Annotated[str | None, MaxLen(2000)] = None
```

## Step 5: Add Health Checks

```python
# In init_app
import threading

def init_app(ctx: InitContext, info: RouterInfo):
    repo = ProductRepo(db_url="...")
    service = CatalogServiceImpl(repo=repo)
    service.register(info.router)

    # Database health check
    db_health = ctx.health_reporter.initialize_health_component("DATABASE")
    db_health.healthy()

    def check_db():
        while True:
            try:
                repo.ping()
                db_health.healthy()
            except Exception as e:
                db_health.error(e)
            import time; time.sleep(30)

    threading.Thread(target=check_db, daemon=True).start()

    # Migration health (if using Transmute)
    migration_health = ctx.health_reporter.initialize_health_component("MIGRATION")
    migration_health.healthy()

    return repo.close
```

## Step 6: Add Schema Migration

```python
# catalog/migrations/v001_add_description.py
import attrs
from semantic_version import Version
from transmute import Migration, MigrationMeta

@attrs.define
class CatalogV1:
    version: int = 1

@attrs.define
class CatalogV2:
    version: int = 2

class AddDescription(Migration[CatalogV1, CatalogV2]):
    meta = MigrationMeta(
        id="add-product-description",
        name="Add Product Description",
        description="Add description column to products table",
        schema_version=2,
        source_version=Version("1.0.0"),
        target_version=Version("1.1.0"),
        idempotency_mode="strict",
        owner_team="platform",
        primary_owner="platform@team.com",
    )
    source_schema = CatalogV1
    target_schema = CatalogV2

    def schema_additions(self):
        self.context.execute(
            "ALTER TABLE products ADD COLUMN IF NOT EXISTS description TEXT"
        )

    def rollback(self):
        self.context.execute(
            "ALTER TABLE products DROP COLUMN IF EXISTS description"
        )

    def migrate_batch(self, batch_size: int) -> bool:
        return False  # Additive — no data migration needed

    def schema_drops(self):
        pass  # Purely additive
```

```python
# catalog/migrations/registry.py
from transmute import MigrationGraph, MigrationRunner, SoakScheduler
from transmute.storage.sqlalchemy_backend import SQLAlchemyStorage
from datetime import timedelta
from .v001_add_description import AddDescription

def create_runner(engine):
    storage = SQLAlchemyStorage(engine)
    graph = MigrationGraph()
    context = MigrationContext(storage)
    graph.register(AddDescription(storage=storage, context=context))

    return MigrationRunner(
        storage=storage,
        graph=graph,
        node_id="catalog-service",
        soak_scheduler=SoakScheduler(storage,
            pre_execution_soak=timedelta(days=4),
            pre_finalization_soak=timedelta(days=4)),
        batch_size=1000,
        enforce_additive_first=True,
        require_finalization_approval=True,
    )
```

## Step 7: Wire Up the Client

The generated client can be packaged as a standalone Python package:

```python
# catalog-client/pyproject.toml would reference:
# catalog/generated/client/catalog_client.py
# catalog/generated/models/catalog_types.py

# Usage from another service:
from enchant_dialogue import EnchantChannel, ChannelConfig
from catalog.generated.client.catalog_client import CatalogServiceClient

channel = EnchantChannel(ChannelConfig(
    uris=["https://catalog.internal:8443"],
    auth_token_provider=lambda: get_service_token(),
    max_retries=3,
))
client = CatalogServiceClient(channel)

# Type-safe calls with AIMD concurrency control
products = client.list_products()
product = client.get_product(product_id=42)
new_product = client.create_product(CreateProductRequest(
    group_id="com.example",
    artifact_id="new-service",
    name="New Service",
))
```

## Step 8: Test Everything

```python
# tests/test_service.py
import pytest
from py_witchcraft_server.server.server import WitchcraftServer
import httpx

@pytest.fixture
def server():
    srv = WitchcraftServer().with_init_func(init_app).with_self_signed_certificate()
    info = srv.start_async()
    yield info
    srv.stop()
    srv.wait()

def test_create_and_get_product(server):
    client = httpx.Client(base_url=server.base_url, verify=False)

    # Create
    resp = client.post("/api/catalog/products", json={
        "groupId": "com.example",
        "artifactId": "test-service",
        "name": "Test Service",
    })
    assert resp.status_code == 201
    product = resp.json()
    assert product["name"] == "Test Service"

    # Get
    resp = client.get(f"/api/catalog/products/{product['id']}")
    assert resp.status_code == 200
    assert resp.json()["artifactId"] == "test-service"

def test_product_not_found(server):
    client = httpx.Client(base_url=server.base_url, verify=False)
    resp = client.get("/api/catalog/products/99999")
    assert resp.status_code == 404
    assert resp.json()["errorCode"] == "NOT_FOUND"

def test_duplicate_product(server):
    client = httpx.Client(base_url=server.base_url, verify=False)
    body = {"groupId": "com.example", "artifactId": "dup", "name": "Dup"}
    client.post("/api/catalog/products", json=body)
    resp = client.post("/api/catalog/products", json=body)
    assert resp.status_code == 409

# tests/test_models.py
from catalog.service.models import ValidatedCreateProduct
from rune import ValidationError

def test_valid_product():
    p = ValidatedCreateProduct.parse({
        "group_id": "com.example",
        "artifact_id": "my-service",
        "name": "My Service",
    })
    assert p.group_id == "com.example"

def test_invalid_coordinate():
    with pytest.raises(ValidationError):
        ValidatedCreateProduct.parse({
            "group_id": "INVALID UPPERCASE",
            "artifact_id": "my-service",
            "name": "My Service",
        })
```
