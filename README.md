# python-service-skills

A Claude Code plugin providing 48 Python ecosystem skills for service development — covering the Witchcraft stack, data libraries, HTTP, observability, CLI tooling, resilience, and more.

## Usage

Install as a Claude Code plugin:

```bash
claude plugin add jaymd96/python-service-skills
```

Or clone and point Claude Code at the directory:

```bash
git clone https://github.com/jaymd96/python-service-skills.git
claude --plugin-dir ./python-service-skills
```

Each skill provides a `SKILL.md` with quick-start patterns and a `references/` directory with detailed API docs and examples. Claude Code loads the relevant skill when your prompt matches its trigger keywords.

## Skills

### Witchcraft Ecosystem

| Skill | Description |
|-------|-------------|
| [enchant](./enchant) | Type-safe API contract code generation from `.enchant.yml` DSL |
| [cherry](./cherry) | Opinionated Python server framework with structured logging and health checks |
| [rune](./rune) | Composable validation and serialization built on attrs/cattrs |
| [transmute](./transmute) | Zero-downtime online schema migration with 8-state lifecycle |
| [witchcraft-service](./witchcraft-service) | End-to-end workflow for building services with the Witchcraft stack |

### Data Modeling & Serialization

| Skill | Description |
|-------|-------------|
| [attrs](./attrs) | Python classes without boilerplate |
| [cattrs](./cattrs) | Composable converters for structuring/unstructuring data |
| [orjson](./orjson) | Fast JSON serialization with native datetime/numpy support |
| [pyyaml](./pyyaml) | YAML parser and emitter |
| [deepdiff](./deepdiff) | Deep comparison and diffing of Python objects |

### HTTP & Networking

| Skill | Description |
|-------|-------------|
| [httpx](./httpx) | Modern HTTP client with sync and async support |

### Observability

| Skill | Description |
|-------|-------------|
| [opentelemetry](./opentelemetry) | Distributed tracing, metrics, and observability |
| [structlog](./structlog) | Structured logging with processor pipelines |

### CLI

| Skill | Description |
|-------|-------------|
| [click](./click) | Composable command-line interface framework |
| [typer](./typer) | CLI builder using Python type hints, built on Click |
| [rich](./rich) | Rich text and beautiful terminal formatting |

### Datetime & Scheduling

| Skill | Description |
|-------|-------------|
| [pendulum](./pendulum) | Timezone-aware datetime with fluent API |
| [python-dateutil](./python-dateutil) | Flexible date parsing and relative deltas |
| [croniter](./croniter) | Cron expression parser and datetime iterator |
| [datetimerange](./datetimerange) | Datetime range operations (intersection, union, containment) |

### Architecture & Patterns

| Skill | Description |
|-------|-------------|
| [blinker](./blinker) | Signal/event dispatching (observer pattern) |
| [pluggy](./pluggy) | Hook-based plugin system |
| [punq](./punq) | Minimal dependency injection container |
| [transitions](./transitions) | Lightweight finite state machine library |
| [returns](./returns) | Typed error handling with Result, Maybe, and IO containers |
| [toolz](./toolz) | Functional programming primitives (pipe, curry, compose) |
| [immutables](./immutables) | High-performance immutable mappings (HAMT) |

### Resilience & Configuration

| Skill | Description |
|-------|-------------|
| [scientist](./scientist) | Safe refactoring through controlled experiments with py-scientist |
| [feature-toggle](./feature-toggle) | Simple feature flags via Cherry's Refreshable config with Scientist gates |
| [tenacity](./tenacity) | Composable retry logic with backoff |
| [dynaconf](./dynaconf) | Layered configuration management |

### Database

| Skill | Description |
|-------|-------------|
| [sqlalchemy](./sqlalchemy) | SQLAlchemy 2.0 ORM and SQL toolkit |

### Security

| Skill | Description |
|-------|-------------|
| [casbin](./casbin) | Policy-based authorization (ACL, RBAC, ABAC) |

### Parsing & Templating

| Skill | Description |
|-------|-------------|
| [lark](./lark) | Modern parsing library for grammars and DSLs |
| [jinja2](./jinja2) | Template engine with inheritance and macros |
| [rule-engine](./rule-engine) | Safe business rule evaluation |

### Data Structures & Utilities

| Skill | Description |
|-------|-------------|
| [networkx](./networkx) | Graph and network analysis |
| [semantic-version](./semantic-version) | SemVer parsing, comparison, and range matching |
| [burr](./burr) | State machine framework for AI agent workflows |

### Workflow & Conventions

| Skill | Description |
|-------|-------------|
| [changelog](./changelog) | Maintain a changelog following Keep a Changelog and Semantic Versioning |

### Platform & Deployment

| Skill | Description |
|-------|-------------|
| [apollo-clone](./apollo-clone) | Continuous delivery platform on k3s |
| [citadel](./citadel) | Lightweight K3s platform with zero-trust networking |
| [platform-deploy](./platform-deploy) | Deploying services on Apollo/Citadel |
| [forge-pipeline](./forge-pipeline) | Packaging, containerizing, and publishing services |
| [sls-distribution](./sls-distribution) | Pants plugin for SLS distribution packaging |
| [docker-generator](./docker-generator) | Programmatic Dockerfile generation |
| [release-hub](./release-hub) | Apollo Hub publishing client |
| [service-launcher](./service-launcher) | Go binary for running Python services in containers |

## Structure

```
python-service-skills/
├── .claude-plugin/
│   └── marketplace.json    # Plugin manifest (48 skills)
├── attrs/
│   ├── SKILL.md            # Quick-start patterns and trigger keywords
│   └── references/
│       ├── api.md          # Detailed API reference
│       └── examples.md     # Complete examples
├── httpx/
│   ├── SKILL.md
│   └── references/
│       └── ...
└── ... (48 skill directories)
```

## License

MIT
