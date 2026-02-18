# Transmute — Online Schema Migration Framework for Apollo-on-k3s

## Overview

**Transmute** is an embedded, zero-downtime schema and data migration framework for Python services. It provides an 8-state lifecycle with safety gates (soak times, finalization approvals, preflight checks), dual-write routing, DAG-based dependency ordering, dead-letter queues, and hash-chained audit trails.

| Property | Value |
|----------|-------|
| **Package** | `transmute` |
| **Version** | 0.1.0 |
| **License** | MIT |
| **Python** | >=3.13 |
| **Dependencies** | `attrs`, `cattrs`, `sqlalchemy>=2.0`, `transitions`, `blinker`, `networkx`, `semantic-version`, `click` |
| **Repository** | witchcraft/transmute |

### Why Transmute for Apollo-on-k3s

| Capability | Apollo Use Case |
|------------|-----------------|
| 8-state lifecycle | Safe, staged schema evolution for 30+ catalog tables |
| Dual-write routing | Zero-downtime migration of live product catalog |
| Soak times | 4-day default wait before irreversible schema drops |
| DAG dependencies | Ordered execution of multi-table migrations |
| Finalization approval | Manual gate before destructive schema changes |
| Dead-letter queue | Capture and replay failed record migrations |
| Audit trail | Hash-chained, immutable record of all migration actions |
| Checkpoints | Tie migrations to release versions, prevent unsafe downgrades |
| Pre-built templates | AddColumn, DropColumn, RenameColumn, TransformColumn |
| CLI | Operator interface for status, run, rollback, DLQ management |

### Installation

```bash
pip install transmute
```

---

## Project Integration

### Location in Project

```
apollo/
├── migrations/
│   ├── __init__.py
│   ├── registry.py              # MigrationGraph registration
│   ├── runner.py                # MigrationRunner factory
│   ├── v001_add_canary_state.py # Example migration
│   ├── v002_split_config.py     # Example migration
│   └── v003_add_audit_events.py # Example migration
├── catalog/
│   ├── _database.py             # Engine creation (shared with Transmute storage)
│   └── _tables.py               # SQLAlchemy 2.0 ORM models
```

### How We'll Use It

1. **Schema Evolution** — Each catalog schema change is a `Migration` subclass
2. **Staged Rollout** — Soak times prevent premature finalization
3. **Safe Rollback** — Rollback from any pre-finalization state
4. **Release Checkpoints** — Tie migrations to Apollo release versions
5. **Operator Control** — CLI for status, dry-run, finalization approval

---

## Architecture

### 8-State Lifecycle

```
Register migration
        │
        ▼
UNINITIALIZED ◄──────────────────────┐
        │ initialize                  │
        ▼                             │ rollback
INITIALIZING  (schema_additions)      │ (from any pre-FINISHING state)
        │ start                       │
        ▼                             │
   RUNNING  (migrate_batch loop)      │
        │                             │
        ├─► AWAITING_ACTION           │
        │   (external gate)           │
        │                             │
        ▼ await_finalization          │
AWAITING_FINALIZATION ────────────────┘
        │ (soak time + approval)
        ▼
  FINISHING  (schema_drops — POINT OF NO RETURN)
        │
        ▼
   FINISHED
```

### State-Based Read/Write Routing

| Migration State | Read From | Write To |
|-----------------|-----------|----------|
| UNINITIALIZED, INITIALIZING, ROLLING_BACK | old | old |
| RUNNING, AWAITING_ACTION, AWAITING_FINALIZATION | old | old + new (dual-write) |
| FINISHING, FINISHED | new | new |

### Internal Dependency Map

```
┌─────────────────────────────────────────────────┐
│  CLI Layer                          click        │
├─────────────────────────────────────────────────┤
│  Observability                      blinker      │
├─────────────────────────────────────────────────┤
│  Orchestration        transitions + networkx     │
├─────────────────────────────────────────────────┤
│  Data Layer                   attrs + cattrs     │
├─────────────────────────────────────────────────┤
│  Persistence                  sqlalchemy 2.0     │
├─────────────────────────────────────────────────┤
│  Metadata                    semantic-version    │
└─────────────────────────────────────────────────┘
```

---

## API Quick Reference (llms.txt style)

### Core Imports

```python
from transmute import (
    # Migration base classes
    Migration,            # Abstract base: schema_additions, migrate_batch, schema_drops, rollback
    SimpleMigration,      # Helper: iterate_source_records + write_target_record
    MigrationMeta,        # Metadata: id, versions, dependencies, owner, invariants
    MigrationContext,     # Protocol: execute(statement, **params)

    # State machine
    MigrationState,       # Enum: 8 states
    MigrationStateInfo,   # Dataclass: state + timestamps + progress
    MigrationStateMachine,# FSM with guarded transitions

    # Execution
    MigrationRunner,      # Orchestrates migrations with safety gates
    SoakScheduler,        # Manages pre-execution and pre-finalization soak times

    # Routing
    MigrationAwareProxy,  # Routes reads/writes based on migration state
    Read,                 # Decorator: mark method as read operation
    Write,                # Decorator: mark method as write operation
    WriteStrategy,        # Enum: BOTH, OLD_ONLY, NEW_ONLY, CUSTOM

    # Graph
    MigrationGraph,       # DAG registry for dependency ordering
    Checkpoint,           # Release checkpoint
    CheckpointManager,    # Create and enforce checkpoints

    # Storage
    InMemoryStorage,      # For tests
)

from transmute.storage.sqlalchemy_backend import SQLAlchemyStorage  # For production
from transmute.signals.events import (
    migration_started,
    migration_progress,
    migration_state_changed,
    migration_completed,
    migration_failed,
)
```

### Migration Definition

```python
from transmute import Migration, MigrationMeta
from semantic_version import Version

class MyMigration(Migration[SourceSchema, TargetSchema]):
    """Subclass Migration to define a schema change."""

    meta = MigrationMeta(
        id="my-migration-id",
        name="Human-Readable Name",
        description="What this migration does",
        schema_version=2,
        source_version=Version("1.0.0"),
        target_version=Version("2.0.0"),
        depends_on=("prior-migration-id",),      # DAG dependencies
        requires_external_action=False,            # Manual gate
        estimated_duration_seconds=300,
        idempotency_mode="strict",                 # strict | best_effort | manual_exception
        owner_team="platform-team",
        primary_owner="migrations@example.com",
        runbook_ref="https://wiki/runbooks/my-migration",
        escalation_ref="https://pagerduty/my-service",
    )

    source_schema = SourceSchema
    target_schema = TargetSchema

    def schema_additions(self) -> None:
        """Create new schema elements. Must be idempotent."""
        self.context.execute("ALTER TABLE users ADD COLUMN IF NOT EXISTS first_name TEXT")

    def rollback(self) -> None:
        """Undo schema_additions. Must be idempotent."""
        self.context.execute("ALTER TABLE users DROP COLUMN IF EXISTS first_name")

    def migrate_batch(self, batch_size: int) -> bool:
        """Process one batch. Return True if more work remains."""
        # ... iterate and transform records ...
        return has_more

    def schema_drops(self) -> None:
        """Drop old schema. Only called after soak time. IRREVERSIBLE."""
        self.context.execute("ALTER TABLE users DROP COLUMN full_name")
```

### SimpleMigration (Helper)

```python
from transmute import SimpleMigration
from typing import Iterator

class SplitUserName(SimpleMigration[UserV1, UserV2]):
    meta = MigrationMeta(...)
    source_schema = UserV1
    target_schema = UserV2

    def schema_additions(self) -> None:
        self.context.execute("ALTER TABLE users ADD COLUMN first_name TEXT")
        self.context.execute("ALTER TABLE users ADD COLUMN last_name TEXT")

    def rollback(self) -> None:
        self.context.execute("ALTER TABLE users DROP COLUMN IF EXISTS first_name")
        self.context.execute("ALTER TABLE users DROP COLUMN IF EXISTS last_name")

    def iterate_source_records(
        self, after_key: str | None, limit: int
    ) -> Iterator[tuple[str, UserV1]]:
        """Yield (key, source_record) in stable key order."""
        # Query source table, yield records
        ...

    def write_target_record(self, key: str, record: UserV2) -> None:
        """Persist one transformed target record."""
        # UPDATE users SET first_name=?, last_name=? WHERE id=?
        ...

    def transform(self, record: UserV1) -> UserV2:
        """Transform source to target."""
        first, last = record.full_name.split(" ", 1)
        return UserV2(id=record.id, first_name=first, last_name=last)

    def schema_drops(self) -> None:
        self.context.execute("ALTER TABLE users DROP COLUMN full_name")
```

### MigrationRunner

```python
from transmute import MigrationRunner, MigrationGraph, SoakScheduler
from transmute.storage.sqlalchemy_backend import SQLAlchemyStorage

# Setup
storage = SQLAlchemyStorage(engine)
graph = MigrationGraph()
scheduler = SoakScheduler(
    storage,
    pre_execution_soak=timedelta(days=4),
    pre_finalization_soak=timedelta(days=4),
)

runner = MigrationRunner(
    storage=storage,
    graph=graph,
    node_id="node-1",
    soak_scheduler=scheduler,
    batch_size=1000,
    max_retries=3,
    retry_delay=timedelta(seconds=30),
    enforce_idempotency=True,
    enforce_additive_first=True,
    require_finalization_approval=True,
    require_preflight_pass=True,
)
```

**Runner Methods:**

```python
# Preflight checks
report = runner.preflight(migration_id="my-migration")

# Execute one step
did_work = runner.run_once()

# Execute until idle
steps = runner.run_until_idle(max_steps=1000)

# Background loop
runner.start()
runner.stop(timeout=30.0)

# Dry run (no mutations)
plan = runner.dry_run(migration_id="my-migration")

# Rollback
impact = runner.rollback_plan("my-migration")    # Preview
runner.rollback("my-migration")                   # Execute

# Finalization approval
runner.approve_finalization(
    "my-migration",
    approved_by="ops@team.com",
    reason="Verified data consistency",
    ttl=timedelta(hours=24),
)
runner.revoke_finalization_approval("my-migration", actor_id="admin", reason="...")

# Skip soak (emergency)
runner.skip_soak("my-migration", reason="P0 incident", actor_id="oncall")

# Dead-letter queue
records = runner.list_dead_letters("my-migration", status="open")
replayed = runner.replay_dead_letters("my-migration", record_ids=["id1", "id2"])
purged = runner.purge_dead_letters("my-migration", status="replayed")

# Status and metrics
states = runner.status()        # {migration_id: MigrationState}
metrics = runner.metric_snapshot()
```

### MigrationGraph (DAG)

```python
from transmute import MigrationGraph

graph = MigrationGraph()

# Register migrations (edges created from meta.depends_on)
graph.register(migration_a)
graph.register(migration_b)  # depends_on=("migration-a",)

# Query
graph.get_migration("migration-a")
graph.get_dependencies("migration-b")     # ("migration-a",)
graph.get_dependents("migration-a")       # ("migration-b",)
graph.get_execution_order()               # ["migration-a", "migration-b"]
graph.get_parallel_groups()               # [{"migration-a"}, {"migration-b"}]

# Iterate in order
for migration in graph.iter_migrations():
    print(migration.meta.id)
```

### Checkpoints

```python
from transmute import CheckpointManager

manager = CheckpointManager(storage, graph)

# Create checkpoint tied to release
cp = manager.create_checkpoint("2.0.0", description="Apollo v2.0.0 release")

# Check upgrade/downgrade safety
can, reason = manager.can_upgrade_to(Version("3.0.0"))
can, reason = manager.can_downgrade_to(Version("1.5.0"))

# Current version
current = manager.get_current_version()
```

### Proxy (Dual-Write Routing)

```python
from transmute import MigrationAwareProxy, Read, Write, WriteStrategy

# Define interface
class UserRepository:
    @Read
    def get_user(self, user_id: str) -> User: ...

    @Write(strategy=WriteStrategy.BOTH)
    def save_user(self, user: User) -> None: ...

# Create proxy
proxy = MigrationAwareProxy.wrap(
    interface=UserRepository,
    old_impl=OldUserRepo(),
    new_impl=NewUserRepo(),
    migration_id="split-user-name",
    state_provider=storage,
)

# Proxy routes automatically based on migration state
proxy.get_user("123")      # Reads from old or new depending on state
proxy.save_user(user)      # Writes to both during RUNNING state
```

### Signals (Observability)

```python
from transmute.signals.events import (
    migration_started,
    migration_progress,
    migration_state_changed,
    migration_completed,
    migration_failed,
)

# Subscribe to events
@migration_started.connect
def on_started(sender, migration_id, state, **kwargs):
    print(f"Migration {migration_id} started in state {state}")

@migration_progress.connect
def on_progress(sender, migration_id, records_processed, estimated_total, **kwargs):
    pct = (records_processed / estimated_total * 100) if estimated_total else 0
    print(f"Migration {migration_id}: {pct:.1f}% complete")

@migration_failed.connect
def on_failed(sender, migration_id, error, state, **kwargs):
    alert(f"Migration {migration_id} failed: {error}")
```

### Storage Backends

```python
# In-memory (for tests)
from transmute import InMemoryStorage
storage = InMemoryStorage()

# SQLAlchemy (for production)
from transmute.storage.sqlalchemy_backend import SQLAlchemyStorage
storage = SQLAlchemyStorage(engine)  # Creates tables automatically
```

**Storage Tables (SQLAlchemy 2.0 Mapped types):**

| Table | Purpose |
|-------|---------|
| `transmute_migrations` | Migration state, timestamps, retry counts |
| `transmute_migration_progress` | Key-value checkpoint store with hash integrity |
| `transmute_finalization_approvals` | Approval records with optional TTL |
| `transmute_nodes` | Distributed node heartbeats and leader election |
| `transmute_checkpoints` | Release version checkpoints |
| `transmute_locks` | Migration-level distributed locks |
| `transmute_dead_letters` | Failed record tracking with retry counts |
| `transmute_audit_events` | Hash-chained, immutable audit trail |

### Pre-Built Templates

```python
from transmute.templates import (
    AddColumnMigration,         # Add column with optional backfill
    AddTableMigration,          # Create new table
    DropColumnMigration,        # Safe column removal after finalization
    RenameColumnMigration,      # Column rename
    TransformColumnMigration,   # Reshape column values
)
```

### SQLAlchemy Contrib

```python
from transmute.contrib.sqlalchemy import (
    SQLAlchemyMigration,      # Base class with SA session helpers
    diff_sqlalchemy_models,   # Compare two SA models for structural diff
    generate_migration_code,  # Auto-generate migration from model diff
)
```

### Invariants

```python
from transmute import MigrationMeta
from transmute.core.invariants import InvariantSpec, InvariantPhase

def check_backfill_complete(migration):
    remaining = count_rows_without_new_column()
    return (remaining == 0, f"{remaining} rows still pending")

meta = MigrationMeta(
    ...,
    invariants=(
        InvariantSpec(
            name="backfill_complete",
            phase=InvariantPhase.PRE_FINALIZATION,
            params={"check": check_backfill_complete},
        ),
    ),
)
```

**Invariant Phases:** `PRE_RUN`, `DURING_RUN`, `PRE_FINALIZATION`, `POST_FINALIZATION`

---

## Code Examples for Apollo

### Catalog Migration: Add Canary State Table

```python
# apollo/migrations/v001_add_canary_state.py
"""Add canary state tracking to the catalog."""

import attrs
from semantic_version import Version
from transmute import Migration, MigrationMeta, MigrationContext


@attrs.define
class CatalogV1:
    """Schema before canary support."""
    version: int = 1


@attrs.define
class CatalogV2:
    """Schema with canary state tracking."""
    version: int = 2


class AddCanaryState(Migration[CatalogV1, CatalogV2]):
    meta = MigrationMeta(
        id="add-canary-state",
        name="Add Canary State Table",
        description="Add canary_state and canary_checkpoint tables for progressive rollouts",
        schema_version=2,
        source_version=Version("1.0.0"),
        target_version=Version("1.1.0"),
        idempotency_mode="strict",
        owner_team="platform-team",
        primary_owner="platform@apollo.dev",
        runbook_ref="https://wiki/migrations/add-canary-state",
        escalation_ref="https://pagerduty/platform",
    )

    source_schema = CatalogV1
    target_schema = CatalogV2

    def schema_additions(self) -> None:
        self.context.execute("""
            CREATE TABLE IF NOT EXISTS canary_state (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                entity_id INTEGER NOT NULL REFERENCES entity(id),
                status TEXT NOT NULL DEFAULT 'not_started',
                canary_percentage REAL NOT NULL DEFAULT 0.0,
                started_at TIMESTAMP,
                completed_at TIMESTAMP
            )
        """)
        self.context.execute("""
            CREATE TABLE IF NOT EXISTS canary_checkpoint (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                canary_state_id INTEGER NOT NULL REFERENCES canary_state(id),
                checkpoint_name TEXT NOT NULL,
                status TEXT NOT NULL DEFAULT 'pending',
                evaluated_at TIMESTAMP
            )
        """)

    def rollback(self) -> None:
        self.context.execute("DROP TABLE IF EXISTS canary_checkpoint")
        self.context.execute("DROP TABLE IF EXISTS canary_state")

    def migrate_batch(self, batch_size: int) -> bool:
        # No data migration needed — new tables are empty
        return False

    def schema_drops(self) -> None:
        # No old schema to drop — this is purely additive
        pass


### Migration Registry

```python
# apollo/migrations/registry.py
"""Register all Apollo migrations."""

from transmute import MigrationGraph
from transmute.storage.sqlalchemy_backend import SQLAlchemyStorage

from .v001_add_canary_state import AddCanaryState


def build_migration_graph(storage: SQLAlchemyStorage) -> MigrationGraph:
    """Build the Apollo migration DAG."""
    graph = MigrationGraph()

    # Register migrations in dependency order
    context = ApolloMigrationContext(storage)
    graph.register(AddCanaryState(storage=storage, context=context))
    # graph.register(SplitEntityConfig(storage=storage, context=context))
    # graph.register(AddAuditEvents(storage=storage, context=context))

    return graph
```

### Runner Factory

```python
# apollo/migrations/runner.py
"""Migration runner factory for Apollo."""

from datetime import timedelta

from transmute import MigrationRunner, SoakScheduler
from transmute.storage.sqlalchemy_backend import SQLAlchemyStorage

from .registry import build_migration_graph


def create_runner(engine, node_id: str = "local") -> MigrationRunner:
    """Create a configured migration runner."""
    storage = SQLAlchemyStorage(engine)
    graph = build_migration_graph(storage)
    scheduler = SoakScheduler(
        storage,
        pre_execution_soak=timedelta(days=4),
        pre_finalization_soak=timedelta(days=4),
    )

    return MigrationRunner(
        storage=storage,
        graph=graph,
        node_id=node_id,
        soak_scheduler=scheduler,
        batch_size=1000,
        enforce_additive_first=True,
        require_finalization_approval=True,
    )
```

---

## CLI Reference

```bash
# Status
transmute status --app apollo.migrations:build_cli_app
transmute status --json-output --include-metrics

# Preflight checks
transmute preflight --enforce-idempotency --require-owner-runbook

# Run migrations
transmute run                                    # All pending
transmute run my-migration-id                    # Specific migration
transmute run --dry-run                          # Preview only
transmute run --skip-soak --reason "P0 incident" # Emergency
transmute run --require-preflight-pass           # Enforce preflight
transmute run --enforce-additive-first           # Block destructive SQL

# Rollback
transmute rollback my-migration --plan           # Preview impact
transmute rollback my-migration --confirm        # Execute rollback

# Finalization
transmute finalization approve my-migration --by ops@team --reason "verified"
transmute finalization revoke my-migration --by admin --reason "issue found"

# Dead-letter queue
transmute dlq list my-migration
transmute dlq replay my-migration --record-ids id1,id2
transmute dlq purge my-migration --status replayed

# Audit trail
transmute audit list --migration-id my-migration
transmute audit verify-chain
transmute audit export --format jsonl

# Checkpoints
transmute checkpoint create 2.0.0 --description "Apollo v2.0.0"
transmute checkpoint can-upgrade 3.0.0
transmute checkpoint can-downgrade 1.5.0

# Scaffold new migration
transmute create my-new-migration --output apollo/migrations/
```

---

## Best Practices

### 1. One Migration per Schema Change

Keep migrations small and focused. A single migration should do one thing — add a column, rename a table, backfill data. This makes rollback predictable and soak time meaningful.

### 2. Always Implement Idempotent Operations

Both `schema_additions()` and `rollback()` must be safe to call multiple times:

```python
# Good: IF NOT EXISTS / IF EXISTS guards
def schema_additions(self) -> None:
    self.context.execute("ALTER TABLE users ADD COLUMN IF NOT EXISTS age INTEGER")

def rollback(self) -> None:
    self.context.execute("ALTER TABLE users DROP COLUMN IF EXISTS age")
```

### 3. Use Soak Times in Production

The default 4-day soak times exist for a reason — they catch issues that only manifest under real traffic patterns. Only skip soaks for genuine emergencies, and always provide a reason for the audit trail.

### 4. Require Finalization Approval for Destructive Changes

Any migration that drops columns or tables should require explicit approval before `schema_drops()` executes. This is the point of no return.

### 5. Use Checkpoints to Gate Releases

Tie migration checkpoints to Apollo release versions. This prevents deploying a version that requires migrations that haven't been finalized yet, and prevents rolling back past a finalization boundary.

### 6. Monitor via Signals

Subscribe to Transmute signals for alerting and metrics:

```python
@migration_failed.connect
def alert_on_failure(sender, migration_id, error, **kwargs):
    pagerduty.trigger(f"Migration {migration_id} failed: {error}")
```

---

## References

- [SQLAlchemy 2.0 Documentation](https://docs.sqlalchemy.org/en/20/)
- [transitions Documentation](https://github.com/pytransitions/transitions)
- [NetworkX Documentation](https://networkx.org/)
- [blinker Documentation](https://blinker.readthedocs.io/)
