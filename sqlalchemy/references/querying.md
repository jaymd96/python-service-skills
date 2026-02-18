# SQLAlchemy — Querying

> Part of the sqlalchemy skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [select() -- The Main Query Construct](#select----the-main-query-construct)
- [.where() and .filter\_by()](#where-and-filter_by)
- [.join() and .outerjoin()](#join-and-outerjoin)
- [.order\_by(), .group\_by(), .having()](#order_by-group_by-having)
- [.limit() and .offset()](#limit-and-offset)
- [func.\* -- SQL Functions](#func----sql-functions)
- [Subqueries and CTEs](#subqueries-and-ctes)
- [EXISTS](#exists)
- [CASE and Type Casting](#case-and-type-casting)
- [Eager Loading](#eager-loading)
  - [selectinload()](#selectinload----separate-select-per-relationship)
  - [joinedload()](#joinedload----join-based-loading)
  - [subqueryload()](#subqueryload----subquery-based)
  - [lazyload()](#lazyload----explicit-lazy)
  - [contains\_eager()](#contains_eager----for-manual-joins)
  - [raiseload()](#raiseload----prevent-lazy-loading)
  - [Default Loading Strategy on the Model](#default-loading-strategy-on-the-model)
- [CRUD Operations](#crud-operations)
  - [Create](#create)
  - [Read](#read)
  - [Update](#update)
  - [Delete](#delete)
- [session.execute() -- 2.0 Style](#sessionexecute----20-style)
- [Result Processing Cheat Sheet](#result-processing-cheat-sheet)

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

## CRUD Operations

### Create

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

### Read

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

### Update

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

### Delete

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

---

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
