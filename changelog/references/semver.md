# Keep a Changelog — Semantic Versioning Integration

> Part of the [changelog](../SKILL.md) skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Semantic Versioning Rules](#semantic-versioning-rules)
- [Mapping Changelog Categories to Version Bumps](#mapping-changelog-categories-to-version-bumps)
- [Release Workflow](#release-workflow)
- [Initial Development (0.x.y)](#initial-development-0xy)
- [Pre-Release Versions](#pre-release-versions)
- [Build Metadata](#build-metadata)
- [Version Precedence](#version-precedence)
- [Release Cycle Examples](#release-cycle-examples)
- [Gotchas](#gotchas)

---

## Semantic Versioning Rules

SemVer 2.0.0 uses a three-part version number: `MAJOR.MINOR.PATCH`

| Component | When to increment | Resets |
|-----------|------------------|--------|
| **MAJOR** | Incompatible API changes (breaking changes) | MINOR and PATCH to 0 |
| **MINOR** | New functionality, backwards-compatible | PATCH to 0 |
| **PATCH** | Backwards-compatible bug fixes | Nothing |

**The public API contract:** Once you declare a public API (typically at 1.0.0), every version number communicates the nature of the change. Consumers can trust that upgrading within a MINOR range won't break their code.

---

## Mapping Changelog Categories to Version Bumps

The changelog categories map directly to SemVer decisions:

### PATCH bump (x.y.Z)

The release contains only backwards-compatible fixes:

```markdown
## [1.2.1] - 2025-07-01

### Fixed

- Handle null values in JSON response parsing
- Correct off-by-one error in pagination

### Security

- Upgrade `urllib3` to 2.1.0 (CVE-2024-XXXXX)
```

**Rule:** Only `Fixed` and/or `Security` entries, and none of them change observable API behaviour.

### MINOR bump (x.Y.0)

The release adds new functionality or deprecates existing functionality, all backwards-compatible:

```markdown
## [1.3.0] - 2025-08-01

### Added

- Bulk import endpoint `POST /api/users/bulk`
- `--dry-run` flag to CLI deploy command

### Changed

- Improve query performance for large datasets (no API change)

### Deprecated

- `Config.from_file()` — use `Config.load()` instead

### Fixed

- Retry logic now respects `Retry-After` header
```

**Rule:** Has `Added`, `Changed`, or `Deprecated` entries — but nothing breaks existing consumers. `Fixed` and `Security` entries can also be present.

### MAJOR bump (X.0.0)

The release contains backwards-incompatible changes:

```markdown
## [2.0.0] - 2025-09-01

### Removed

- `Config.from_file()` (deprecated since 1.3.0)
- Python 3.9 support

### Changed

- `GET /api/users` response shape changed: `items` field renamed to `data`
- Authentication now requires OAuth 2.0 (API key auth removed)

### Added

- GraphQL endpoint at `/graphql`

### Fixed

- Memory leak in long-running connections
```

**Rule:** Has `Removed` entries, or `Changed` entries that break backwards compatibility. The presence of `Added` or `Fixed` doesn't prevent a MAJOR bump — the highest-impact change determines the version.

### Decision flowchart

```
Does this release remove anything or break existing behaviour?
├── Yes → MAJOR bump (X.0.0)
└── No
    ├── Does it add new features, change behaviour (compatibly), or deprecate?
    │   ├── Yes → MINOR bump (x.Y.0)
    │   └── No
    │       └── Only fixes and security patches → PATCH bump (x.y.Z)
    └── (empty release should not be published)
```

---

## Release Workflow

### Step-by-step: cutting a release

**1. Review the Unreleased section**

```markdown
## [Unreleased]

### Added

- Bulk import endpoint

### Fixed

- Retry logic respects Retry-After header
```

**2. Determine the version bump**

This release has `Added` and `Fixed` — the highest impact is `Added` (new feature, backwards-compatible), so this is a MINOR bump. If the current version is `1.2.3`, the new version is `1.3.0`.

**3. Update the changelog**

Replace the `[Unreleased]` heading and add a fresh empty one:

```markdown
## [Unreleased]

## [1.3.0] - 2025-08-15

### Added

- Bulk import endpoint

### Fixed

- Retry logic respects Retry-After header
```

**4. Update comparison links**

Before:
```markdown
[unreleased]: https://github.com/user/project/compare/v1.2.3...HEAD
```

After:
```markdown
[unreleased]: https://github.com/user/project/compare/v1.3.0...HEAD
[1.3.0]: https://github.com/user/project/compare/v1.2.3...v1.3.0
```

**5. Commit, tag, push**

```bash
git add CHANGELOG.md
git commit -m "Release v1.3.0"
git tag v1.3.0
git push origin main --tags
```

---

## Initial Development (0.x.y)

Before your first stable release, use `0.x.y` versions:

```
0.1.0 — First usable version
0.2.0 — Breaking changes are fine (no stability promise yet)
0.2.1 — Bug fix
0.15.0 — Still pre-1.0, still allowed to break things
1.0.0 — Public API declared stable
```

**During 0.x.y:**
- MINOR bumps (0.X.0) may include breaking changes
- PATCH bumps (0.x.Y) are still just bug fixes
- There is no obligation to use MAJOR bumps for breaking changes
- The changelog still documents everything — the version number just doesn't carry the same stability guarantee

**When to go 1.0.0:**
- Your software is used in production
- You have a stable API you're willing to commit to
- Users depend on your software and need upgrade guarantees

---

## Pre-Release Versions

For release candidates, betas, and alphas:

```
2.0.0-alpha.1
2.0.0-beta.1
2.0.0-rc.1
2.0.0
```

**Rules:**
- Pre-release versions have lower precedence than the release: `1.0.0-alpha < 1.0.0`
- Use dot-separated identifiers after a hyphen
- Numeric identifiers are compared as integers; alphanumeric as ASCII

**In the changelog:**

```markdown
## [2.0.0-rc.1] - 2025-08-01

### Changed

- Final API shape for v2 (see migration guide)

## [2.0.0-beta.2] - 2025-07-15

### Fixed

- Auth token refresh race condition in new OAuth flow
```

Pre-release entries go in the changelog like any other version, in reverse chronological order.

---

## Build Metadata

Build metadata is appended with `+` and is ignored for precedence:

```
1.0.0+20250815
1.0.0+build.123
1.0.0+sha.a1b2c3d
```

`1.0.0+build.1` and `1.0.0+build.2` are the **same version** for precedence purposes. Build metadata is typically not included in changelog entries — it's for internal tracking, not user-facing documentation.

---

## Version Precedence

Versions are compared left to right:

```
1.0.0 < 2.0.0 < 2.1.0 < 2.1.1
1.0.0-alpha < 1.0.0-alpha.1 < 1.0.0-beta < 1.0.0-rc.1 < 1.0.0
```

**Comparison rules:**
1. Compare MAJOR, MINOR, PATCH as integers
2. A version with a pre-release tag has lower precedence than the same version without
3. Pre-release fields: numeric fields compared as integers, alphanumeric as ASCII strings
4. Build metadata (`+...`) is ignored entirely for precedence

---

## Release Cycle Examples

### Example: Library with steady feature development

```
0.1.0  — Initial release, core feature set
0.2.0  — Add caching layer (breaking: changed constructor signature)
0.2.1  — Fix cache invalidation bug
0.3.0  — Add async support
1.0.0  — Stable API declared
1.0.1  — Fix edge case in async context manager
1.1.0  — Add batch processing support
1.1.1  — Security: upgrade dependency for CVE fix
1.2.0  — Add metrics export, deprecate old config format
2.0.0  — Remove old config format, require Python 3.11+
```

### Example: Service with careful rollout

```
1.0.0      — Initial production release
1.0.1      — Fix health check timeout
1.1.0      — Add audit logging
1.1.1      — Fix audit log rotation
2.0.0-rc.1 — Preview: new auth system (breaking)
2.0.0-rc.2 — Fix token refresh in new auth
2.0.0      — New auth system released
2.0.1      — Fix migration script for auth upgrade
```

---

## Gotchas

**Breaking change in a "Changed" entry requires MAJOR bump.** A change that alters existing API behaviour (even if the function signature stays the same) is a breaking change if consumers depend on the old behaviour.

**Deprecation is not removal.** `Deprecated` is MINOR (you're adding a warning, not breaking anything). `Removed` is MAJOR (the feature is gone).

**Multiple categories in one release — highest impact wins.** If a release has `Added` (MINOR) and `Fixed` (PATCH), the version bump is MINOR. If it also has `Removed` (MAJOR), the bump is MAJOR.

**Don't retroactively edit released entries.** If you discover an error in a published changelog entry, add a correction in the next release's notes rather than silently editing history. The exception is fixing genuine typos that don't change meaning.

**SemVer applies to your public API, not internal implementation.** Refactoring internals with no observable API change is a PATCH (or no release at all). Adding a new public function is MINOR even if the internal change was trivial.

**The first release can be any version.** Starting at `0.1.0` is conventional for pre-stable, `1.0.0` for stable. Starting at `0.0.1` is also valid but less common. Never start at `0.0.0`.
