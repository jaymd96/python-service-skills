# Transmute — Examples & Gotchas

> Part of the transmute skill. See [SKILL.md](../SKILL.md) for overview.

## Complete Migration Example

```python
# apollo/migrations/v001_add_canary_state.py
import attrs
from semantic_version import Version
from transmute import Migration, MigrationMeta

@attrs.define
class CatalogV1:
    version: int = 1

@attrs.define
class CatalogV2:
    version: int = 2

class AddCanaryState(Migration[CatalogV1, CatalogV2]):
    meta = MigrationMeta(
        id="add-canary-state",
        name="Add Canary State Table",
        description="Add canary tables for progressive rollouts",
        schema_version=2,
        source_version=Version("1.0.0"),
        target_version=Version("1.1.0"),
        idempotency_mode="strict",
        owner_team="platform-team",
        primary_owner="platform@apollo.dev",
    )
    source_schema = CatalogV1
    target_schema = CatalogV2

    def schema_additions(self):
        self.context.execute("""
            CREATE TABLE IF NOT EXISTS canary_state (
                id INTEGER PRIMARY KEY, entity_id INTEGER NOT NULL,
                status TEXT NOT NULL DEFAULT 'not_started',
                canary_percentage REAL NOT NULL DEFAULT 0.0
            )
        """)

    def rollback(self):
        self.context.execute("DROP TABLE IF EXISTS canary_state")

    def migrate_batch(self, batch_size: int) -> bool:
        return False  # New tables — no data migration

    def schema_drops(self):
        pass  # Purely additive
```

## Migration Registry

```python
# apollo/migrations/registry.py
from transmute import MigrationGraph
from .v001_add_canary_state import AddCanaryState

def build_migration_graph(storage):
    graph = MigrationGraph()
    context = ApolloMigrationContext(storage)
    graph.register(AddCanaryState(storage=storage, context=context))
    return graph
```

## Runner Factory

```python
# apollo/migrations/runner.py
from datetime import timedelta
from transmute import MigrationRunner, SoakScheduler
from transmute.storage.sqlalchemy_backend import SQLAlchemyStorage

def create_runner(engine, node_id="local"):
    storage = SQLAlchemyStorage(engine)
    graph = build_migration_graph(storage)
    return MigrationRunner(
        storage=storage, graph=graph, node_id=node_id,
        soak_scheduler=SoakScheduler(storage,
            pre_execution_soak=timedelta(days=4),
            pre_finalization_soak=timedelta(days=4)),
        batch_size=1000,
        enforce_additive_first=True,
        require_finalization_approval=True,
    )
```

## Signal Subscribers

```python
from transmute.signals.events import migration_failed, migration_completed

@migration_failed.connect
def alert_on_failure(sender, migration_id, error, **kwargs):
    pagerduty.trigger(f"Migration {migration_id} failed: {error}")

@migration_completed.connect
def notify_completion(sender, migration_id, **kwargs):
    slack.post(f"Migration {migration_id} completed successfully")
```

## CLI Reference

```bash
# Status
transmute status
transmute status --json-output --include-metrics

# Run
transmute run                                    # All pending
transmute run my-migration-id                    # Specific
transmute run --dry-run                          # Preview
transmute run --skip-soak --reason "P0 incident" # Emergency

# Rollback
transmute rollback my-migration --plan           # Preview
transmute rollback my-migration --confirm        # Execute

# Finalization
transmute finalization approve my-migration --by ops@team --reason "verified"
transmute finalization revoke my-migration --by admin --reason "issue found"

# Dead-letter queue
transmute dlq list my-migration
transmute dlq replay my-migration --record-ids id1,id2
transmute dlq purge my-migration --status replayed

# Audit
transmute audit list --migration-id my-migration
transmute audit verify-chain

# Checkpoints
transmute checkpoint create 2.0.0 --description "Apollo v2.0.0"
transmute checkpoint can-upgrade 3.0.0

# Scaffold
transmute create my-new-migration --output apollo/migrations/
```

## Gotchas

1. **`schema_drops()` is the POINT OF NO RETURN**: Once called, old schema is gone. Always use soak times and finalization approval
2. **Always implement `rollback()`**: Even if it's just `pass`, but ideally undo `schema_additions()`
3. **Use IF NOT EXISTS / IF EXISTS guards**: Both `schema_additions()` and `rollback()` must be idempotent
4. **One migration per schema change**: Keep migrations small and focused for predictable rollback
5. **Soak times exist for a reason**: 4-day defaults catch issues under real traffic. Only skip for genuine emergencies
6. **Dependencies are a DAG**: `depends_on` creates edges in a NetworkX DAG — cycles are rejected
7. **attrs/cattrs foundation**: Source/target schemas are attrs classes, compatible with Rune
