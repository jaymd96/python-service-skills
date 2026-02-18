# SQLAlchemy — Models & Schema

> Part of the sqlalchemy skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [ORM Declarative 2.0 Style](#orm-declarative-20-style)
  - [DeclarativeBase](#declarativebase----the-modern-base-class)
  - [Mapped\[type\]](#mappedtype----typed-column-declarations)
  - [mapped\_column()](#mapped_column----column-configuration)
  - [Full mapped\_column() Parameter Reference](#full-mapped_column-parameter-reference)
  - [relationship()](#relationship----orm-relationships)
- [Column Types](#column-types)
  - [Standard Types](#standard-types)
  - [Specialized Types](#specialized-types)
  - [Column Type Usage Examples](#column-type-usage-examples)
  - [Working with JSON Columns](#working-with-json-columns)
  - [Working with Enum Columns](#working-with-enum-columns)
- [Relationships](#relationships)
  - [One-to-Many / Many-to-One](#one-to-many--many-to-one)
  - [One-to-One](#one-to-one)
  - [Many-to-Many](#many-to-many)
  - [Many-to-Many with Association Object](#many-to-many-with-association-object-extra-columns)
  - [Self-Referential (Adjacency List)](#self-referential-adjacency-list)
  - [back\_populates vs backref](#back_populates-vs-backref)
  - [Multiple Foreign Keys to the Same Table](#multiple-foreign-keys-to-the-same-table)
  - [Cascade Options](#cascade-options)
- [Mixin Pattern](#mixin-pattern-timestampmixin-and-softdeletemixin)

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

## Mixin Pattern: TimestampMixin and SoftDeleteMixin

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
