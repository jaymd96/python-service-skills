# dynaconf — Examples & Gotchas

> Part of the dynaconf skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Complete Code Examples](#complete-code-examples)
  - [Twelve-Factor Application Configuration](#twelve-factor-application-configuration)
  - [Usage in Application Code](#usage-in-application-code)
  - [Dynamic Settings Reloading](#dynamic-settings-reloading)
- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Case Insensitivity](#1-case-insensitivity)
  - [2. Environments Must Be Explicitly Enabled](#2-environments-must-be-explicitly-enabled)
  - [3. Environment Variable Prefix](#3-environment-variable-prefix)
  - [4. Missing @int / @float Markers](#4-missing-int--float-markers)
  - [5. .secrets.* Files Must Be in .gitignore](#5-secrets-files-must-be-in-gitignore)
  - [6. Merge Behavior Is Not Default](#6-merge-behavior-is-not-default)
  - [7. Validator default Only Sets Missing Values](#7-validator-default-only-sets-missing-values)
  - [8. load_dotenv Requires python-dotenv](#8-load_dotenv-requires-python-dotenv)
  - [9. Django Integration Order](#9-django-integration-order)
  - [10. Accessing Nested Settings](#10-accessing-nested-settings)

## Complete Code Examples

### Twelve-Factor Application Configuration

```python
# config.py
from dynaconf import Dynaconf, Validator

settings = Dynaconf(
    envvar_prefix="APP",
    settings_files=["settings.toml", ".secrets.toml"],
    environments=True,
    env_switcher="APP_ENV",
    load_dotenv=True,
    validators=[
        Validator("DATABASE_URL", must_exist=True),
        Validator("SECRET_KEY", must_exist=True, min_len=32),
        Validator("PORT", is_type_of=int, default=8000, gte=1024),
        Validator("DEBUG", is_type_of=bool, default=False),
        Validator("LOG_LEVEL", is_in=["DEBUG", "INFO", "WARNING", "ERROR"], default="INFO"),
        Validator("ALLOWED_HOSTS", must_exist=True, when=Validator("DEBUG", eq=False)),
    ],
)
```

```toml
# settings.toml
[default]
port = 8000
debug = false
log_level = "INFO"
allowed_hosts = ["localhost"]

[default.database]
pool_size = 5
pool_timeout = 30

[development]
debug = true
log_level = "DEBUG"
database_url = "sqlite:///dev.db"

[production]
log_level = "WARNING"
allowed_hosts = ["myapp.com", "www.myapp.com"]

[production.database]
pool_size = 20
pool_timeout = 10
```

```toml
# .secrets.toml (in .gitignore)
[default]
secret_key = "dev-secret-key-at-least-32-characters-long!"

[production]
secret_key = "@vault secret/data/myapp/SECRET_KEY"
database_url = "postgresql://user:password@prod-db:5432/myapp"
```

### Usage in Application Code

```python
# app.py
from config import settings

def create_app():
    print(f"Starting on port {settings.PORT}")
    print(f"Debug mode: {settings.DEBUG}")
    print(f"Database: {settings.DATABASE_URL}")
    print(f"DB pool size: {settings.DATABASE.POOL_SIZE}")

    if settings.DEBUG:
        print("WARNING: Running in debug mode")

    # Export for inspection
    print(settings.as_dict())
```

### Dynamic Settings Reloading

```python
from config import settings

# Reload settings from all sources
settings.reload()

# Temporarily override settings (useful for testing)
with settings.using_env("testing"):
    assert settings.DEBUG is True
    assert settings.DATABASE_URL == "sqlite:///test.db"
# After the context manager, settings revert to the previous environment
```

---

## Gotchas and Common Mistakes

### 1. Case Insensitivity

dynaconf settings are case-insensitive. `settings.DEBUG`, `settings.debug`, and `settings.Debug` all refer to the same value. Internally, keys are stored as uppercase.

### 2. Environments Must Be Explicitly Enabled

Setting `environments=True` is required for environment sections (`[development]`, `[production]`) to work. Without it, the entire TOML file is loaded as flat key-value pairs and section headers become nested dict keys.

```python
# WRONG: environments sections are treated as nested dicts
settings = Dynaconf(settings_files=["settings.toml"])
settings.development.debug  # Accesses nested dict

# CORRECT: environments are layered
settings = Dynaconf(settings_files=["settings.toml"], environments=True)
settings.DEBUG  # Uses the active environment's value
```

### 3. Environment Variable Prefix

If you change `envvar_prefix`, all environment variables must use the new prefix. The default prefix is `DYNACONF_`.

```python
settings = Dynaconf(envvar_prefix="MYAPP")
# Now: MYAPP_DEBUG=true (not DYNACONF_DEBUG)
```

Setting `envvar_prefix=False` disables environment variable reading entirely.

### 4. Missing `@int` / `@float` Markers

Environment variables are always strings. Without type markers, numeric values remain strings:

```bash
export DYNACONF_PORT=8000        # settings.PORT is "8000" (string!)
export DYNACONF_PORT="@int 8000" # settings.PORT is 8000 (integer)
```

### 5. `.secrets.*` Files Must Be in `.gitignore`

dynaconf will load `.secrets.toml` if listed in `settings_files`, but it does not automatically prevent it from being committed. Add it to `.gitignore` manually.

### 6. Merge Behavior Is Not Default

By default, nested dicts and lists are completely replaced, not merged. If you expect merge behavior, explicitly enable it with `merge_enabled=True` or use `dynaconf_merge` markers.

### 7. Validator `default` Only Sets Missing Values

A validator's `default` only applies if the setting is completely absent. It does not override an explicit `None` or empty string value.

### 8. `load_dotenv` Requires python-dotenv

If you set `load_dotenv=True`, ensure `python-dotenv` is installed. dynaconf does not list it as a hard dependency.

```bash
pip install python-dotenv
```

### 9. Django Integration Order

In Django's `settings.py`, the `dynaconf.DjangoDynaconf(__name__)` call must come at the very bottom of the file, after all standard Django settings are defined. dynaconf overlays on top of the existing settings.

### 10. Accessing Nested Settings

Nested settings use dot notation in code but double underscores in environment variables:

```python
# In code:
settings.DATABASE.HOST

# In environment variables:
export DYNACONF_DATABASE__HOST="localhost"
```

Single underscore in env vars is treated as part of the key name, not as a nesting separator.
