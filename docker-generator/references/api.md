# Docker-Generator — Core API Reference

> Part of the docker-generator skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [DockerfileBuilder](#dockerfilebuilder)
- [Dockerfile](#dockerfile)
- [Stage](#stage)
- [Directives](#directives)
- [DockerignoreConfig](#dockerignoreconfig)
- [OCI Labels](#oci-labels)
- [sls_dockerfile Preset](#sls_dockerfile-preset)

## DockerfileBuilder

Fluent builder for constructing Dockerfiles.

```python
from pants_docker_generator import DockerfileBuilder
```

### Builder Methods

| Method | Description |
|--------|-------------|
| `from_(image, alias=None, platform=None)` | Start new FROM stage |
| `run(command)` | RUN instruction |
| `copy(src, dst, from_stage=None, chown=None)` | COPY instruction |
| `add(src, dst, chown=None)` | ADD instruction |
| `workdir(path)` | WORKDIR instruction |
| `user(user, group=None)` | USER instruction |
| `expose(port, protocol=None)` | EXPOSE instruction |
| `entrypoint(*args)` | ENTRYPOINT (exec form) |
| `cmd(*args)` | CMD (exec form) |
| `arg(name, default=None)` | ARG instruction |
| `env(key, value)` | ENV instruction |
| `label(labels: dict[str, str])` | LABEL instruction |
| `volume(*paths)` | VOLUME instruction |
| `shell(*args)` | SHELL instruction |
| `healthcheck(command, interval_seconds=10, timeout_seconds=5, start_period_seconds=30, retries=3)` | HEALTHCHECK |
| `healthcheck_none()` | HEALTHCHECK NONE (disable inherited) |
| `comment(text)` | Add comment line |
| `blank()` | Add blank line |
| `directive(d: Directive)` | Add pre-built directive |
| `build() -> Dockerfile` | Finalize and return Dockerfile |

All methods return `self` for chaining (except `build()`).

## Dockerfile

Immutable representation of a complete Dockerfile. Frozen dataclass.

```python
from pants_docker_generator import Dockerfile

# Fields
dockerfile.stages: tuple[Stage, ...]

# Methods
dockerfile.render() -> str  # Render full Dockerfile as string
```

## Stage

Single stage in a multi-stage Dockerfile. Frozen dataclass.

```python
from pants_docker_generator import Stage

# Fields
stage.from_directive: From
stage.directives: tuple[Directive, ...] = ()

# Properties
stage.alias: str | None  # Stage alias if any

# Methods
stage.render() -> str
```

## Directives

All directives are frozen dataclasses with a `render() -> str` method.

```python
from pants_docker_generator import (
    Add, Arg, BlankLine, Cmd, Comment, Copy, Entrypoint,
    Env, Expose, From, HealthCheck, HealthCheckNone,
    Label, Run, Shell, User, Volume, Workdir,
)
```

| Directive | Fields | Renders As |
|-----------|--------|------------|
| `From` | `image, alias=None, platform=None` | `FROM [--platform=...] image [AS alias]` |
| `Run` | `command` | `RUN command` |
| `Copy` | `src, dst, from_stage=None, chown=None` | `COPY [--from=...] [--chown=...] src dst` |
| `Add` | `src, dst, chown=None` | `ADD [--chown=...] src dst` |
| `Workdir` | `path` | `WORKDIR path` |
| `User` | `user, group=None` | `USER user[:group]` |
| `Expose` | `port: int, protocol=None` | `EXPOSE port[/protocol]` |
| `Entrypoint` | `args: tuple[str, ...]` | `ENTRYPOINT ["arg1", "arg2"]` |
| `Cmd` | `args: tuple[str, ...]` | `CMD ["arg1", "arg2"]` |
| `Arg` | `name, default=None` | `ARG name[=default]` |
| `Env` | `key, value` | `ENV key=value` |
| `Label` | `labels: dict[str, str]` | `LABEL key="value" ...` (multiline for multiple) |
| `Volume` | `paths: tuple[str, ...]` | `VOLUME ["path1", "path2"]` |
| `Shell` | `args: tuple[str, ...]` | `SHELL ["arg1", "arg2"]` |
| `HealthCheck` | `command, interval_seconds=10, timeout_seconds=5, start_period_seconds=30, retries=3` | `HEALTHCHECK --interval=... CMD command` |
| `HealthCheckNone` | (none) | `HEALTHCHECK NONE` |
| `Comment` | `text` | `# text` |
| `BlankLine` | (none) | (empty line) |

## DockerignoreConfig

Configuration for `.dockerignore` generation. Frozen dataclass.

```python
from pants_docker_generator import DockerignoreConfig

config = DockerignoreConfig(
    ignore_all=True,          # Start with * (deny all)
    allow_patterns=("app/",), # Patterns to allow
    deny_patterns=(),         # Additional deny patterns
    comment=None,             # Header comment
)
config.render() -> str
```

### generate_dockerignore

```python
from pants_docker_generator import generate_dockerignore

content = generate_dockerignore(
    allow=("app/", "requirements.txt"),
    comment="Auto-generated",
)
```

## OCI Labels

Helper for OCI annotation labels:

```python
from pants_docker_generator import oci_labels

label = oci_labels(
    title="My Service",
    version="1.0.0",
    vendor="Example Corp",
    description="Service description",
    url="https://example.com",
    source="https://github.com/example/repo",
    licenses="MIT",
    revision="abc123",
    created="2024-01-01T00:00:00Z",
    extra={"custom.key": "value"},
)
# Returns a Label directive with org.opencontainers.image.* keys
```

## sls_dockerfile Preset

Generate a complete Dockerfile for SLS distributions:

```python
from pants_docker_generator import sls_dockerfile, sls_dockerignore

df = sls_dockerfile(
    base_image="python:3.11-slim",
    product_name="my-service",
    product_version="1.0.0",
    product_group="com.example",
    dist_name="my-service-1.0.0",
    tarball_name="my-service-1.0.0.sls.tgz",
    install_path="/opt/services",            # default
    product_type="helm.v1",                  # default
    health_check_interval=None,              # optional
    health_check_timeout=None,               # optional
    health_check_start_period=None,          # optional
    health_check_retries=None,               # optional
    use_hook_init=False,                     # enable hook entrypoint
    expose_ports=(8080,),                    # ports to expose
    labels={"team": "platform"},             # additional labels
)
print(df.render())

# SLS-specific .dockerignore
ignore = sls_dockerignore()
```
