---
name: docker-generator
description: Pure Python Dockerfile generation library. Use when programmatically building Dockerfiles with DockerfileBuilder fluent API, multi-stage builds, 17 directive types, or the sls_dockerfile() preset for SLS distributions. Zero external dependencies. Triggers on docker-generator, DockerfileBuilder, Dockerfile generation, sls_dockerfile, Docker directives.
---

# docker-generator — Dockerfile Generation (v0.1.0)

## Quick Start

```bash
pip install jaymd96-pants-docker-generator
```

```python
from pants_docker_generator import DockerfileBuilder

dockerfile = (
    DockerfileBuilder()
    .from_("python:3.11-slim", alias="runtime")
    .workdir("/app")
    .copy("requirements.txt", ".")
    .run("pip install -r requirements.txt")
    .copy(".", ".")
    .expose(8080)
    .entrypoint(["python", "-m", "app"])
    .build()
)
print(dockerfile.render())
```

## Key Patterns

### SLS preset
```python
from pants_docker_generator import sls_dockerfile

df = sls_dockerfile(product_name="my-service", version="1.0.0",
                     base_image="python:3.11-slim", install_path="/opt/services")
```

### Multi-stage build
```python
builder = (
    DockerfileBuilder()
    .from_("python:3.11", alias="build")
    .workdir("/build")
    .run("pip wheel -w /wheels .")
    .from_("python:3.11-slim", alias="runtime")
    .copy("--from=build", "/wheels", "/wheels")
    .run("pip install /wheels/*")
    .build()
)
```

## References

- **[api.md](references/api.md)** — DockerfileBuilder, 17 directives, Stage, sls_dockerfile preset
- **[examples.md](references/examples.md)** — Multi-stage builds, custom Dockerfiles, gotchas
