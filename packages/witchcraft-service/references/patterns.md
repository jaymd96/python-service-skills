# Witchcraft Service — Patterns & Gotchas

> Part of the witchcraft-service workflow. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [Contract-First Development](#contract-first-development)
- [Validation Strategy](#validation-strategy)
- [Migration Strategy](#migration-strategy)
- [Client Packaging](#client-packaging)
- [Testing Strategy](#testing-strategy)
- [Observability Patterns](#observability-patterns)
- [Gotchas](#gotchas)

## Contract-First Development

The Enchant DSL is the single source of truth for API contracts:

```
1. Define types, errors, services in .enchant.yml
2. Generate code (never edit generated files)
3. Implement abstract base class methods
4. Push constraints into the DSL where possible
5. Use Rune for business logic validation beyond DSL capabilities
```

### When to use DSL validation vs. Rune

| Rule | Where | Why |
|------|-------|-----|
| Field required/optional | `.enchant.yml` | Wire format enforcement |
| String min/max length | `.enchant.yml` | Client-side validation too |
| Enum values | `.enchant.yml` | Forward-compatible enums |
| Regex pattern match | Rune `@inscribed` | Complex patterns, DSL doesn't support regex |
| Cross-field validation | Rune `@inscribed` | DSL is per-field only |
| Business rules | Service implementation | Domain logic, not contract |

## Validation Strategy

### Three validation layers

```
Layer 1: Wire format (Enchant-generated deserializer)
  - Required fields present
  - Types correct
  - DSL constraints (min-length, etc.)

Layer 2: Business rules (Rune @inscribed)
  - Cross-field validation
  - Pattern matching
  - Domain-specific constraints

Layer 3: Domain logic (Service implementation)
  - Uniqueness checks (requires DB query)
  - Authorization checks
  - State machine transitions
```

### Pattern: Enchant at the boundary, Rune internally

```python
class CatalogServiceImpl(CatalogServiceBase):
    async def create_product(self, request: CreateProductRequest) -> Product:
        # Layer 1: Already passed (Enchant deserializer handled it)

        # Layer 2: Rune validation
        validated = ValidatedCreateProduct.parse({
            "group_id": request.group_id,
            "artifact_id": request.artifact_id,
            "name": request.name,
        })

        # Layer 3: Domain logic
        if self._repo.exists(validated.group_id, validated.artifact_id):
            raise ProductAlreadyExists(...)

        return self._repo.create(validated)
```

## Migration Strategy

### When to create a Transmute migration

| Change | Migration needed? | Migration type |
|--------|------------------|----------------|
| Add optional field to type | Yes — add column | Additive (no `schema_drops`) |
| Add required field to type | Yes — add column + backfill | Full (with `migrate_batch`) |
| Remove field from type | Yes — drop column | Full (with `schema_drops`) |
| Rename field | Yes — add new + migrate + drop old | Full migration |
| Change field type | Yes — add new column + transform + drop old | Full migration |
| Add new type | Maybe — if it needs a table | Additive |
| Add new endpoint | No — endpoints are stateless | N/A |

### Migration + contract update sequence

```
1. Create Transmute migration (schema_additions adds new column)
2. Update .enchant.yml (add optional field to type)
3. Regenerate code (enchant build)
4. Update implementation to use new field
5. Deploy — migration runs, dual-write active
6. After soak period, approve finalization (schema_drops removes old column)
```

### Dual-write proxy pattern

```python
# During migration: reads from old schema, writes to both
proxy = MigrationAwareProxy.wrap(
    interface=ProductRepo,
    old_impl=ProductRepoV1(engine),
    new_impl=ProductRepoV2(engine),
    migration_id="add-product-description",
    state_provider=storage,
)

# Use proxy in service implementation
service = CatalogServiceImpl(repo=proxy)
```

## Client Packaging

Generated clients can be distributed as standalone Python packages:

### Package structure

```
catalog-client/
  pyproject.toml
  catalog_client/
    __init__.py           # Re-exports from generated code
    models.py             # Copy of generated models
    client.py             # Copy of generated client
```

### pyproject.toml

```toml
[project]
name = "catalog-client"
version = "1.0.0"
dependencies = [
    "enchant-dialogue>=0.1.0",
    "attrs>=23.0",
    "cattrs>=23.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

### Usage by consumers

```python
from catalog_client import CatalogServiceClient
from enchant_dialogue import EnchantChannel, ChannelConfig

channel = EnchantChannel(ChannelConfig(
    uris=["https://catalog.internal:8443"],
    auth_token_provider=get_token,
))
client = CatalogServiceClient(channel)
products = client.list_products()
```

## Testing Strategy

### Unit tests (no server needed)

```python
# Test Rune models
def test_valid_model():
    p = ValidatedCreateProduct.parse({"group_id": "com.example", ...})
    assert p.group_id == "com.example"

# Test service logic with mock repo
def test_create_product():
    repo = MockProductRepo()
    service = CatalogServiceImpl(repo=repo)
    result = await service.create_product(CreateProductRequest(...))
    assert result.name == "expected"
```

### Integration tests (with Cherry server)

```python
@pytest.fixture
def server():
    srv = WitchcraftServer().with_init_func(init_app).with_self_signed_certificate()
    info = srv.start_async()
    yield info
    srv.stop(); srv.wait()

def test_endpoint(server):
    resp = httpx.get(f"{server.base_url}/api/catalog/products", verify=False)
    assert resp.status_code == 200
```

### Contract conformance tests

```bash
# Verify generated code matches contract
enchant verify catalog-service.enchant.yml \
  --test-suite tests/conformance/
```

### Migration tests

```python
from transmute import InMemoryStorage

def test_migration():
    storage = InMemoryStorage()
    runner = create_runner_with_storage(storage)
    runner.run_until_idle()
    assert storage.get_state("add-product-description").name == "FINISHED"
```

## Observability Patterns

### Structured logging in handlers

```python
from py_witchcraft_server.logging import svc1log

class CatalogServiceImpl(CatalogServiceBase):
    async def create_product(self, request):
        logger = svc1log.from_context()
        logger.info("Creating product",
            svc1log.safe_param("group_id", request.group_id),
            svc1log.safe_param("artifact_id", request.artifact_id))

        product = self._repo.create(request)

        logger.info("Product created",
            svc1log.safe_param("product_id", product.id))
        return product
```

### Migration event subscribers

```python
from transmute.signals.events import migration_failed, migration_completed

@migration_failed.connect
def on_migration_failed(sender, migration_id, error, **kwargs):
    logger = svc1log.from_context()
    logger.error("Migration failed",
        svc1log.safe_param("migration_id", migration_id),
        svc1log.unsafe_param("error", str(error)))

@migration_completed.connect
def on_migration_completed(sender, migration_id, **kwargs):
    migration_health.healthy()
```

### Custom metrics

```python
def init_app(ctx, info):
    products_created = ctx.metrics_registry.counter(
        "catalog.products.created", {"service": "catalog"})

    class CatalogServiceImpl(CatalogServiceBase):
        async def create_product(self, request):
            product = self._repo.create(request)
            products_created.inc()
            return product
```

## Gotchas

1. **Never edit generated code**: Files in `generated/` are overwritten on every `enchant build`. Put all implementation in separate files
2. **Enchant models are attrs classes**: They're compatible with Rune's `@inscribed` because both use attrs/cattrs. But don't mix — use Enchant types at the API boundary, Rune types internally
3. **Safety annotations are enforced**: If you mark a field as `unsafe` in `.enchant.yml`, Cherry's structlog will redact it in production. Don't mark internal IDs as `unsafe` — that makes debugging impossible
4. **Forward compatibility by default**: Enchant clients tolerate unknown fields, enum values, and union variants. Don't rely on exhaustive enum matching on the client side
5. **Contract changes require regeneration**: After editing `.enchant.yml`, always run `enchant build` before testing. Stale generated code causes subtle type mismatches
6. **Transmute migrations are separate from Enchant changes**: A new field in `.enchant.yml` doesn't automatically create a migration. You must create the Transmute migration explicitly
7. **Health component names are uppercase**: Cherry's `initialize_health_component("DATABASE")` requires uppercase names matching `^[A-Z_]+$`
8. **Cherry is HTTPS-only**: Use `.with_self_signed_certificate()` for dev. Plain HTTP is not supported
9. **Rune `.parse()` vs `.validate()`**: `.parse()` coerces types (string "42" → int 42). `.validate()` is strict — type must already match. Use `.parse()` at API boundaries
10. **Soak times exist for a reason**: Transmute's 4-day default soak periods catch issues under real traffic. Only skip with `--skip-soak --reason "P0 incident"`
