# Transmute — Lifecycle & Safety

> Part of the transmute skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [8-State FSM](#8-state-fsm)
- [State Descriptions](#state-descriptions)
- [Dual-Write Routing](#dual-write-routing)
- [Soak Times](#soak-times)
- [Finalization](#finalization)
- [Rollback](#rollback)
- [Dead-Letter Queue](#dead-letter-queue)
- [Audit Trail](#audit-trail)
- [Checkpoints](#checkpoints)

## 8-State FSM

```
Register migration
        |
        v
UNINITIALIZED <-----------------------------+
        | initialize                         |
        v                                    | rollback
INITIALIZING  (schema_additions)             | (from any pre-FINISHING state)
        | start                              |
        v                                    |
   RUNNING  (migrate_batch loop)             |
        |                                    |
        +---> AWAITING_ACTION                |
        |     (external gate)                |
        |                                    |
        v await_finalization                 |
AWAITING_FINALIZATION -----------------------+
        | (soak time + approval)
        v
  FINISHING  (schema_drops -- POINT OF NO RETURN)
        |
        v
   FINISHED
```

## State Descriptions

| State | What Happens | Next States |
|-------|-------------|-------------|
| `UNINITIALIZED` | Migration registered, not started | INITIALIZING |
| `INITIALIZING` | `schema_additions()` executes (CREATE TABLE, ADD COLUMN) | RUNNING, ROLLING_BACK |
| `RUNNING` | `migrate_batch()` loop processes records | AWAITING_ACTION, AWAITING_FINALIZATION, ROLLING_BACK |
| `AWAITING_ACTION` | Paused for external action (manual gate) | RUNNING, ROLLING_BACK |
| `AWAITING_FINALIZATION` | Waiting for soak time + explicit approval | FINISHING, ROLLING_BACK |
| `FINISHING` | `schema_drops()` executes (DROP COLUMN, DROP TABLE) | FINISHED |
| `FINISHED` | Migration complete, old schema removed | Terminal |
| `ROLLING_BACK` | `rollback()` executes, undoing schema_additions | UNINITIALIZED |

## Dual-Write Routing

| Migration State | Read From | Write To |
|-----------------|-----------|----------|
| UNINITIALIZED, INITIALIZING, ROLLING_BACK | old | old |
| RUNNING, AWAITING_ACTION, AWAITING_FINALIZATION | old | old + new (dual-write) |
| FINISHING, FINISHED | new | new |

During RUNNING state, all writes go to both old and new schemas. Reads come from old schema until FINISHING, when they switch to new.

## Soak Times

Two configurable soak periods:

| Soak Period | Default | Purpose |
|-------------|---------|---------|
| Pre-execution | 4 days | Wait before running migration in production |
| Pre-finalization | 4 days | Wait before irreversible schema_drops |

```python
SoakScheduler(storage,
    pre_execution_soak=timedelta(days=4),
    pre_finalization_soak=timedelta(days=4),
)

# Emergency skip (requires reason for audit trail)
runner.skip_soak("my-migration", reason="P0 incident", actor_id="oncall")
```

## Finalization

Finalization is the point of no return — `schema_drops()` removes old columns/tables.

```python
# Approve (required before FINISHING)
runner.approve_finalization(
    "my-migration",
    approved_by="ops@team.com",
    reason="Verified data consistency",
    ttl=timedelta(hours=24),        # Approval expires
)

# Revoke approval
runner.revoke_finalization_approval("my-migration", actor_id="admin", reason="issue found")
```

## Rollback

Available from any pre-FINISHING state:

```python
# Preview impact
impact = runner.rollback_plan("my-migration")

# Execute rollback
runner.rollback("my-migration")
```

Rollback calls `migration.rollback()` which should undo `schema_additions()`. After rollback, migration returns to UNINITIALIZED.

**Once FINISHING begins, rollback is impossible** — the old schema is being dropped.

## Dead-Letter Queue

Failed records during `migrate_batch()` are captured in the DLQ:

```python
records = runner.list_dead_letters("my-migration", status="open")
replayed = runner.replay_dead_letters("my-migration", record_ids=["id1", "id2"])
purged = runner.purge_dead_letters("my-migration", status="replayed")
```

Records track retry counts and failure reasons.

## Audit Trail

Hash-chained, immutable record of all migration actions:

```bash
transmute audit list --migration-id my-migration
transmute audit verify-chain     # Verify hash integrity
transmute audit export --format jsonl
```

Every state change, approval, rollback, skip, and DLQ action is recorded with actor, timestamp, and reason.

## Checkpoints

Tie migrations to release versions:

```python
manager = CheckpointManager(storage, graph)
manager.create_checkpoint("2.0.0", description="Apollo v2.0.0")

# Prevent deploying version requiring unfinished migrations
can, reason = manager.can_upgrade_to(Version("3.0.0"))

# Prevent rolling back past finalization boundary
can, reason = manager.can_downgrade_to(Version("1.5.0"))
```
