# semantic-version

A Python library for **semantic versioning** (SemVer) as defined by [semver.org](https://semver.org/).

`semantic-version` provides classes to parse, compare, and manipulate version strings that follow the Semantic Versioning 2.0.0 specification. It also provides specifier classes (similar to pip/npm version ranges) for matching versions against constraints, and helpers for coercing non-compliant version strings into valid semver.

---

## Latest Stable Version

| Detail | Value |
|--------|-------|
| **Package** | `semantic-version` |
| **Latest version** | 2.10.0 (released August 2022) |
| **Python support** | Python >= 3.7 |
| **License** | BSD 2-Clause |
| **Repository** | https://github.com/rbarrois/python-semanticversion |
| **Documentation** | https://python-semanticversion.readthedocs.io |

The library is mature and stable. The import name is `semantic_version` (underscore, not hyphen).

---

## Installation

```bash
pip install semantic-version
```

`semantic-version` has **zero dependencies**. It is pure Python.

```python
import semantic_version
```

---

## Core API Reference

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

---

## Gotchas and Common Mistakes

### 1. Import Name Uses Underscore

```python
# WRONG
import semantic-version   # SyntaxError

# CORRECT
import semantic_version
```

### 2. `Version` Rejects Partial Strings

```python
# WRONG -- raises ValueError
Version("1.2")

# CORRECT -- use coerce or NativeVersion for partial versions
Version.coerce("1.2")       # Version('1.2.0')
NativeVersion("1.2")        # NativeVersion('1.2')
```

### 3. Build Metadata Is Ignored in Comparisons

Per the SemVer spec, build metadata does not affect version precedence. Two versions that differ only in build metadata are considered equal:

```python
Version("1.0.0+build1") == Version("1.0.0+build2")  # True
```

If you need to distinguish build metadata, compare the `.build` tuples directly.

### 4. `SimpleSpec` vs `Spec` Syntax Differences

`SimpleSpec` and `Spec` have slightly different syntax rules. `Spec` does not support `^` or `~` operators. Always prefer `SimpleSpec` for new code.

### 5. `next_patch()` on Pre-release Clears Pre-release

```python
v = Version("1.0.0-beta.2")
v.next_patch()   # Version('1.0.0'), NOT Version('1.0.1')
# This is correct per SemVer: 1.0.0 is the next release after 1.0.0-beta.2
```

### 6. Immutability

`Version` objects are immutable. Bumping methods return new `Version` instances:

```python
v = Version("1.0.0")
v2 = v.next_minor()
v   # still Version('1.0.0')
v2  # Version('1.1.0')
```

---

## Complete Examples

### Example 1: Dependency Resolution

```python
from semantic_version import Version, SimpleSpec

available = [
    Version("1.0.0"),
    Version("1.1.0"),
    Version("1.2.0-beta.1"),
    Version("1.2.0"),
    Version("2.0.0"),
]

# Find the best version matching a constraint
constraint = SimpleSpec(">=1.0.0,<2.0.0")
best = constraint.select(available)
print(f"Best match: {best}")  # Best match: 1.2.0

# Filter all matching versions
matches = [v for v in available if constraint.match(v)]
print(matches)  # [Version('1.0.0'), Version('1.1.0'), Version('1.2.0-beta.1'), Version('1.2.0')]
```

### Example 2: Release Workflow

```python
from semantic_version import Version

current = Version("1.3.7")

# Decide next version based on change type
def next_version(current: Version, change: str) -> Version:
    if change == "breaking":
        return current.next_major()
    elif change == "feature":
        return current.next_minor()
    elif change == "fix":
        return current.next_patch()
    else:
        raise ValueError(f"Unknown change type: {change}")

print(next_version(current, "fix"))       # 1.3.8
print(next_version(current, "feature"))   # 1.4.0
print(next_version(current, "breaking"))  # 2.0.0
```

### Example 3: Sorting Mixed Versions

```python
from semantic_version import Version

versions = [
    Version("2.0.0"),
    Version("1.0.0-alpha"),
    Version("1.0.0"),
    Version("1.0.0-rc.1"),
    Version("1.0.0-beta"),
    Version("0.9.0"),
]

for v in sorted(versions):
    print(v)
# 0.9.0
# 1.0.0-alpha
# 1.0.0-beta
# 1.0.0-rc.1
# 1.0.0
# 2.0.0
```

### Example 4: Coercing Real-World Version Strings

```python
from semantic_version import Version

raw_versions = ["v2.3", "1.0", "3.1.4.1592", "0.9.1-beta"]

for raw in raw_versions:
    coerced = Version.coerce(raw)
    print(f"{raw!r:20s} -> {coerced}")
# 'v2.3'               -> 2.3.0
# '1.0'                -> 1.0.0
# '3.1.4.1592'         -> 3.1.4
# '0.9.1-beta'         -> 0.9.1-beta
```

---

## See Also

- [SemVer 2.0.0 Specification](https://semver.org/)
- [python-semanticversion on GitHub](https://github.com/rbarrois/python-semanticversion)
- [Full Documentation](https://python-semanticversion.readthedocs.io)
