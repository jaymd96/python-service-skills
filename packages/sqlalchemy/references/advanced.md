# SQLAlchemy — Advanced Features

> Part of the sqlalchemy skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Engine and Connection](#engine-and-connection)
  - [create\_engine()](#create_engine----the-starting-point)
  - [Full create\_engine() Parameter Reference](#full-create_engine-parameter-reference)
  - [Connection Pool Classes](#connection-pool-classes)
  - [engine.connect()](#engineconnect----connection-context-manager)
  - [engine.begin()](#enginebegin----transactional-context-manager)
  - [URL Construction with URL.create()](#url-construction-with-urlcreate)
- [Session Management](#session-management)
  - [Session and sessionmaker](#session-and-sessionmaker)
  - [session.merge()](#sessionmerge----merge-detached-or-external-objects)
  - [flush() vs commit()](#flush-vs-commit)
  - [session.refresh() and session.expire()](#sessionrefresh-and-sessionexpire)
- [Migrations with Alembic](#migrations-with-alembic)
  - [Setup](#setup)
  - [Configuration](#configuration)
  - [Workflow](#workflow)
  - [Example Migration](#example-migration)
- [Events](#events)
  - [@event.listens\_for -- Lifecycle Hooks](#eventlistens_for----lifecycle-hooks)
  - [Session Events](#session-events)
  - [Connection / Engine Events](#connection--engine-events)
  - [Pool Events](#pool-events)
- [Async Usage](#async-usage)
- [Quick Reference](#quick-reference)
  - [Imports Cheat Sheet](#imports-cheat-sheet)
  - [Common Patterns](#common-patterns)

---

## Engine and Connection

### `create_engine()` -- The Starting Point

The `Engine` is the source of database connectivity and connection pooling. One `Engine` per database per application is typical.

```python
from sqlalchemy import create_engine

# SQLite (file-based)
engine = create_engine("sqlite:///app.db")

# SQLite (in-memory)
engine = create_engine("sqlite:///:memory:")

# PostgreSQL
engine = create_engine("postgresql+psycopg2://user:password@localhost:5432/mydb")

# PostgreSQL with psycopg3
engine = create_engine("postgresql+psycopg://user:password@localhost:5432/mydb")

# MySQL
engine = create_engine("mysql+mysqldb://user:password@localhost:3306/mydb")

# MySQL with PyMySQL
engine = create_engine("mysql+pymysql://user:password@localhost:3306/mydb")
```

#### Full `create_engine()` Parameter Reference

```python
engine = create_engine(
    "postgresql+psycopg2://user:pass@host:5432/dbname",

    # Logging / debugging
    echo=False,              # If True, log all SQL to stdout (uses Python logging)
    echo_pool=False,         # If True, log connection pool checkouts/checkins

    # Connection pool configuration
    pool_size=5,             # Number of persistent connections in the pool
    max_overflow=10,         # Max connections beyond pool_size (burst)
    pool_timeout=30,         # Seconds to wait for a connection before raising an error
    pool_recycle=3600,       # Seconds before a connection is recycled (prevents stale connections)
    pool_pre_ping=True,      # Issue a SELECT 1 before using a connection (detects disconnects)
    poolclass=None,          # Override pool implementation (QueuePool, NullPool, StaticPool, etc.)

    # Execution options
    execution_options={},    # Default execution options for all statements
    isolation_level=None,    # "SERIALIZABLE", "REPEATABLE READ", "READ COMMITTED", "AUTOCOMMIT"

    # Other
    future=True,             # Redundant in 2.0 (always True), kept for 1.x migration
    connect_args={},         # Dict passed directly to the DBAPI connect() call
    query_cache_size=500,    # Size of the SQL compilation cache
)
```

#### Connection Pool Classes

```python
from sqlalchemy.pool import (
    QueuePool,       # Default. Maintains a fixed-size pool with overflow.
    NullPool,        # No pooling. Each connect() opens a new connection, closes on return.
    StaticPool,      # A single connection reused for all requests. Good for SQLite in-memory.
    AsyncAdaptedQueuePool,  # Default for async engines.
)

# NullPool is common for serverless / short-lived processes
engine = create_engine("postgresql://...", poolclass=NullPool)

# StaticPool is required for SQLite in-memory with multiple threads
engine = create_engine(
    "sqlite:///:memory:",
    poolclass=StaticPool,
    connect_args={"check_same_thread": False},
)
```

### `engine.connect()` -- Connection Context Manager

Returns a `Connection` object. Changes are **not** automatically committed.

```python
from sqlalchemy import text

with engine.connect() as conn:
    result = conn.execute(text("SELECT * FROM users WHERE id = :id"), {"id": 1})
    row = result.fetchone()
    print(row)
    # Connection is returned to the pool when the block exits
    # Any uncommitted transaction is ROLLED BACK
```

**Manual commit with `connect()`:**

```python
with engine.connect() as conn:
    conn.execute(text("INSERT INTO users (name) VALUES (:name)"), {"name": "Alice"})
    conn.commit()  # Explicit commit required
```

### `engine.begin()` -- Transactional Context Manager

Wraps the block in a transaction. Commits on success, rolls back on exception.

```python
with engine.begin() as conn:
    conn.execute(text("INSERT INTO users (name) VALUES (:name)"), {"name": "Alice"})
    conn.execute(text("INSERT INTO users (name) VALUES (:name)"), {"name": "Bob"})
    # Automatically commits if no exception
    # Automatically rolls back if an exception is raised
```

### URL Construction with `URL.create()`

For programmatic URL building (avoids string escaping issues):

```python
from sqlalchemy import URL

url = URL.create(
    drivername="postgresql+psycopg2",
    username="user",
    password="p@ss:word!",  # Special chars handled automatically
    host="localhost",
    port=5432,
    database="mydb",
    query={"sslmode": "require"},
)
engine = create_engine(url)
```

---

## Session Management

### `Session` and `sessionmaker`

The `Session` manages the ORM's "unit of work" -- it tracks changes to objects and flushes them to the database.

```python
from sqlalchemy.orm import Session, sessionmaker

# Option 1: Direct Session (simple scripts)
with Session(engine) as session:
    ...

# Option 2: sessionmaker factory (applications)
SessionFactory = sessionmaker(bind=engine)

with SessionFactory() as session:
    ...

# Option 3: sessionmaker with begin (auto-commit/rollback)
with SessionFactory.begin() as session:
    session.add(User(name="Alice"))
    # Commits on exit, rolls back on exception
```

### `session.merge()` -- Merge Detached or External Objects

`merge()` copies the state of an object into the session. If an object with the same PK already exists in the session, it merges into that; otherwise it loads one from the database or creates a new pending instance.

```python
# Common use case: merging data from an API or deserialized object
external_user = User(id=42, name="Updated Name", email="new@example.com")

with Session(engine) as session:
    merged_user = session.merge(external_user)
    # merged_user is now the session-managed instance
    session.commit()
```

### `flush()` vs `commit()`

| Method | Effect |
|---|---|
| `session.flush()` | Writes pending changes to the database (issues SQL) but does **not** commit the transaction. Auto-generated PKs become available. |
| `session.commit()` | Calls `flush()` implicitly, then commits the transaction. Makes changes permanent and visible to other connections. |

```python
with Session(engine) as session:
    user = User(name="Alice")
    session.add(user)

    session.flush()
    # user.id is now available (the INSERT was sent)
    # But the transaction has NOT been committed

    print(user.id)  # e.g., 1

    session.commit()
    # NOW the transaction is committed
```

### `session.refresh()` and `session.expire()`

```python
with Session(engine) as session:
    user = session.get(User, 1)

    # refresh(): Immediately reloads the object from the database
    session.refresh(user)
    # user's attributes now reflect the current database state

    # expire(): Marks the object as stale. Next attribute access triggers a reload.
    session.expire(user)
    # No SQL yet...
    print(user.name)  # NOW SQL is issued to reload

    # expire_all(): Expire every object in the session
    session.expire_all()
```

---

## Migrations with Alembic

**Alembic** is SQLAlchemy's official database migration tool. It auto-generates migration scripts based on model changes.

### Setup

```bash
pip install alembic

# Initialize alembic in your project
alembic init alembic
```

This creates an `alembic/` directory and `alembic.ini`.

### Configuration

Edit `alembic/env.py` to import your models and set the target metadata:

```python
# alembic/env.py
from myapp.models import Base

target_metadata = Base.metadata
```

Set the database URL in `alembic.ini`:

```ini
sqlalchemy.url = postgresql+psycopg2://user:pass@localhost/mydb
```

### Workflow

```bash
# Auto-generate a migration from model changes
alembic revision --autogenerate -m "add users table"

# Review the generated migration in alembic/versions/xxxx_add_users_table.py
# ALWAYS review autogenerated migrations before applying!

# Apply migrations (upgrade to latest)
alembic upgrade head

# Downgrade one step
alembic downgrade -1

# Show current migration status
alembic current

# Show migration history
alembic history --verbose
```

### Example Migration

```python
"""add users table

Revision ID: a1b2c3d4e5f6
Revises:
Create Date: 2024-01-15 10:30:00.000000
"""
from typing import Sequence, Union
from alembic import op
import sqlalchemy as sa

revision: str = "a1b2c3d4e5f6"
down_revision: Union[str, None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None

def upgrade() -> None:
    op.create_table(
        "users",
        sa.Column("id", sa.Integer(), nullable=False),
        sa.Column("name", sa.String(100), nullable=False),
        sa.Column("email", sa.String(255), nullable=True),
        sa.Column("created_at", sa.DateTime(), server_default=sa.func.now()),
        sa.PrimaryKeyConstraint("id"),
    )
    op.create_index("ix_users_email", "users", ["email"], unique=True)

def downgrade() -> None:
    op.drop_index("ix_users_email", table_name="users")
    op.drop_table("users")
```

---

## Events

SQLAlchemy provides an event system for hooking into ORM and Core lifecycle stages.

### `@event.listens_for` -- Lifecycle Hooks

```python
from sqlalchemy import event

# ORM events -- before/after insert, update, delete

@event.listens_for(User, "before_insert")
def user_before_insert(mapper, connection, target):
    """Called before a User is INSERTed."""
    target.name = target.name.strip()

@event.listens_for(User, "before_update")
def user_before_update(mapper, connection, target):
    """Called before a User is UPDATEd."""
    target.updated_at = datetime.utcnow()

@event.listens_for(User, "after_insert")
def user_after_insert(mapper, connection, target):
    """Called after a User is INSERTed (within the same transaction)."""
    print(f"Created user {target.id}")

@event.listens_for(User, "after_delete")
def user_after_delete(mapper, connection, target):
    """Called after a User is DELETEd."""
    print(f"Deleted user {target.id}")
```

### Session Events

```python
@event.listens_for(Session, "before_commit")
def before_commit(session):
    """Called before session.commit()."""
    for obj in session.new:
        print(f"About to insert: {obj}")

@event.listens_for(Session, "after_commit")
def after_commit(session):
    """Called after session.commit() succeeds."""
    pass

@event.listens_for(Session, "after_rollback")
def after_rollback(session):
    """Called after session.rollback()."""
    pass
```

### Connection / Engine Events

```python
@event.listens_for(engine, "before_cursor_execute")
def before_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    """Log or modify every SQL statement before execution."""
    conn.info.setdefault("query_start_time", []).append(datetime.utcnow())

@event.listens_for(engine, "after_cursor_execute")
def after_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    """Measure query execution time."""
    total = datetime.utcnow() - conn.info["query_start_time"].pop()
    if total.total_seconds() > 1.0:
        print(f"SLOW QUERY ({total}): {statement[:200]}")
```

### Pool Events

```python
@event.listens_for(engine, "connect")
def on_connect(dbapi_connection, connection_record):
    """Called when a new DBAPI connection is created."""
    # Example: set PostgreSQL search_path
    cursor = dbapi_connection.cursor()
    cursor.execute("SET search_path TO myschema, public")
    cursor.close()

@event.listens_for(engine, "checkout")
def on_checkout(dbapi_connection, connection_record, connection_proxy):
    """Called when a connection is checked out from the pool."""
    pass
```

---

## Async Usage

```python
"""SQLAlchemy 2.0 async example using asyncio."""
import asyncio
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy import select
from sqlalchemy.orm import selectinload

# Async engine (note the async driver: asyncpg, aiosqlite, etc.)
async_engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/mydb",
    echo=True,
)

# For SQLite async:
# async_engine = create_async_engine("sqlite+aiosqlite:///app.db")

AsyncSessionFactory = async_sessionmaker(async_engine, class_=AsyncSession)


async def main():
    # Create tables
    async with async_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    # CRUD operations
    async with AsyncSessionFactory() as session:
        async with session.begin():
            session.add(User(username="alice", email="alice@example.com"))

    # Queries
    async with AsyncSessionFactory() as session:
        stmt = (
            select(User)
            .options(selectinload(User.posts))
            .where(User.username == "alice")
        )
        result = await session.execute(stmt)
        user = result.scalar_one()
        print(user.username, user.posts)

    # Cleanup
    await async_engine.dispose()


asyncio.run(main())
```

**Important async notes:**

- Lazy loading does NOT work in async -- you must always use eager loading or `await session.run_sync()`.
- Use `aiosqlite` for SQLite, `asyncpg` for PostgreSQL, `aiomysql` for MySQL.
- The rest of the ORM API (models, `select()`, `relationship()`) is identical to sync.

---

## Quick Reference

### Imports Cheat Sheet

```python
# Engine and connection
from sqlalchemy import create_engine, URL, text

# ORM base and mapping
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

# Session
from sqlalchemy.orm import Session, sessionmaker

# Querying
from sqlalchemy import select, insert, update, delete
from sqlalchemy import func, desc, asc, or_, and_, not_, case, cast, exists

# Column types
from sqlalchemy import (
    String, Text, Integer, BigInteger, SmallInteger,
    Float, Numeric, Boolean, DateTime, Date, Time,
    Interval, LargeBinary, JSON, Uuid, Enum,
)

# Foreign keys and constraints
from sqlalchemy import ForeignKey, UniqueConstraint, CheckConstraint, Index

# Eager loading strategies
from sqlalchemy.orm import (
    selectinload, joinedload, subqueryload,
    lazyload, contains_eager, raiseload,
)

# Events
from sqlalchemy import event

# Async
from sqlalchemy.ext.asyncio import (
    create_async_engine, AsyncSession, async_sessionmaker,
)
```

### Common Patterns

```python
# Session-per-request (web app pattern)
def get_db():
    """FastAPI / Starlette dependency."""
    db = SessionFactory()
    try:
        yield db
    finally:
        db.close()

# Or with a context manager
from contextlib import contextmanager

@contextmanager
def get_session():
    session = SessionFactory()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()
```
