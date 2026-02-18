---
name: semantic-version
description: Semantic versioning (SemVer 2.0.0) parsing, comparison, and range matching. Use when parsing version strings, comparing versions, matching version constraints/ranges, or bumping version numbers. Triggers on semver, semantic version, version comparison, version range, version bump, version constraint.
---

# semantic-version — SemVer Library (v2.10.0)

## Quick Start

```bash
pip install semantic-version
```

```python
from semantic_version import Version, NpmSpec

v = Version("1.4.2")
v.next_minor()  # Version('1.5.0')
NpmSpec(">=1.0.0 <2.0.0").match(v)  # True
```

## Key Patterns

### Version parsing and bumping
```python
from semantic_version import Version

v = Version("1.2.3")
v.major, v.minor, v.patch  # 1, 2, 3
v.next_major()  # Version('2.0.0')
v.next_minor()  # Version('1.3.0')
v.next_patch()  # Version('1.2.4')
```

### Version ranges
```python
from semantic_version import SimpleSpec

spec = SimpleSpec(">=1.0.0,<2.0.0")
spec.match(Version("1.5.0"))  # True
spec.select([Version("1.0.0"), Version("1.5.0"), Version("2.0.0")])  # Version('1.5.0')
```

### Comparison (follows SemVer precedence)
```python
Version("1.0.0-alpha") < Version("1.0.0")          # True (pre-release < release)
Version("1.0.0+build1") == Version("1.0.0+build2")  # True (build metadata ignored)
```

## References

- **[api.md](references/api.md)** — Full API for Version, NativeVersion, Version.coerce(), SimpleSpec, Spec, and pre-release/build metadata handling
- **[examples.md](references/examples.md)** — Gotchas (underscore import, partial strings, immutability) and complete examples (dependency resolution, release workflow, sorting, coercion)
