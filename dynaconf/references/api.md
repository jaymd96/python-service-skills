# dynaconf — API Reference

> Part of the dynaconf skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Quick Start with CLI](#quick-start-with-cli)
- [Core API](#core-api)
  - [Dynaconf()](#dynaconf)
  - [Key Constructor Parameters](#key-constructor-parameters)
  - [Accessing Settings](#accessing-settings)
  - [settings.as_dict()](#settingsas_dict)
  - [settings.from_env()](#settingsfrom_env)
- [Settings Files](#settings-files)
  - [TOML (Recommended)](#toml-recommended)
  - [YAML](#yaml)
  - [JSON](#json)
  - [INI](#ini)
  - [.env File](#env-file)
- [Environment Variables](#environment-variables)
  - [Prefix Configuration](#prefix-configuration)
  - [Type Casting with @ Markers](#type-casting-with--markers)
  - [Nested Values](#nested-values)
- [Environment Switching](#environment-switching)
  - [Precedence Order (Lowest to Highest)](#precedence-order-lowest-to-highest)
- [Validators](#validators)
  - [Validator Parameters](#validator-parameters)
- [Secrets Management](#secrets-management)
  - [.secrets.* Files](#secrets-files)
  - [HashiCorp Vault Integration](#hashicorp-vault-integration)
  - [Environment Variable Secrets](#environment-variable-secrets)
- [Flask Integration](#flask-integration)
  - [Flask-Specific Settings File](#flask-specific-settings-file)
- [Django Integration](#django-integration)
  - [Django Environment Variables](#django-environment-variables)
- [Merging Strategies](#merging-strategies)
  - [Default Behavior (Override)](#default-behavior-override)
  - [Merge Mode](#merge-mode)
  - [List Merge with @merge](#list-merge-with-merge)
- [Custom Loaders](#custom-loaders)

## Quick Start with CLI

dynaconf provides a CLI tool to scaffold configuration files:

```bash
# Initialize dynaconf in your project
dynaconf init -f toml

# This creates:
#   config.py        - settings instance
#   settings.toml    - default settings
#   .secrets.toml    - sensitive settings (add to .gitignore)
```

The generated `config.py`:

```python
from dynaconf import Dynaconf

settings = Dynaconf(
    settings_files=["settings.toml", ".secrets.toml"],
)
```

---

## Core API

### `Dynaconf()`

The primary settings class. Create a single instance, typically at the module level.

```python
from dynaconf import Dynaconf

settings = Dynaconf(
    settings_files=["settings.toml", ".secrets.toml"],
    environments=True,
    envvar_prefix="MYAPP",
    env_switcher="MYAPP_ENV",
    load_dotenv=True,
)
```

#### Key Constructor Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `settings_files` | `list[str]` | `[]` | List of settings file paths to load |
| `environments` | `bool` | `False` | Enable layered environments (dev, production, etc.) |
| `envvar_prefix` | `str` | `"DYNACONF"` | Prefix for environment variables |
| `env_switcher` | `str` | `"ENV_FOR_DYNACONF"` | Env var that selects the active environment |
| `load_dotenv` | `bool` | `False` | Load `.env` file into environment |
| `root_path` | `str` | `None` | Root directory for settings file resolution |
| `validators` | `list` | `[]` | List of `Validator` instances |
| `merge_enabled` | `bool` | `False` | Enable global merge mode for nested structures |
| `core_loaders` | `list` | `["YAML", "TOML", ...]` | Loaders to use |
| `preload` | `list` | `[]` | Files to load before the main settings files |
| `includes` | `list` | `[]` | Additional files to include after main loading |

### Accessing Settings

```python
from config import settings

# Attribute-style access (case-insensitive)
settings.DATABASE_URL
settings.database_url  # Same as above

# Dict-style access
settings["DATABASE_URL"]

# With default value
settings.get("MISSING_KEY", "default_value")

# Check existence
"DATABASE_URL" in settings
settings.exists("DATABASE_URL")

# Get as specific type
settings.as_int("PORT")
settings.as_float("RATE")
settings.as_bool("DEBUG")
settings.as_json("COMPLEX_VALUE")
```

### `settings.as_dict()`

Export all settings as a plain dictionary:

```python
all_settings = settings.as_dict()
# {"DATABASE_URL": "...", "DEBUG": True, "PORT": 8000, ...}

# Optionally filter by prefix
db_settings = settings.as_dict(env="production")
```

### `settings.from_env()`

Create a new settings object locked to a specific environment:

```python
production_settings = settings.from_env("production")
print(production_settings.DATABASE_URL)
```

---

## Settings Files

dynaconf supports multiple configuration file formats.

### TOML (Recommended)

```toml
# settings.toml
[default]
debug = false
database_url = "sqlite:///default.db"
allowed_hosts = ["localhost"]

[development]
debug = true
database_url = "sqlite:///dev.db"

[production]
debug = false
database_url = "postgresql://user:pass@db-host/prod"
allowed_hosts = ["example.com", "www.example.com"]
```

### YAML

```yaml
# settings.yaml
default:
  debug: false
  database_url: "sqlite:///default.db"
  allowed_hosts:
    - localhost

development:
  debug: true
  database_url: "sqlite:///dev.db"

production:
  debug: false
  database_url: "postgresql://user:pass@db-host/prod"
```

### JSON

```json
{
  "default": {
    "debug": false,
    "database_url": "sqlite:///default.db"
  },
  "development": {
    "debug": true
  }
}
```

### INI

```ini
[default]
debug = false
database_url = sqlite:///default.db

[development]
debug = true
```

### `.env` File

```bash
# .env
DYNACONF_DATABASE_URL="postgresql://localhost/mydb"
DYNACONF_DEBUG=true
DYNACONF_PORT=@int 8000
DYNACONF_ALLOWED_HOSTS='@json ["localhost", "127.0.0.1"]'
```

---

## Environment Variables

Environment variables override settings from files. By default, they use the `DYNACONF_` prefix.

```bash
# Set via environment
export DYNACONF_DATABASE_URL="postgresql://prod-host/db"
export DYNACONF_DEBUG=false
export DYNACONF_PORT="@int 8000"
```

### Prefix Configuration

```python
settings = Dynaconf(envvar_prefix="MYAPP")
# Now reads: MYAPP_DATABASE_URL, MYAPP_DEBUG, etc.
```

### Type Casting with `@` Markers

Environment variables are strings by default. Use `@` markers for type casting:

```bash
export DYNACONF_PORT="@int 8000"
export DYNACONF_RATE="@float 0.5"
export DYNACONF_DEBUG="@bool true"
export DYNACONF_HOSTS='@json ["host1", "host2"]'
export DYNACONF_CONFIG='@json {"key": "value"}'
```

### Nested Values

Use double underscores for nested keys:

```bash
export DYNACONF_DATABASE__HOST="localhost"
export DYNACONF_DATABASE__PORT="@int 5432"
export DYNACONF_DATABASE__NAME="mydb"
```

```python
settings.DATABASE.HOST   # "localhost"
settings.DATABASE.PORT   # 5432
settings.DATABASE.NAME   # "mydb"
```

---

## Environment Switching

When `environments=True`, dynaconf supports layered environments. Settings from the active environment override the `[default]` section.

```python
settings = Dynaconf(
    settings_files=["settings.toml"],
    environments=True,
    env_switcher="MYAPP_ENV",
)
```

```bash
# Switch environment
export MYAPP_ENV=production
```

### Precedence Order (Lowest to Highest)

1. `[default]` section in settings files
2. Active environment section (e.g., `[production]`)
3. `[global]` section in settings files
4. Environment variables (with prefix)
5. `.secrets.*` files
6. Vault / Redis (if configured)

---

## Validators

Validators enforce that settings meet requirements at startup. They raise `ValidationError` if constraints are violated.

```python
from dynaconf import Dynaconf, Validator

settings = Dynaconf(
    settings_files=["settings.toml"],
    validators=[
        # Ensure DATABASE_URL is set
        Validator("DATABASE_URL", must_exist=True),

        # Ensure DEBUG is a boolean with a default
        Validator("DEBUG", is_type_of=bool, default=False),

        # Ensure PORT is an integer in a range
        Validator("PORT", is_type_of=int, gte=1024, lte=65535, default=8000),

        # Conditional validation
        Validator("DATABASE_URL", must_exist=True, when=Validator("DEBUG", eq=False)),

        # Multiple settings at once
        Validator("API_KEY", "API_SECRET", must_exist=True),

        # Custom validation function
        Validator("EMAIL", condition=lambda v: "@" in v, messages={"condition": "Invalid email"}),

        # Environment-specific validation
        Validator("SECRET_KEY", must_exist=True, env="production"),
    ],
)

# Manually trigger validation (also runs at startup if validators are passed)
settings.validators.validate()
```

### Validator Parameters

| Parameter | Description |
|-----------|-------------|
| `must_exist` | Setting must be defined (not None) |
| `default` | Default value if setting is missing |
| `is_type_of` | Expected Python type |
| `eq`, `ne` | Equal to / not equal to |
| `gt`, `gte`, `lt`, `lte` | Numeric comparisons |
| `is_in`, `is_not_in` | Value must (not) be in a list |
| `len_eq`, `len_min`, `len_max` | Length constraints (strings, lists) |
| `condition` | Custom callable returning True/False |
| `messages` | Custom error messages |
| `when` | Conditional validator (only validate when another condition is met) |
| `env` | Only apply in the specified environment |

---

## Secrets Management

### `.secrets.*` Files

By convention, sensitive settings go in `.secrets.toml` (or `.secrets.yaml`, etc.) which should be in `.gitignore`.

```toml
# .secrets.toml
[default]
secret_key = "super-secret-key-here"
database_password = "s3cur3p@ss"

[production]
secret_key = "production-secret-key"
```

### HashiCorp Vault Integration

```python
settings = Dynaconf(
    settings_files=["settings.toml"],
    vault_enabled=True,
    vault_url="https://vault.example.com",
    vault_token="hvs.your-token-here",
    vault_path="secret/data/myapp",
)
```

### Environment Variable Secrets

```bash
export DYNACONF_SECRET_KEY="production-key"
```

---

## Flask Integration

```python
from flask import Flask
from dynaconf import FlaskDynaconf

app = Flask(__name__)
FlaskDynaconf(app, settings_files=["settings.toml"])

# Access settings via app.config (case-insensitive)
app.config["DEBUG"]
app.config["DATABASE_URL"]

# Or via the dynaconf settings object
app.config.get("SECRET_KEY")
```

### Flask-Specific Settings File

```toml
# settings.toml
[default]
DEBUG = false
SECRET_KEY = "change-me"

[development]
DEBUG = true

[production]
DEBUG = false
```

Switch environments:

```bash
export FLASK_ENV=development
```

---

## Django Integration

```python
# In settings.py (at the bottom)
import dynaconf  # noqa

settings = dynaconf.DjangoDynaconf(
    __name__,
    settings_files=["settings.toml", ".secrets.toml"],
    environments=True,
)
```

Or using the `dynaconf init -f toml --django` command.

```bash
# Django-specific initialization
cd your_django_project/
dynaconf init -f toml --django
```

This adds the necessary line to `settings.py` and creates the configuration files.

### Django Environment Variables

Django integration uses `DJANGO_` as the default prefix for environment variables:

```bash
export DJANGO_DEBUG=false
export DJANGO_DATABASES__default__ENGINE="django.db.backends.postgresql"
```

---

## Merging Strategies

When settings come from multiple sources, dynaconf needs to decide how to merge nested structures.

### Default Behavior (Override)

By default, a later source completely replaces the value from an earlier source:

```toml
# settings.toml
[default]
servers = ["server1", "server2"]

[production]
servers = ["prod-server1"]
# Result: servers = ["prod-server1"] (replaced, not merged)
```

### Merge Mode

Enable merge mode to deep-merge dictionaries and extend lists:

```python
settings = Dynaconf(
    settings_files=["settings.toml"],
    merge_enabled=True,
)
```

Or per-key using the `dynaconf_merge` marker:

```toml
# settings.toml
[default]
[default.database]
host = "localhost"
port = 5432

[production]
[production.database]
host = "prod-host"
dynaconf_merge = true
# Result: database = {host: "prod-host", port: 5432} (merged)
```

### List Merge with `@merge`

```toml
[default]
allowed_hosts = ["localhost"]

[production]
allowed_hosts = ["@merge", "example.com"]
# Result: ["localhost", "example.com"]
```

---

## Custom Loaders

Create custom loaders to read settings from any source:

```python
# my_loader.py
def load(obj, env=None, silent=True, key=None, filename=None):
    """Load settings from a custom source."""
    data = fetch_from_custom_source()
    obj.update(data)

def clean(obj, settings, env, silent=True):
    """Optional: clean up when settings are reloaded."""
    pass
```

Register the loader:

```python
settings = Dynaconf(
    loaders=["my_loader"],  # Module path
    settings_files=["settings.toml"],
)
```
