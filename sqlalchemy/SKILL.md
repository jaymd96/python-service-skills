---
name: sqlalchemy
description: SQLAlchemy 2.0 ORM and SQL toolkit. Use when defining database models with DeclarativeBase and Mapped[], writing queries with select(), configuring relationships, eager loading, JSON columns, or session management. Triggers on SQLAlchemy, ORM models, database queries, mapped_column, relationships, selectinload, joinedload, session.
---

# SQLAlchemy 2.0 — ORM and SQL Toolkit (v2.0.40)

## Quick Start

```bash
pip install sqlalchemy
```

```python
from sqlalchemy import create_engine, String, select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, Session

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str | None]

engine = create_engine("sqlite:///app.db")
Base.metadata.create_all(engine)
```

## Key Patterns

### Querying (2.0 style — NO session.query())
```python
with Session(engine) as session:
    stmt = select(User).where(User.name == "Alice")
    user = session.scalars(stmt).first()

    # Eager loading (always use to avoid N+1)
    from sqlalchemy.orm import selectinload
    stmt = select(User).options(selectinload(User.posts))
    users = session.scalars(stmt).all()
```

### Relationships (2.0 style)
```python
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    posts: Mapped[list["Post"]] = relationship(back_populates="author")

class Post(Base):
    __tablename__ = "posts"
    id: Mapped[int] = mapped_column(primary_key=True)
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    author: Mapped["User"] = relationship(back_populates="posts")
```

## References

- **[models.md](references/models.md)** — Model definition, Mapped[], relationships, column types, constraints, mixins
- **[querying.md](references/querying.md)** — select(), eager loading, filtering, joins, aggregation, result processing
- **[advanced.md](references/advanced.md)** — Session management, events, hybrid properties, JSON/Enum columns, Alembic, async
- **[examples.md](references/examples.md)** — Complete examples, gotchas, migration patterns

## Grep Patterns

- `DeclarativeBase|Mapped\[` — Find model definitions
- `select\(.*\)\.where` — Find query patterns
- `selectinload|joinedload` — Find eager loading
- `Session\(|session\.` — Find session usage
