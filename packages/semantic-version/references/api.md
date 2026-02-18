# semantic-version — API Reference

> Part of the semantic-version skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [`Version` -- Strict SemVer](#version----strict-semver)
  - [Version Bumping](#version-bumping)
  - [Comparison Operators](#comparison-operators)
- [`NativeVersion` -- Partial / Loose Versions](#nativeversion----partial--loose-versions)
- [`Version.coerce()` -- Coercion from Non-SemVer Strings](#versioncoerce----coercion-from-non-semver-strings)
- [`SimpleSpec` -- NPM-style Version Specifiers](#simplespec----npm-style-version-specifiers)
  - [SimpleSpec Operators](#simplespec-operators)
- [`Spec` (Legacy)](#spec-legacy)
- [Working with Pre-release and Build Metadata](#working-with-pre-release-and-build-metadata)

### `Version` -- Strict SemVer

Represents a version that strictly conforms to SemVer 2.0.0: `MAJOR.MINOR.PATCH[-prerelease][+build]`.

```python
from semantic_version import Version

v = Version("1.4.2")
v.major   # 1
v.minor   # 4
v.patch   # 2

# Pre-release and build metadata
v2 = Version("2.0.0-rc.1+build.42")
v2.prerelease   # ('rc', '1')
v2.build        # ('build', '42')
```

#### Version Bumping

```python
v = Version("1.2.3")

v.next_major()  # Version('2.0.0')
v.next_minor()  # Version('1.3.0')
v.next_patch()  # Version('1.2.4')

# Pre-release versions
v2 = Version("1.0.0-alpha.1")
v2.next_patch()  # Version('1.0.0')  -- clears pre-release, since 1.0.0 > 1.0.0-alpha.1
```

#### Comparison Operators

Versions support all rich comparison operators, following SemVer 2.0.0 precedence rules:

```python
Version("1.2.3") < Version("1.3.0")        # True
Version("1.0.0-alpha") < Version("1.0.0")   # True (pre-release < release)
Version("1.0.0-alpha") < Version("1.0.0-beta")  # True (alphabetic comparison)

# Build metadata is IGNORED in comparisons (per SemVer spec)
Version("1.0.0+build1") == Version("1.0.0+build2")  # True
```

### `NativeVersion` -- Partial / Loose Versions

`NativeVersion` allows partial version strings (e.g., `"1.2"` or `"1"`) and does not enforce the full SemVer format.

```python
from semantic_version import NativeVersion

v = NativeVersion("1.2")
v.major   # 1
v.minor   # 2
v.patch   # None  -- not specified

# Partial versions can still be compared
NativeVersion("1.2") < NativeVersion("1.3")  # True
```

### `Version.coerce()` -- Coercion from Non-SemVer Strings

Convert loose or non-compliant version strings into strict `Version` objects:

```python
Version.coerce("1")         # Version('1.0.0')
Version.coerce("1.2")       # Version('1.2.0')
Version.coerce("1.2.3.4")   # Version('1.2.3')  -- extra components dropped
Version.coerce("v2.0.1")    # Version('2.0.1')  -- leading 'v' stripped
```

### `SimpleSpec` -- NPM-style Version Specifiers

`SimpleSpec` implements a version range syntax inspired by npm/node-semver:

```python
from semantic_version import SimpleSpec, Version

spec = SimpleSpec(">=1.0.0,<2.0.0")
spec.match(Version("1.5.0"))   # True
spec.match(Version("2.0.0"))   # False

# Supports various range operators
SimpleSpec("~=1.4.0")           # >=1.4.0, <1.5.0 (compatible release)
SimpleSpec("^1.2.3")            # >=1.2.3, <2.0.0 (caret / next major)
SimpleSpec(">=1.0,<2.0 || >=3.0")  # Union of ranges
SimpleSpec("*")                 # Matches any version
SimpleSpec("1.x.x")             # >=1.0.0, <2.0.0

# Use `select()` to find the best matching version from a list
versions = [Version("1.0.0"), Version("1.5.3"), Version("2.1.0")]
spec = SimpleSpec(">=1.0.0,<2.0.0")
spec.select(versions)  # Version('1.5.3')  -- highest match
```

#### SimpleSpec Operators

| Operator | Example | Meaning |
|----------|---------|---------|
| `>=`, `<=`, `>`, `<`, `==`, `!=` | `>=1.2.0` | Standard comparisons |
| `^` (caret) | `^1.2.3` | `>=1.2.3, <2.0.0` |
| `~` (tilde) | `~1.2.3` | `>=1.2.3, <1.3.0` |
| `~=` (compatible) | `~=1.4.0` | `>=1.4.0, <1.5.0` |
| `*`, `x` | `1.x.x` | Wildcard |
| `,` | `>=1.0,<2.0` | AND (intersection) |
| `\|\|` | `>=1.0 \|\| >=3.0` | OR (union) |

### `Spec` (Legacy)

`Spec` is the older specifier class. Prefer `SimpleSpec` for new code. `Spec` uses a slightly different syntax and does not support caret or tilde ranges.

```python
from semantic_version import Spec, Version

spec = Spec(">=1.0.0,<2.0.0")
spec.match(Version("1.5.0"))  # True
spec.select([Version("1.0.0"), Version("1.9.0")])  # Version('1.9.0')
```

---

## Working with Pre-release and Build Metadata

```python
v = Version("1.0.0-alpha.1+linux.x86")

v.prerelease  # ('alpha', '1')
v.build       # ('linux', 'x86')

# Pre-release ordering follows SemVer rules:
# numeric identifiers are compared as integers, string identifiers lexicographically
Version("1.0.0-alpha") < Version("1.0.0-alpha.1")   # True
Version("1.0.0-alpha.1") < Version("1.0.0-alpha.2") # True
Version("1.0.0-1") < Version("1.0.0-2")             # True (numeric comparison)

# Constructing versions programmatically
v = Version(major=2, minor=0, patch=0, prerelease=('rc', '1'), build=('build', '99'))
str(v)  # '2.0.0-rc.1+build.99'
```
