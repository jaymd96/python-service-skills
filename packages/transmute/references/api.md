# Transmute — Core API Reference

> Part of the transmute skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [Migration Base Class](#migration-base-class)
- [SimpleMigration](#simplemigration)
- [MigrationMeta](#migrationmeta)
- [MigrationRunner](#migrationrunner)
- [MigrationGraph](#migrationgraph)
- [CheckpointManager](#checkpointmanager)
- [MigrationAwareProxy](#migrationawareproxy)
- [Storage Backends](#storage-backends)
- [Signals](#signals)
- [Invariants](#invariants)
- [Pre-Built Templates](#pre-built-templates)

## Migration Base Class

```python
from transmute import Migration, MigrationMeta

class MyMigration(Migration[SourceSchema, TargetSchema]):
    meta = MigrationMeta(...)
    source_schema = SourceSchema
    target_schema = TargetSchema

    def schema_additions(self) -> None:
        """Create new schema elements. Must be idempotent."""
        self.context.execute("ALTER TABLE ... ADD COLUMN IF NOT EXISTS ...")

    def rollback(self) -> None:
        """Undo schema_additions. Must be idempotent."""
        self.context.execute("ALTER TABLE ... DROP COLUMN IF EXISTS ...")

    def migrate_batch(self, batch_size: int) -> bool:
        """Process one batch. Return True if more work remains."""
        return has_more

    def schema_drops(self) -> None:
        """Drop old schema. IRREVERSIBLE. Only called after soak + approval."""
        self.context.execute("ALTER TABLE ... DROP COLUMN ...")
```

## SimpleMigration

Helper with record-level iteration:

```python
from transmute import SimpleMigration

class SplitName(SimpleMigration[UserV1, UserV2]):
    meta = MigrationMeta(...)

    def iterate_source_records(self, after_key, limit) -> Iterator[tuple[str, UserV1]]:
        """Yield (key, source_record) in stable key order."""

    def write_target_record(self, key: str, record: UserV2) -> None:
        """Persist one transformed target record."""

    def transform(self, record: UserV1) -> UserV2:
        """Transform source to target."""
```

## MigrationMeta

```python
MigrationMeta(
    id="unique-migration-id",
    name="Human-Readable Name",
    description="What this migration does",
    schema_version=2,
    source_version=Version("1.0.0"),
    target_version=Version("2.0.0"),
    depends_on=("prior-migration-id",),       # DAG dependencies
    requires_external_action=False,             # Manual gate
    estimated_duration_seconds=300,
    idempotency_mode="strict",                  # strict | best_effort | manual_exception
    owner_team="platform-team",
    primary_owner="migrations@example.com",
    runbook_ref="https://wiki/runbooks/...",
    escalation_ref="https://pagerduty/...",
    invariants=(...),                           # InvariantSpec tuple
)
```

## MigrationRunner

```python
from transmute import MigrationRunner, SoakScheduler

runner = MigrationRunner(
    storage=storage, graph=graph, node_id="node-1",
    soak_scheduler=SoakScheduler(storage,
        pre_execution_soak=timedelta(days=4),
        pre_finalization_soak=timedelta(days=4)),
    batch_size=1000, max_retries=3,
    retry_delay=timedelta(seconds=30),
    enforce_idempotency=True,
    enforce_additive_first=True,
    require_finalization_approval=True,
    require_preflight_pass=True,
)
```

### Runner Methods

| Method | Description |
|--------|-------------|
| `preflight(migration_id)` | Run preflight checks |
| `run_once()` | Execute one step, return bool |
| `run_until_idle(max_steps)` | Execute until no work remains |
| `start()` / `stop(timeout)` | Background loop |
| `dry_run(migration_id)` | Preview without mutations |
| `rollback_plan(id)` | Preview rollback impact |
| `rollback(id)` | Execute rollback |
| `approve_finalization(id, approved_by, reason, ttl)` | Approve schema_drops |
| `revoke_finalization_approval(id, actor_id, reason)` | Revoke approval |
| `skip_soak(id, reason, actor_id)` | Emergency skip |
| `list_dead_letters(id, status)` | List DLQ records |
| `replay_dead_letters(id, record_ids)` | Retry failed records |
| `purge_dead_letters(id, status)` | Clean DLQ |
| `status()` | `{migration_id: MigrationState}` |
| `metric_snapshot()` | Current metrics |

## MigrationGraph

```python
from transmute import MigrationGraph

graph = MigrationGraph()
graph.register(migration_a)
graph.register(migration_b)  # depends_on=("migration-a",)

graph.get_migration("migration-a")
graph.get_dependencies("migration-b")     # ("migration-a",)
graph.get_dependents("migration-a")       # ("migration-b",)
graph.get_execution_order()               # ["migration-a", "migration-b"]
graph.get_parallel_groups()               # [{"migration-a"}, {"migration-b"}]
```

## CheckpointManager

```python
from transmute import CheckpointManager

manager = CheckpointManager(storage, graph)
cp = manager.create_checkpoint("2.0.0", description="Apollo v2.0.0 release")
can, reason = manager.can_upgrade_to(Version("3.0.0"))
can, reason = manager.can_downgrade_to(Version("1.5.0"))
current = manager.get_current_version()
```

## MigrationAwareProxy

Dual-write routing based on migration state:

```python
from transmute import MigrationAwareProxy, Read, Write, WriteStrategy

class UserRepository:
    @Read
    def get_user(self, user_id: str) -> User: ...

    @Write(strategy=WriteStrategy.BOTH)
    def save_user(self, user: User) -> None: ...

proxy = MigrationAwareProxy.wrap(
    interface=UserRepository,
    old_impl=OldUserRepo(), new_impl=NewUserRepo(),
    migration_id="split-user-name", state_provider=storage,
)
proxy.get_user("123")    # Routes based on migration state
proxy.save_user(user)    # Dual-writes during RUNNING state
```

WriteStrategy: `BOTH`, `OLD_ONLY`, `NEW_ONLY`, `CUSTOM`

## Storage Backends

```python
from transmute import InMemoryStorage                         # Tests
from transmute.storage.sqlalchemy_backend import SQLAlchemyStorage  # Production

storage = SQLAlchemyStorage(engine)  # Auto-creates 8 tables
```

| Table | Purpose |
|-------|---------|
| `transmute_migrations` | State, timestamps, retry counts |
| `transmute_migration_progress` | Key-value checkpoint with hash integrity |
| `transmute_finalization_approvals` | Approval records with TTL |
| `transmute_nodes` | Distributed node heartbeats, leader election |
| `transmute_checkpoints` | Release version checkpoints |
| `transmute_locks` | Migration-level distributed locks |
| `transmute_dead_letters` | Failed record tracking |
| `transmute_audit_events` | Hash-chained immutable audit trail |

## Signals

```python
from transmute.signals.events import (
    migration_started, migration_progress, migration_state_changed,
    migration_completed, migration_failed,
)

@migration_failed.connect
def on_failed(sender, migration_id, error, state, **kwargs):
    alert(f"Migration {migration_id} failed: {error}")
```

## Invariants

```python
from transmute.core.invariants import InvariantSpec, InvariantPhase

InvariantSpec(
    name="backfill_complete",
    phase=InvariantPhase.PRE_FINALIZATION,  # PRE_RUN | DURING_RUN | PRE_FINALIZATION | POST_FINALIZATION
    params={"check": check_fn},
)
```

## Pre-Built Templates

```python
from transmute.templates import (
    AddColumnMigration, AddTableMigration, DropColumnMigration,
    RenameColumnMigration, TransformColumnMigration,
)
```

### SQLAlchemy Contrib

```python
from transmute.contrib.sqlalchemy import (
    SQLAlchemyMigration, diff_sqlalchemy_models, generate_migration_code,
)
```
