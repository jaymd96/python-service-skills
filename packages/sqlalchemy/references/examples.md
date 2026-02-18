# SQLAlchemy — Examples & Gotchas

> Part of the sqlalchemy skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Session Lifecycle Management](#1-session-lifecycle-management)
  - [2. N+1 Query Problem (Lazy Loading)](#2-n1-query-problem-lazy-loading)
  - [3. Detached Instance Errors](#3-detached-instance-errors)
  - [4. expire\_on\_commit Behavior](#4-expire_on_commit-behavior)
  - [5. Transaction Isolation Surprises](#5-transaction-isolation-surprises)
  - [6. Bulk Operations Performance](#6-bulk-operations-performance)
  - [7. 1.x vs 2.0 API Differences](#7-1x-vs-20-api-differences)
  - [8. Forgetting .scalars() on ORM Queries](#8-forgetting-scalars-on-orm-queries)
  - [9. Implicit vs Explicit Autoflush](#9-implicit-vs-explicit-autoflush)
- [Complete Code Examples](#complete-code-examples)
  - [Full Application with Models, Relationships, and CRUD](#full-application-with-models-relationships-and-crud)
  - [JSON Column Usage Example](#json-column-usage-example)

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
