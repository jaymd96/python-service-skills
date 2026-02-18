---
name: transmute
description: Zero-downtime online schema migration framework. Use when defining database migrations with 8-state lifecycle, dual-write routing, soak times, finalization approvals, DAG dependencies, dead-letter queues, hash-chained audit trails, or release checkpoints. Triggers on transmute, schema migration, dual-write, soak time, MigrationRunner, MigrationGraph, finalization, dead-letter queue, online migration.
---

# Transmute — Online Schema Migration (v0.1.0)

## Quick Start

```bash
pip install transmute
```

```python
from transmute import Migration, MigrationMeta, MigrationRunner
from transmute.storage.sqlalchemy_backend import SQLAlchemyStorage
from semantic_version import Version

class AddColumn(Migration[SchemaV1, SchemaV2]):
    meta = MigrationMeta(
        id="add-user-email",
        name="Add email column",
        source_version=Version("1.0.0"),
        target_version=Version("1.1.0"),
    )
    def schema_additions(self):
        self.context.execute("ALTER TABLE users ADD COLUMN IF NOT EXISTS email TEXT")
    def rollback(self):
        self.context.execute("ALTER TABLE users DROP COLUMN IF EXISTS email")
    def migrate_batch(self, batch_size: int) -> bool:
        return False  # No data migration needed
    def schema_drops(self):
        pass  # No old schema to drop
```

## Key Patterns

### 8-State lifecycle
```
UNINITIALIZED -> INITIALIZING (schema_additions) -> RUNNING (migrate_batch)
-> AWAITING_FINALIZATION (soak + approval) -> FINISHING (schema_drops) -> FINISHED
  Any pre-FINISHING state -> ROLLING_BACK -> UNINITIALIZED
```

### Runner setup
```python
from transmute import MigrationRunner, MigrationGraph, SoakScheduler
from transmute.storage.sqlalchemy_backend import SQLAlchemyStorage

storage = SQLAlchemyStorage(engine)
graph = MigrationGraph()
graph.register(my_migration)

runner = MigrationRunner(storage=storage, graph=graph, node_id="node-1",
    soak_scheduler=SoakScheduler(storage, pre_execution_soak=timedelta(days=4)))
runner.run_until_idle()
```

## References

- **[overview.md](references/overview.md)** — Full Transmute framework overview, architecture, API quick reference, Apollo integration examples
- **[api.md](references/api.md)** — Migration, MigrationMeta, MigrationRunner, MigrationGraph, Storage, Proxy, Signals
- **[lifecycle.md](references/lifecycle.md)** — 8-state FSM, dual-write routing, soak times, finalization, rollback, DLQ, audit
- **[examples.md](references/examples.md)** — Complete migration examples, runner factory, CLI reference, gotchas

## Grep Patterns

- `Migration\[|MigrationMeta` — Find migration definitions
- `MigrationRunner|MigrationGraph` — Find runner/graph setup
- `schema_additions|migrate_batch|schema_drops` — Find migration lifecycle methods
