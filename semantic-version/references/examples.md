# semantic-version — Examples & Gotchas

> Part of the semantic-version skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Import Name Uses Underscore](#1-import-name-uses-underscore)
  - [2. `Version` Rejects Partial Strings](#2-version-rejects-partial-strings)
  - [3. Build Metadata Is Ignored in Comparisons](#3-build-metadata-is-ignored-in-comparisons)
  - [4. `SimpleSpec` vs `Spec` Syntax Differences](#4-simplespec-vs-spec-syntax-differences)
  - [5. `next_patch()` on Pre-release Clears Pre-release](#5-next_patch-on-pre-release-clears-pre-release)
  - [6. Immutability](#6-immutability)
- [Complete Examples](#complete-examples)
  - [Example 1: Dependency Resolution](#example-1-dependency-resolution)
  - [Example 2: Release Workflow](#example-2-release-workflow)
  - [Example 3: Sorting Mixed Versions](#example-3-sorting-mixed-versions)
  - [Example 4: Coercing Real-World Version Strings](#example-4-coercing-real-world-version-strings)
- [See Also](#see-also)

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
