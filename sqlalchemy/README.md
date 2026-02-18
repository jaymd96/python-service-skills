# SQLAlchemy -- The Python SQL Toolkit and ORM (2.0 Style)

## Overview

**SQLAlchemy** is the most widely adopted database toolkit and object-relational mapper (ORM) for Python. It provides a full suite of well-known enterprise-level persistence patterns, designed for efficient and high-performing database access, adapted into a simple and Pythonic domain language. SQLAlchemy 2.0 represents a major modernization of the API, introducing first-class type annotation support, a unified query interface via `select()`, and a cleaner declarative ORM style built on `DeclarativeBase` and `Mapped[]` type hints.

**Key Characteristics:**

- **Version:** 2.0.40 (latest stable as of early 2026)
- **Python:** 3.7+ (3.9+ recommended for full typing support)
- **License:** MIT
- **Dependencies:** `typing_extensions` (for older Python versions); database drivers are separate installs
- **Thread Safety:** Engine is thread-safe; Session is NOT thread-safe (one Session per thread/task)
- **Async Support:** Native asyncio via `create_async_engine` and `AsyncSession`

**SQLAlchemy has two distinct APIs:**

| Layer | Purpose | Key Construct |
|---|---|---|
| **Core** | SQL expression language, schema DDL, connection management | `Engine`, `Connection`, `Table`, `select()` |
| **ORM** | Object-relational mapping, unit of work, identity map | `Session`, `DeclarativeBase`, `Mapped`, `relationship()` |

Both layers use the same `select()` construct in 2.0 style. The ORM builds on top of Core.

**When to use SQLAlchemy:**

- Building applications that require relational database persistence
- Projects needing database-agnostic SQL generation (PostgreSQL, MySQL, SQLite, Oracle, MSSQL)
- Applications requiring sophisticated relationship modeling and eager/lazy loading strategies
- Systems needing connection pooling and transaction management
- When you want compile-time-checkable database models via type annotations

---

## Installation

```bash
pip install sqlalchemy
```

**With database driver extras (recommended):**

```bash
# PostgreSQL (psycopg2 -- the classic driver)
pip install sqlalchemy[postgresql]

# PostgreSQL (psycopg -- the modern async-capable driver, aka psycopg3)
pip install sqlalchemy[postgresql_psycopg]

# MySQL
pip install sqlalchemy[mysql]

# SQLite (built into Python, no extra needed)
pip install sqlalchemy

# PostgreSQL async (asyncpg)
pip install sqlalchemy[postgresql_asyncpg]

# MySQL async (aiomysql)
pip install sqlalchemy[aiomysql]
```

**Installing drivers separately:**

```bash
pip install psycopg2-binary   # PostgreSQL (binary wheels, convenient for development)
pip install psycopg[binary]   # psycopg3 with binary builds
pip install mysqlclient        # MySQL/MariaDB (C-based, fast)
pip install pymysql            # MySQL (pure Python)
pip install asyncpg            # PostgreSQL async
```

**Version pinning for 2.0:**

```bash
pip install "sqlalchemy>=2.0,<3.0"
```

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

## ORM Declarative 2.0 Style

### `DeclarativeBase` -- The Modern Base Class

In SQLAlchemy 2.0, `DeclarativeBase` replaces the old `declarative_base()` function.

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass
```

All ORM models inherit from this `Base` class. The base maintains the `MetaData` and a registry for all models.

**Custom type mapping in the base (optional):**

```python
import datetime
from decimal import Decimal
from sqlalchemy import String, DECIMAL
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    type_annotation_map = {
        str: String(255),              # Default str maps to VARCHAR(255) instead of TEXT
        Decimal: DECIMAL(10, 2),       # Default decimal precision
    }
```

### `Mapped[type]` -- Typed Column Declarations

`Mapped[T]` declares a column's Python type. SQLAlchemy automatically infers the SQL type.

```python
from datetime import datetime
from typing import Optional
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    # Required (NOT NULL) integer primary key
    id: Mapped[int] = mapped_column(primary_key=True)

    # Required (NOT NULL) string
    name: Mapped[str] = mapped_column()

    # Nullable string (Optional makes it NULL-able)
    email: Mapped[Optional[str]] = mapped_column()

    # With default value
    is_active: Mapped[bool] = mapped_column(default=True)

    # Timestamp
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```

**Type inference table:**

| Python Type | SQL Type |
|---|---|
| `int` | `INTEGER` |
| `str` | `VARCHAR` / `TEXT` |
| `float` | `FLOAT` |
| `bool` | `BOOLEAN` |
| `bytes` | `BLOB` / `BYTEA` |
| `datetime.datetime` | `DATETIME` / `TIMESTAMP` |
| `datetime.date` | `DATE` |
| `datetime.time` | `TIME` |
| `datetime.timedelta` | `INTERVAL` |
| `decimal.Decimal` | `NUMERIC` |
| `uuid.UUID` | `UUID` (PostgreSQL) / `CHAR(32)` |

### `mapped_column()` -- Column Configuration

```python
from sqlalchemy import String, Text, func
from sqlalchemy.orm import Mapped, mapped_column

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)

    # Explicit SQL type
    name: Mapped[str] = mapped_column(String(100))

    # Nullable (prefer Optional[] but can be explicit)
    bio: Mapped[Optional[str]] = mapped_column(Text)

    # Default value (Python-side, set before INSERT)
    is_active: Mapped[bool] = mapped_column(default=True)

    # Server-side default (SQL expression, evaluated by the DB)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())

    # Index
    email: Mapped[str] = mapped_column(String(255), index=True, unique=True)

    # Composite index via __table_args__
    first_name: Mapped[str] = mapped_column(String(50))
    last_name: Mapped[str] = mapped_column(String(50))

    __table_args__ = (
        Index("ix_full_name", "first_name", "last_name"),
        UniqueConstraint("email", name="uq_user_email"),
    )
```

#### Full `mapped_column()` Parameter Reference

```python
mapped_column(
    # Positional: SQL type (optional, inferred from Mapped[T] if omitted)
    String(255),

    # Column DDL
    primary_key=False,       # Is this a primary key column?
    nullable=None,           # None = inferred from Mapped[T] vs Mapped[Optional[T]]
    unique=False,            # Add a UNIQUE constraint
    index=False,             # Create an index on this column
    autoincrement=True,      # For integer PKs. "auto" | True | False
    comment=None,            # SQL comment on the column

    # Default values
    default=None,            # Python-side default (called before INSERT)
    default_factory=None,    # Python callable that returns the default (for mutable defaults)
    server_default=None,     # SQL expression evaluated by the database server
    server_onupdate=None,    # SQL expression for ON UPDATE

    # Foreign keys
    # Use ForeignKey() inside mapped_column: mapped_column(ForeignKey("other.id"))

    # ORM-specific
    init=True,               # Include in __init__ if using MappedAsDataclass
    repr=True,               # Include in __repr__ if using MappedAsDataclass
    compare=True,            # Include in __eq__ if using MappedAsDataclass
    insert_default=None,     # Default for dataclass-style INSERT

    # Naming / aliasing
    name=None,               # Override the SQL column name (Python attr name used if None)
    key=None,                # Override the attribute name in the mapping
)
```

### `relationship()` -- ORM Relationships

```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

    # One-to-many: User has many posts
    posts: Mapped[list["Post"]] = relationship(back_populates="author")

class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))

    # Many-to-one: Post belongs to one user
    author: Mapped["User"] = relationship(back_populates="posts")
```

**Important:** The `Mapped[list["Post"]]` type hint tells SQLAlchemy (and type checkers) this is a collection relationship. For scalar relationships, use `Mapped["User"]`.

---

## Column Types

### Standard Types

```python
from sqlalchemy import (
    String,        # VARCHAR(length). Use String(255) or just String for unlimited.
    Text,          # TEXT -- unlimited length string
    Integer,       # INTEGER (32-bit)
    BigInteger,    # BIGINT (64-bit)
    SmallInteger,  # SMALLINT (16-bit)
    Float,         # FLOAT
    Numeric,       # NUMERIC(precision, scale) -- exact decimal
    Boolean,       # BOOLEAN
    DateTime,      # DATETIME / TIMESTAMP
    Date,          # DATE
    Time,          # TIME
    Interval,      # INTERVAL (PostgreSQL, Oracle)
    LargeBinary,   # BLOB / BYTEA -- binary data
    Uuid,          # UUID (native on PostgreSQL, emulated elsewhere) -- added in 2.0
)
```

### Specialized Types

```python
from sqlalchemy import JSON, Enum, ARRAY
from sqlalchemy.dialects.postgresql import (
    JSONB,         # PostgreSQL binary JSON (indexable, faster queries)
    ARRAY as PG_ARRAY,
    HSTORE,
    INET,
    CIDR,
    MACADDR,
    TSVECTOR,
    UUID as PG_UUID,
    INTERVAL as PG_INTERVAL,
)
```

### Column Type Usage Examples

```python
import enum
import uuid
from datetime import datetime
from decimal import Decimal
from typing import Optional

from sqlalchemy import (
    String, Text, Integer, Float, Numeric, Boolean,
    DateTime, JSON, Enum as SAEnum, LargeBinary, Uuid,
    ARRAY, ForeignKey, func,
)
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Priority(enum.Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"


class Product(Base):
    __tablename__ = "products"

    # UUID primary key
    id: Mapped[uuid.UUID] = mapped_column(
        Uuid, primary_key=True, default=uuid.uuid4
    )

    # String with length constraint
    name: Mapped[str] = mapped_column(String(200))

    # Unlimited text
    description: Mapped[Optional[str]] = mapped_column(Text)

    # Exact decimal for currency
    price: Mapped[Decimal] = mapped_column(Numeric(10, 2))

    # Float for approximate values
    weight_kg: Mapped[Optional[float]] = mapped_column(Float)

    # Boolean
    is_available: Mapped[bool] = mapped_column(Boolean, default=True)

    # Enum column (stored as VARCHAR by default)
    priority: Mapped[Priority] = mapped_column(
        SAEnum(Priority), default=Priority.MEDIUM
    )

    # JSON column (works on PostgreSQL, MySQL 5.7+, SQLite 3.9+)
    metadata_: Mapped[Optional[dict]] = mapped_column(
        "metadata", JSON, default=None  # "metadata" overrides column name
    )

    # JSONB (PostgreSQL only -- indexable, faster queries)
    # attributes: Mapped[Optional[dict]] = mapped_column(JSONB, default=None)

    # Binary data
    thumbnail: Mapped[Optional[bytes]] = mapped_column(LargeBinary)

    # Timestamp with server default
    created_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now()
    )

    # PostgreSQL ARRAY (PostgreSQL only)
    # tags: Mapped[Optional[list[str]]] = mapped_column(ARRAY(String(50)))
```

### Working with JSON Columns

```python
from sqlalchemy import JSON, cast, type_coerce

class Config(Base):
    __tablename__ = "configs"

    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[dict] = mapped_column(JSON, default=dict)

# Insert
config = Config(data={"theme": "dark", "lang": "en", "features": ["beta", "export"]})
session.add(config)
session.commit()

# Query JSON fields (PostgreSQL JSONB is much more capable here)
stmt = select(Config).where(Config.data["theme"].as_string() == "dark")

# For nested access
stmt = select(Config).where(Config.data["features"][0].as_string() == "beta")
```

### Working with Enum Columns

```python
import enum

class Status(enum.Enum):
    DRAFT = "draft"
    PUBLISHED = "published"
    ARCHIVED = "archived"

class Article(Base):
    __tablename__ = "articles"

    id: Mapped[int] = mapped_column(primary_key=True)

    # Using Python enum -- stores the .value string in the database
    status: Mapped[Status] = mapped_column(
        SAEnum(Status, values_callable=lambda e: [i.value for i in e]),
        default=Status.DRAFT,
    )

    # Or simply (auto-detects values from enum):
    status2: Mapped[Status] = mapped_column(SAEnum(Status), default=Status.DRAFT)

# Query by enum member
stmt = select(Article).where(Article.status == Status.PUBLISHED)
```

---

## Relationships

### One-to-Many / Many-to-One

```python
class Department(Base):
    __tablename__ = "departments"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

    # One department has many employees
    employees: Mapped[list["Employee"]] = relationship(
        back_populates="department",
        cascade="all, delete-orphan",  # Delete employees when department is deleted
    )

class Employee(Base):
    __tablename__ = "employees"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    department_id: Mapped[int] = mapped_column(ForeignKey("departments.id"))

    # Each employee belongs to one department
    department: Mapped["Department"] = relationship(back_populates="employees")
```

### One-to-One

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

    # uselist=False makes this a scalar (one-to-one)
    profile: Mapped[Optional["Profile"]] = relationship(
        back_populates="user", uselist=False, cascade="all, delete-orphan"
    )

class Profile(Base):
    __tablename__ = "profiles"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), unique=True)
    bio: Mapped[Optional[str]] = mapped_column(Text)

    user: Mapped["User"] = relationship(back_populates="profile")
```

### Many-to-Many

Requires an association table.

```python
from sqlalchemy import Table, Column, ForeignKey

# Association table (no ORM class needed for simple many-to-many)
article_tags = Table(
    "article_tags",
    Base.metadata,
    Column("article_id", Integer, ForeignKey("articles.id"), primary_key=True),
    Column("tag_id", Integer, ForeignKey("tags.id"), primary_key=True),
)

class Article(Base):
    __tablename__ = "articles"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))

    tags: Mapped[list["Tag"]] = relationship(
        secondary=article_tags, back_populates="articles"
    )

class Tag(Base):
    __tablename__ = "tags"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)

    articles: Mapped[list["Article"]] = relationship(
        secondary=article_tags, back_populates="tags"
    )
```

**Usage:**

```python
tag = Tag(name="python")
article = Article(title="SQLAlchemy Guide")
article.tags.append(tag)
session.add(article)
session.commit()
```

### Many-to-Many with Association Object (Extra Columns)

When the association table needs additional columns, use an explicit model.

```python
from datetime import datetime

class Enrollment(Base):
    """Association object with extra data."""
    __tablename__ = "enrollments"

    student_id: Mapped[int] = mapped_column(
        ForeignKey("students.id"), primary_key=True
    )
    course_id: Mapped[int] = mapped_column(
        ForeignKey("courses.id"), primary_key=True
    )
    enrolled_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    grade: Mapped[Optional[str]] = mapped_column(String(2))

    # Relationships to both sides
    student: Mapped["Student"] = relationship(back_populates="enrollments")
    course: Mapped["Course"] = relationship(back_populates="enrollments")


class Student(Base):
    __tablename__ = "students"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

    enrollments: Mapped[list["Enrollment"]] = relationship(
        back_populates="student", cascade="all, delete-orphan"
    )

class Course(Base):
    __tablename__ = "courses"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))

    enrollments: Mapped[list["Enrollment"]] = relationship(
        back_populates="course", cascade="all, delete-orphan"
    )
```

### Self-Referential (Adjacency List)

```python
class Category(Base):
    __tablename__ = "categories"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    parent_id: Mapped[Optional[int]] = mapped_column(ForeignKey("categories.id"))

    # Children of this category
    children: Mapped[list["Category"]] = relationship(
        back_populates="parent",
        cascade="all, delete-orphan",
    )

    # Parent category
    parent: Mapped[Optional["Category"]] = relationship(
        back_populates="children",
        remote_side=[id],  # Tells SQLAlchemy which side is the "one"
    )
```

**Usage:**

```python
root = Category(name="Electronics")
phones = Category(name="Phones", parent=root)
android = Category(name="Android", parent=phones)
session.add(root)
session.commit()

# Navigate the tree
print(root.children)            # [Category(name="Phones")]
print(phones.parent.name)       # "Electronics"
print(phones.children)          # [Category(name="Android")]
```

### `back_populates` vs `backref`

**`back_populates` (recommended in 2.0):** Explicit, declared on both sides. Both relationship attributes are visible in the source code.

```python
class Parent(Base):
    __tablename__ = "parents"
    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[list["Child"]] = relationship(back_populates="parent")

class Child(Base):
    __tablename__ = "children"
    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parents.id"))
    parent: Mapped["Parent"] = relationship(back_populates="children")
```

**`backref` (legacy, still works):** Declared on one side only. Creates the attribute on the other model automatically, but it is invisible in the other model's source code. This makes code harder to understand and type-check.

```python
class Parent(Base):
    __tablename__ = "parents"
    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[list["Child"]] = relationship(backref="parent")

class Child(Base):
    __tablename__ = "children"
    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parents.id"))
    # NOTE: "parent" attribute exists at runtime but is NOT declared here
```

**Recommendation:** Always use `back_populates` for clarity, discoverability, and type checker support.

### Multiple Foreign Keys to the Same Table

When a model has multiple FKs pointing to the same table, you must specify `foreign_keys` to resolve ambiguity.

```python
class Employee(Base):
    __tablename__ = "employees"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    manager_id: Mapped[Optional[int]] = mapped_column(ForeignKey("employees.id"))
    mentor_id: Mapped[Optional[int]] = mapped_column(ForeignKey("employees.id"))

    # Must specify foreign_keys to resolve ambiguity
    manager: Mapped[Optional["Employee"]] = relationship(
        foreign_keys=[manager_id],
        remote_side=[id],
    )
    mentor: Mapped[Optional["Employee"]] = relationship(
        foreign_keys=[mentor_id],
        remote_side=[id],
    )
```

Another common example -- a `Transfer` referencing two different `Account` rows:

```python
class Account(Base):
    __tablename__ = "accounts"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

class Transfer(Base):
    __tablename__ = "transfers"

    id: Mapped[int] = mapped_column(primary_key=True)
    amount: Mapped[Decimal] = mapped_column(Numeric(12, 2))
    from_account_id: Mapped[int] = mapped_column(ForeignKey("accounts.id"))
    to_account_id: Mapped[int] = mapped_column(ForeignKey("accounts.id"))

    from_account: Mapped["Account"] = relationship(foreign_keys=[from_account_id])
    to_account: Mapped["Account"] = relationship(foreign_keys=[to_account_id])
```

### Cascade Options

```python
class Parent(Base):
    __tablename__ = "parents"
    id: Mapped[int] = mapped_column(primary_key=True)

    children: Mapped[list["Child"]] = relationship(
        back_populates="parent",
        cascade="all, delete-orphan",
        # cascade options:
        #   "save-update"    -- (default) add child to session when parent is added
        #   "merge"          -- (default) merge child when parent is merged
        #   "expunge"        -- expunge child when parent is expunged
        #   "delete"         -- delete child when parent is deleted
        #   "delete-orphan"  -- delete child when removed from parent's collection
        #   "refresh-expire" -- (default) refresh/expire child when parent is
        #   "all"            -- shorthand for save-update, merge, refresh-expire,
        #                       expunge, delete
        passive_deletes=True,  # Rely on DB CASCADE instead of loading children to delete
    )
```

---

## Session and CRUD

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

### CRUD Operations

#### Create

```python
with Session(engine) as session:
    # Single insert
    user = User(name="Alice", email="alice@example.com")
    session.add(user)
    session.commit()

    # After commit, user.id is populated (auto-increment)
    print(user.id)  # e.g., 1

    # Bulk insert (more efficient)
    users = [
        User(name="Bob", email="bob@example.com"),
        User(name="Charlie", email="charlie@example.com"),
    ]
    session.add_all(users)
    session.commit()
```

#### Read

```python
from sqlalchemy import select

with Session(engine) as session:
    # Get by primary key
    user = session.get(User, 1)  # Returns User or None

    # Select one
    stmt = select(User).where(User.name == "Alice")
    user = session.execute(stmt).scalar_one()       # Raises if not exactly 1
    user = session.execute(stmt).scalar_one_or_none()  # Returns None if 0

    # Select all
    stmt = select(User).order_by(User.name)
    users = session.execute(stmt).scalars().all()  # list[User]

    # Iterate without loading all into memory
    stmt = select(User)
    for user in session.execute(stmt).scalars():
        print(user.name)
```

#### Update

```python
with Session(engine) as session:
    # Fetch-then-modify (ORM tracks changes automatically)
    user = session.get(User, 1)
    user.name = "Alice Updated"
    session.commit()  # Issues UPDATE

    # Bulk UPDATE without fetching (efficient for many rows)
    from sqlalchemy import update

    stmt = (
        update(User)
        .where(User.is_active == False)
        .values(is_active=True)
    )
    session.execute(stmt)
    session.commit()
```

#### Delete

```python
with Session(engine) as session:
    # Fetch-then-delete
    user = session.get(User, 1)
    if user:
        session.delete(user)
        session.commit()

    # Bulk DELETE without fetching
    from sqlalchemy import delete

    stmt = delete(User).where(User.is_active == False)
    session.execute(stmt)
    session.commit()
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

### `session.execute()` -- 2.0 Style

In 2.0 style, all queries go through `session.execute()` with `select()`, `insert()`, `update()`, and `delete()` constructs.

```python
from sqlalchemy import select, insert, update, delete

with Session(engine) as session:
    # SELECT
    stmt = select(User).where(User.name == "Alice")
    result = session.execute(stmt)
    user = result.scalar_one()

    # INSERT (returns the inserted rows on supporting backends)
    stmt = insert(User).values(name="Dave", email="dave@example.com")
    result = session.execute(stmt)
    session.commit()

    # UPDATE with RETURNING (PostgreSQL, SQLite 3.35+)
    stmt = (
        update(User)
        .where(User.id == 1)
        .values(name="Alice V2")
        .returning(User.id, User.name)
    )
    result = session.execute(stmt)
    row = result.one()
    session.commit()

    # DELETE
    stmt = delete(User).where(User.is_active == False)
    session.execute(stmt)
    session.commit()
```

---

## Querying (2.0 Style)

All queries use `select()` and `session.execute()`. The legacy `session.query()` API is deprecated.

### `select()` -- The Main Query Construct

```python
from sqlalchemy import select

# Select entire ORM entity
stmt = select(User)

# Select specific columns
stmt = select(User.name, User.email)

# Execute and get results
with Session(engine) as session:
    # Full entities
    users = session.execute(select(User)).scalars().all()

    # Specific columns (returns Row objects, not ORM entities)
    rows = session.execute(select(User.name, User.email)).all()
    for row in rows:
        print(row.name, row.email)  # Named tuple-like access
```

### `.where()` and `.filter_by()`

```python
# .where() -- uses SQL expressions (column operators)
stmt = select(User).where(User.name == "Alice")
stmt = select(User).where(User.age >= 18, User.is_active == True)  # AND
stmt = select(User).where(User.name.like("%ali%"))
stmt = select(User).where(User.name.ilike("%ali%"))  # Case-insensitive
stmt = select(User).where(User.id.in_([1, 2, 3]))
stmt = select(User).where(User.email.is_(None))       # IS NULL
stmt = select(User).where(User.email.is_not(None))    # IS NOT NULL
stmt = select(User).where(User.name.startswith("A"))
stmt = select(User).where(User.name.contains("lic"))
stmt = select(User).where(User.age.between(18, 65))

# OR conditions
from sqlalchemy import or_, and_, not_
stmt = select(User).where(or_(User.name == "Alice", User.name == "Bob"))
stmt = select(User).where(and_(User.age >= 18, or_(User.role == "admin", User.role == "mod")))
stmt = select(User).where(not_(User.is_active))

# .filter_by() -- uses keyword arguments (simpler for equality checks)
stmt = select(User).filter_by(name="Alice", is_active=True)

# Multiple .where() calls are ANDed together
stmt = select(User).where(User.age >= 18).where(User.is_active == True)
```

### `.join()` and `.outerjoin()`

```python
# Implicit join condition (uses foreign key)
stmt = select(User).join(Post)  # INNER JOIN posts ON users.id = posts.author_id

# Explicit join condition
stmt = select(User).join(Post, User.id == Post.author_id)

# Join on relationship attribute
stmt = select(User).join(User.posts)

# LEFT OUTER JOIN
stmt = select(User).outerjoin(Post)
stmt = select(User).outerjoin(User.posts)

# Multiple joins
stmt = (
    select(User)
    .join(User.posts)
    .join(Post.comments)
    .where(Comment.text.contains("great"))
)

# Join with aliased (for self-joins or joining same table multiple times)
from sqlalchemy.orm import aliased

manager = aliased(Employee, name="manager")
stmt = (
    select(Employee, manager)
    .join(manager, Employee.manager_id == manager.id)
)
```

### `.order_by()`, `.group_by()`, `.having()`

```python
from sqlalchemy import func, desc, asc

# Order by
stmt = select(User).order_by(User.name)              # ASC (default)
stmt = select(User).order_by(User.name.desc())        # DESC
stmt = select(User).order_by(desc(User.created_at))   # Alternative DESC syntax
stmt = select(User).order_by(User.last_name, User.first_name)  # Multiple

# Group by with aggregation
stmt = (
    select(User.department_id, func.count(User.id).label("user_count"))
    .group_by(User.department_id)
)

# Having (filter on aggregated values)
stmt = (
    select(User.department_id, func.count(User.id).label("user_count"))
    .group_by(User.department_id)
    .having(func.count(User.id) > 5)
)
```

### `.limit()` and `.offset()`

```python
# Pagination
page = 2
page_size = 20
stmt = (
    select(User)
    .order_by(User.id)
    .limit(page_size)
    .offset((page - 1) * page_size)
)
```

### `func.*` -- SQL Functions

```python
from sqlalchemy import func

# Common aggregate functions
stmt = select(func.count(User.id))                        # COUNT(users.id)
stmt = select(func.count()).select_from(User)              # COUNT(*)
stmt = select(func.sum(Order.total))                       # SUM
stmt = select(func.avg(Order.total))                       # AVG
stmt = select(func.min(Order.total), func.max(Order.total))  # MIN, MAX

# String functions
stmt = select(func.lower(User.name))
stmt = select(func.upper(User.name))
stmt = select(func.length(User.name))
stmt = select(func.coalesce(User.nickname, User.name))     # COALESCE

# Date functions
stmt = select(func.now())
stmt = select(func.date(User.created_at))
stmt = select(func.extract("year", User.created_at))

# Window functions
from sqlalchemy import over
stmt = select(
    User.name,
    func.row_number().over(
        partition_by=User.department_id,
        order_by=User.created_at.desc(),
    ).label("row_num"),
)

# Execute scalar aggregate
with Session(engine) as session:
    count = session.execute(select(func.count()).select_from(User)).scalar()
```

### Subqueries and CTEs

```python
# Subquery
subq = (
    select(func.max(Post.created_at).label("latest_post"))
    .where(Post.author_id == User.id)
    .correlate(User)
    .scalar_subquery()
)
stmt = select(User.name, subq.label("latest_post_date"))

# Subquery as a derived table
subq = (
    select(
        Post.author_id,
        func.count(Post.id).label("post_count"),
    )
    .group_by(Post.author_id)
    .subquery()
)
stmt = (
    select(User.name, subq.c.post_count)
    .join(subq, User.id == subq.c.author_id)
)

# Common Table Expression (CTE)
cte = (
    select(
        Post.author_id,
        func.count(Post.id).label("post_count"),
    )
    .group_by(Post.author_id)
    .cte("post_counts")
)
stmt = (
    select(User.name, cte.c.post_count)
    .join(cte, User.id == cte.c.author_id)
    .where(cte.c.post_count > 10)
)

# Recursive CTE (e.g., for tree traversal)
hierarchy = (
    select(Category.id, Category.name, Category.parent_id)
    .where(Category.parent_id.is_(None))
    .cte("hierarchy", recursive=True)
)
hierarchy = hierarchy.union_all(
    select(Category.id, Category.name, Category.parent_id)
    .join(hierarchy, Category.parent_id == hierarchy.c.id)
)
stmt = select(hierarchy)
```

### EXISTS

```python
from sqlalchemy import exists

# Users who have at least one post
subq = exists().where(Post.author_id == User.id)
stmt = select(User).where(subq)

# Or using .has() on the relationship (ORM shorthand)
stmt = select(User).where(User.posts.any())

# Users who have a post with a specific title
stmt = select(User).where(User.posts.any(Post.title == "Hello"))

# Inverse: users with NO posts
stmt = select(User).where(~User.posts.any())
```

### CASE and Type Casting

```python
from sqlalchemy import case, cast, String

# CASE expression
stmt = select(
    User.name,
    case(
        (User.age < 18, "minor"),
        (User.age < 65, "adult"),
        else_="senior",
    ).label("age_group"),
)

# Type casting
stmt = select(cast(User.id, String))
```

---

## Eager Loading

By default, relationships are **lazily loaded** -- accessing `user.posts` triggers a separate SQL query. This causes the N+1 problem when iterating over many objects. Eager loading strategies solve this.

### `selectinload()` -- Separate SELECT per Relationship

Loads related objects using a separate `SELECT ... WHERE id IN (...)` query. Best general-purpose strategy.

```python
from sqlalchemy.orm import selectinload

stmt = (
    select(User)
    .options(selectinload(User.posts))
)

with Session(engine) as session:
    users = session.execute(stmt).scalars().all()
    # The posts for ALL returned users were loaded in a single extra query
    for user in users:
        print(user.posts)  # No additional SQL
```

**Nested eager loading:**

```python
stmt = (
    select(User)
    .options(
        selectinload(User.posts).selectinload(Post.comments)
    )
)
```

### `joinedload()` -- JOIN-Based Loading

Loads related objects using a JOIN in the same query. Efficient for small result sets or one-to-one/many-to-one.

```python
from sqlalchemy.orm import joinedload

# Many-to-one (efficient -- no row duplication)
stmt = (
    select(Post)
    .options(joinedload(Post.author))
)

# One-to-many (may duplicate parent rows -- use selectinload instead for large sets)
stmt = (
    select(User)
    .options(joinedload(User.posts))
)
```

**Warning:** `joinedload()` for one-to-many relationships produces cartesian-like joins that duplicate parent rows. SQLAlchemy deduplicates in Python, but it transfers more data. Prefer `selectinload()` for one-to-many.

### `subqueryload()` -- Subquery-Based

Similar to `selectinload()` but uses a subquery instead of `IN`. Useful when the parent query is complex.

```python
from sqlalchemy.orm import subqueryload

stmt = (
    select(User)
    .options(subqueryload(User.posts))
)
```

### `lazyload()` -- Explicit Lazy

Overrides the default or a class-level loading strategy back to lazy loading for a specific query.

```python
from sqlalchemy.orm import lazyload

stmt = (
    select(User)
    .options(lazyload(User.posts))  # Force lazy even if class default is eager
)
```

### `contains_eager()` -- For Manual Joins

When you write an explicit `.join()`, use `contains_eager()` to tell SQLAlchemy the relationship data is already in the result (so it does not issue additional queries).

```python
from sqlalchemy.orm import contains_eager

stmt = (
    select(User)
    .join(User.posts)
    .where(Post.is_published == True)
    .options(contains_eager(User.posts))
)

with Session(engine) as session:
    users = session.execute(stmt).unique().scalars().all()
    # user.posts is populated from the JOIN, but only contains published posts
```

**Important:** When using `contains_eager()` with one-to-many joins, you must call `.unique()` on the result to deduplicate parent entities.

### `raiseload()` -- Prevent Lazy Loading

Raises an error if a relationship is accessed without being eagerly loaded. Useful for catching N+1 queries.

```python
from sqlalchemy.orm import raiseload

# Raise on any lazy load for User.posts
stmt = (
    select(User)
    .options(raiseload(User.posts))
)

with Session(engine) as session:
    user = session.execute(stmt).scalars().first()
    user.posts  # Raises InvalidRequestError!

# Raise on ALL relationships
stmt = (
    select(User)
    .options(raiseload("*"))
)
```

### Default Loading Strategy on the Model

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

    posts: Mapped[list["Post"]] = relationship(
        back_populates="author",
        lazy="selectin",  # Default eager loading for this relationship
        # Options: "select" (lazy, default), "selectin", "joined", "subquery",
        #          "raise", "raise_on_sql", "noload", "dynamic" (legacy), "write_only"
    )
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

## Gotchas and Common Mistakes

### 1. Session Lifecycle Management

**Problem:** Using a single long-lived Session for the entire application.

**Why it breaks:** Sessions hold an identity map and a transaction. A long-lived session accumulates stale objects, grows in memory, and holds database locks.

**Fix:** Use short-lived sessions scoped to a unit of work (e.g., one per web request).

```python
# WRONG -- session lives forever
session = Session(engine)
# ... used throughout the application ...

# RIGHT -- session per unit of work
def handle_request():
    with Session(engine) as session:
        with session.begin():
            # do work
            pass
    # session is closed, objects are detached
```

### 2. N+1 Query Problem (Lazy Loading)

**Problem:** Iterating over N parent objects and accessing a lazy-loaded relationship on each triggers N additional queries.

```python
# BAD -- N+1: 1 query for users + N queries for posts
users = session.execute(select(User)).scalars().all()
for user in users:
    print(len(user.posts))  # Each access triggers a SELECT

# GOOD -- 2 queries total: 1 for users + 1 for all related posts
users = session.execute(
    select(User).options(selectinload(User.posts))
).scalars().all()
for user in users:
    print(len(user.posts))  # Already loaded, no SQL
```

### 3. Detached Instance Errors

**Problem:** Accessing a lazy-loaded attribute on an object after the Session is closed.

```python
# BAD
with Session(engine) as session:
    user = session.execute(select(User).where(User.id == 1)).scalar_one()

# Session is now closed, user is "detached"
print(user.posts)  # DetachedInstanceError!

# FIX 1: Eager load before closing
with Session(engine) as session:
    user = session.execute(
        select(User).where(User.id == 1).options(selectinload(User.posts))
    ).scalar_one()
print(user.posts)  # Works -- already loaded

# FIX 2: Keep the session open while you need the data
with Session(engine) as session:
    user = session.execute(select(User).where(User.id == 1)).scalar_one()
    posts = user.posts  # Lazy load happens within session
    # ... use posts ...
```

### 4. `expire_on_commit` Behavior

By default, all objects are **expired** after `session.commit()`. This means accessing any attribute triggers a fresh SELECT.

```python
with Session(engine) as session:
    user = User(name="Alice")
    session.add(user)
    session.commit()

    # user is now expired -- next access triggers a SELECT
    print(user.name)  # Issues: SELECT users.name FROM users WHERE users.id = ?

# To avoid this (e.g., for returning data in an API response):
SessionFactory = sessionmaker(bind=engine, expire_on_commit=False)

with SessionFactory() as session:
    user = User(name="Alice")
    session.add(user)
    session.commit()

    print(user.name)  # No SQL -- uses cached value
```

**Trade-off:** `expire_on_commit=False` means your in-memory objects may diverge from the database if other processes modify rows concurrently.

### 5. Transaction Isolation Surprises

**Problem:** Within a transaction, you only see data as of the transaction's start (depending on isolation level). This can cause "phantom reads" or stale data in long transactions.

```python
# If another process inserts a row after your transaction starts,
# you will NOT see it (under REPEATABLE READ or SERIALIZABLE).
# Under READ COMMITTED (PostgreSQL default), you WILL see it on the next query.

# To force a fresh read within the same transaction:
session.expire_all()  # Expire all cached objects
# Next access will issue fresh SELECTs
```

### 6. Bulk Operations Performance

The ORM's unit-of-work pattern adds overhead per row. For bulk operations on thousands+ of rows, use Core-level bulk operations.

```python
# SLOW -- ORM unit of work (tracks each object individually)
for i in range(10000):
    session.add(User(name=f"user_{i}"))
session.commit()

# FAST -- Bulk insert using Core
from sqlalchemy import insert
session.execute(
    insert(User),
    [{"name": f"user_{i}"} for i in range(10000)],
)
session.commit()

# FAST -- Bulk update using Core
from sqlalchemy import update
session.execute(
    update(User).where(User.is_active == False).values(is_active=True)
)
session.commit()
```

### 7. 1.x vs 2.0 API Differences

| Feature | 1.x Style (deprecated) | 2.0 Style |
|---|---|---|
| Base class | `declarative_base()` | `class Base(DeclarativeBase)` |
| Column declaration | `Column(Integer, primary_key=True)` | `Mapped[int] = mapped_column(primary_key=True)` |
| Query | `session.query(User).filter(...)` | `session.execute(select(User).where(...))` |
| Get by PK | `session.query(User).get(1)` | `session.get(User, 1)` |
| Results | `session.query(User).all()` | `session.execute(select(User)).scalars().all()` |
| Relationship type | `relationship("Post")` | `Mapped[list["Post"]] = relationship()` |

**Key migration rule:** If your code uses `session.query()`, it is 1.x style. Migrate to `session.execute(select(...))`.

### 8. Forgetting `.scalars()` on ORM Queries

```python
# Returns Row objects containing (User,) tuples
result = session.execute(select(User)).all()
# result[0] is a Row, not a User!

# Returns User objects directly
result = session.execute(select(User)).scalars().all()
# result[0] is a User
```

### 9. Implicit vs Explicit Autoflush

Sessions auto-flush before queries by default. This can trigger unexpected INSERTs/UPDATEs.

```python
user = User(name="test")
session.add(user)
# No flush yet...

# This SELECT triggers an autoflush! The INSERT for "test" is sent first.
result = session.execute(select(User).where(User.name == "test"))

# To disable autoflush temporarily:
with session.no_autoflush:
    result = session.execute(select(User).where(User.name == "test"))
```

---

## Complete Code Examples

### Full Application with Models, Relationships, and CRUD

```python
"""
Complete SQLAlchemy 2.0 example: Blog application with users, posts,
comments, and tags.
"""
from __future__ import annotations

import enum
from datetime import datetime
from typing import Optional

from sqlalchemy import (
    Column,
    Enum as SAEnum,
    ForeignKey,
    Index,
    Integer,
    String,
    Table,
    Text,
    create_engine,
    func,
    select,
)
from sqlalchemy.orm import (
    DeclarativeBase,
    Mapped,
    Session,
    mapped_column,
    relationship,
    selectinload,
    sessionmaker,
)


# ── Base ─────────────────────────────────────────────────────────────

class Base(DeclarativeBase):
    pass


# ── Enums ────────────────────────────────────────────────────────────

class PostStatus(enum.Enum):
    DRAFT = "draft"
    PUBLISHED = "published"
    ARCHIVED = "archived"


# ── Mixins ───────────────────────────────────────────────────────────

class TimestampMixin:
    """Mixin that adds created_at and updated_at columns."""
    created_at: Mapped[datetime] = mapped_column(
        default=func.now(), server_default=func.now()
    )
    updated_at: Mapped[Optional[datetime]] = mapped_column(
        onupdate=func.now(), server_onupdate=func.now(), default=None
    )


# ── Association Table ────────────────────────────────────────────────

post_tags = Table(
    "post_tags",
    Base.metadata,
    Column("post_id", Integer, ForeignKey("posts.id", ondelete="CASCADE"), primary_key=True),
    Column("tag_id", Integer, ForeignKey("tags.id", ondelete="CASCADE"), primary_key=True),
)


# ── Models ───────────────────────────────────────────────────────────

class User(TimestampMixin, Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, index=True)
    email: Mapped[str] = mapped_column(String(255), unique=True)
    display_name: Mapped[Optional[str]] = mapped_column(String(100))
    is_active: Mapped[bool] = mapped_column(default=True)

    # Relationships
    posts: Mapped[list["Post"]] = relationship(
        back_populates="author", cascade="all, delete-orphan"
    )
    comments: Mapped[list["Comment"]] = relationship(
        back_populates="author", cascade="all, delete-orphan"
    )

    def __repr__(self) -> str:
        return f"User(id={self.id!r}, username={self.username!r})"


class Post(TimestampMixin, Base):
    __tablename__ = "posts"
    __table_args__ = (
        Index("ix_posts_author_status", "author_id", "status"),
    )

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    slug: Mapped[str] = mapped_column(String(200), unique=True, index=True)
    body: Mapped[str] = mapped_column(Text)
    status: Mapped[PostStatus] = mapped_column(
        SAEnum(PostStatus), default=PostStatus.DRAFT
    )
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))

    # Relationships
    author: Mapped["User"] = relationship(back_populates="posts")
    comments: Mapped[list["Comment"]] = relationship(
        back_populates="post", cascade="all, delete-orphan"
    )
    tags: Mapped[list["Tag"]] = relationship(
        secondary=post_tags, back_populates="posts"
    )

    def __repr__(self) -> str:
        return f"Post(id={self.id!r}, title={self.title!r})"


class Comment(TimestampMixin, Base):
    __tablename__ = "comments"

    id: Mapped[int] = mapped_column(primary_key=True)
    body: Mapped[str] = mapped_column(Text)
    post_id: Mapped[int] = mapped_column(ForeignKey("posts.id"))
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))

    # Self-referential: threaded comments
    parent_id: Mapped[Optional[int]] = mapped_column(ForeignKey("comments.id"))

    # Relationships
    post: Mapped["Post"] = relationship(back_populates="comments")
    author: Mapped["User"] = relationship(back_populates="comments")
    replies: Mapped[list["Comment"]] = relationship(
        back_populates="parent", cascade="all, delete-orphan"
    )
    parent: Mapped[Optional["Comment"]] = relationship(
        back_populates="replies", remote_side=[id]
    )

    def __repr__(self) -> str:
        return f"Comment(id={self.id!r}, post_id={self.post_id!r})"


class Tag(Base):
    __tablename__ = "tags"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)

    posts: Mapped[list["Post"]] = relationship(
        secondary=post_tags, back_populates="tags"
    )

    def __repr__(self) -> str:
        return f"Tag(id={self.id!r}, name={self.name!r})"


# ── Engine and Session Setup ─────────────────────────────────────────

engine = create_engine("sqlite:///blog.db", echo=True)
Base.metadata.create_all(engine)

SessionFactory = sessionmaker(bind=engine)


# ── CRUD Operations ──────────────────────────────────────────────────

def create_sample_data():
    """Seed the database with sample data."""
    with SessionFactory.begin() as session:
        # Create users
        alice = User(username="alice", email="alice@example.com", display_name="Alice")
        bob = User(username="bob", email="bob@example.com", display_name="Bob")

        # Create tags
        python_tag = Tag(name="python")
        sqlalchemy_tag = Tag(name="sqlalchemy")
        tutorial_tag = Tag(name="tutorial")

        # Create posts
        post1 = Post(
            title="Getting Started with SQLAlchemy 2.0",
            slug="getting-started-sqlalchemy-2",
            body="SQLAlchemy 2.0 introduces a modern API...",
            status=PostStatus.PUBLISHED,
            author=alice,
            tags=[python_tag, sqlalchemy_tag, tutorial_tag],
        )
        post2 = Post(
            title="Advanced Querying Techniques",
            slug="advanced-querying",
            body="In this post we explore subqueries, CTEs...",
            status=PostStatus.DRAFT,
            author=alice,
            tags=[sqlalchemy_tag],
        )
        post3 = Post(
            title="Python Best Practices",
            slug="python-best-practices",
            body="Here are some best practices for Python...",
            status=PostStatus.PUBLISHED,
            author=bob,
            tags=[python_tag],
        )

        # Create comments (including threaded reply)
        comment1 = Comment(body="Great article!", post=post1, author=bob)
        reply1 = Comment(
            body="Thanks Bob!", post=post1, author=alice, parent=comment1
        )

        session.add_all([alice, bob, post1, post2, post3])


def query_examples():
    """Demonstrate various 2.0-style queries."""
    with SessionFactory() as session:

        # 1. Simple select with eager loading
        print("=== All users with their posts ===")
        stmt = select(User).options(selectinload(User.posts))
        users = session.execute(stmt).scalars().all()
        for user in users:
            print(f"{user.username}: {[p.title for p in user.posts]}")

        # 2. Filter with join
        print("\n=== Published posts by alice ===")
        stmt = (
            select(Post)
            .join(Post.author)
            .where(User.username == "alice")
            .where(Post.status == PostStatus.PUBLISHED)
            .order_by(Post.created_at.desc())
        )
        posts = session.execute(stmt).scalars().all()
        for post in posts:
            print(f"  {post.title} ({post.status.value})")

        # 3. Aggregate query
        print("\n=== Post count per user ===")
        stmt = (
            select(User.username, func.count(Post.id).label("post_count"))
            .join(User.posts)
            .group_by(User.username)
            .order_by(func.count(Post.id).desc())
        )
        for row in session.execute(stmt):
            print(f"  {row.username}: {row.post_count} posts")

        # 4. Posts with a specific tag
        print("\n=== Posts tagged 'python' ===")
        stmt = (
            select(Post)
            .join(Post.tags)
            .where(Tag.name == "python")
            .options(selectinload(Post.tags), selectinload(Post.author))
        )
        posts = session.execute(stmt).scalars().unique().all()
        for post in posts:
            tag_names = [t.name for t in post.tags]
            print(f"  {post.title} by {post.author.username} [{', '.join(tag_names)}]")

        # 5. Subquery: users who have published posts
        print("\n=== Users with published posts ===")
        subq = (
            select(Post.author_id)
            .where(Post.status == PostStatus.PUBLISHED)
            .distinct()
            .subquery()
        )
        stmt = select(User).where(User.id.in_(select(subq.c.author_id)))
        users = session.execute(stmt).scalars().all()
        for user in users:
            print(f"  {user.username}")

        # 6. Comments with threaded replies (eager loaded)
        print("\n=== Comments with replies ===")
        stmt = (
            select(Comment)
            .where(Comment.parent_id.is_(None))
            .options(
                selectinload(Comment.replies),
                selectinload(Comment.author),
            )
        )
        comments = session.execute(stmt).scalars().all()
        for comment in comments:
            print(f"  {comment.author.username}: {comment.body}")
            for reply in comment.replies:
                print(f"    -> {reply.body}")


def update_example():
    """Demonstrate updates."""
    with SessionFactory.begin() as session:
        # Fetch and modify
        user = session.execute(
            select(User).where(User.username == "alice")
        ).scalar_one()
        user.display_name = "Alice Wonderland"

        # Bulk update
        from sqlalchemy import update
        session.execute(
            update(Post)
            .where(Post.status == PostStatus.DRAFT)
            .values(status=PostStatus.ARCHIVED)
        )


def delete_example():
    """Demonstrate deletes with cascade."""
    with SessionFactory.begin() as session:
        # Deleting a user cascades to their posts and comments
        user = session.execute(
            select(User).where(User.username == "bob")
        ).scalar_one_or_none()
        if user:
            session.delete(user)
            # All of bob's posts and comments are deleted via cascade


# ── Run ──────────────────────────────────────────────────────────────

if __name__ == "__main__":
    create_sample_data()
    query_examples()
    update_example()
```

### JSON Column Usage Example

```python
from sqlalchemy import JSON, select
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column

class Base(DeclarativeBase):
    pass

class UserPreferences(Base):
    __tablename__ = "user_preferences"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(unique=True)
    preferences: Mapped[dict] = mapped_column(JSON, default=dict)

# Create and populate
with Session(engine) as session:
    prefs = UserPreferences(
        user_id=1,
        preferences={
            "theme": "dark",
            "notifications": {"email": True, "push": False},
            "language": "en",
            "favorites": ["dashboard", "reports"],
        },
    )
    session.add(prefs)
    session.commit()

# Query JSON fields
with Session(engine) as session:
    # Access nested JSON keys
    stmt = select(UserPreferences).where(
        UserPreferences.preferences["theme"].as_string() == "dark"
    )
    result = session.execute(stmt).scalars().all()

    # IMPORTANT: When modifying a JSON column, you must either:
    # 1. Replace the entire dict (triggers change detection)
    prefs = session.get(UserPreferences, 1)
    new_prefs = dict(prefs.preferences)
    new_prefs["theme"] = "light"
    prefs.preferences = new_prefs  # Reassign to trigger change detection
    session.commit()

    # 2. Or flag the attribute as modified
    from sqlalchemy.orm.attributes import flag_modified
    prefs.preferences["theme"] = "dark"
    flag_modified(prefs, "preferences")
    session.commit()
```

### Mixin Pattern: TimestampMixin and SoftDeleteMixin

```python
from __future__ import annotations

from datetime import datetime
from typing import Optional

from sqlalchemy import DateTime, Boolean, func, select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class TimestampMixin:
    """Adds created_at and updated_at to any model."""
    created_at: Mapped[datetime] = mapped_column(
        DateTime,
        default=func.now(),
        server_default=func.now(),
    )
    updated_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime,
        default=None,
        onupdate=func.now(),
    )


class SoftDeleteMixin:
    """Adds soft delete capability via is_deleted flag."""
    is_deleted: Mapped[bool] = mapped_column(Boolean, default=False, index=True)
    deleted_at: Mapped[Optional[datetime]] = mapped_column(DateTime, default=None)

    def soft_delete(self):
        self.is_deleted = True
        self.deleted_at = datetime.utcnow()

    @classmethod
    def active(cls):
        """Return a where clause filtering to non-deleted rows."""
        return cls.is_deleted == False


class Article(TimestampMixin, SoftDeleteMixin, Base):
    __tablename__ = "articles"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    body: Mapped[str] = mapped_column(Text)


# Usage
with Session(engine) as session:
    # Query only active (not soft-deleted) articles
    stmt = select(Article).where(Article.active()).order_by(Article.created_at.desc())
    articles = session.execute(stmt).scalars().all()

    # Soft delete an article
    article = session.get(Article, 1)
    article.soft_delete()
    session.commit()
```

### Async Usage

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

### Result Processing Cheat Sheet

```python
result = session.execute(stmt)

# Single ORM entity queries (select(User)):
result.scalars().all()           # list[User]
result.scalars().first()         # User | None
result.scalar_one()              # User (raises if not exactly 1)
result.scalar_one_or_none()      # User | None (raises if > 1)
result.scalars()                 # Iterator[User]

# Multi-column queries (select(User.name, User.email)):
result.all()                     # list[Row]  -- Row has .name, .email
result.first()                   # Row | None
result.one()                     # Row (raises if not exactly 1)
result.one_or_none()             # Row | None
result.columns("name")          # Re-map to just one column

# Aggregate / scalar queries (select(func.count())):
result.scalar()                  # The single value (e.g., 42)
result.scalar_one()              # Same but raises if not exactly 1 row
```
