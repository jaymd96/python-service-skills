---
name: dynaconf
description: Layered configuration management with multi-format and environment support. Use when managing app settings across environments, reading from TOML/YAML/env vars, integrating with Django/Flask settings, or validating configuration. Triggers on dynaconf, configuration management, settings, environment config, TOML settings, twelve-factor config.
---

# dynaconf — Configuration Management (v3.3.0)

## Quick Start

```bash
pip install dynaconf
dynaconf init -f toml  # scaffolds settings.toml + .secrets.toml
```

```python
from dynaconf import Dynaconf

settings = Dynaconf(settings_files=["settings.toml", ".secrets.toml"])
settings.DATABASE_URL  # reads from file or env var
```

## Key Patterns

### Basic setup with environments
```python
from dynaconf import Dynaconf

settings = Dynaconf(
    settings_files=["settings.toml", ".secrets.toml"],
    environments=True,          # enable [development]/[production] sections
    envvar_prefix="MYAPP",      # MYAPP_DATABASE_URL overrides DATABASE_URL
    env_switcher="MYAPP_ENV",   # switch env via MYAPP_ENV=production
    load_dotenv=True,           # load .env file
)
settings.DATABASE_URL           # attribute access (case-insensitive)
settings.get("MISSING", "default")  # with default
```

### Validation
```python
from dynaconf import Dynaconf, Validator

settings = Dynaconf(
    settings_files=["settings.toml"],
    validators=[
        Validator("DATABASE_URL", must_exist=True),
        Validator("DEBUG", is_type_of=bool, default=False),
        Validator("PORT", gte=1024, lte=65535),
    ],
)
```

## References

- **[api.md](references/api.md)** — Core API, constructor parameters, settings files, environment variables, validators, secrets, framework integration, and merging strategies
- **[examples.md](references/examples.md)** — Complete application examples, dynamic reloading, and common mistakes to avoid
