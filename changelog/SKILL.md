---
name: changelog
description: Maintain a human-readable changelog following Keep a Changelog and Semantic Versioning. Use when creating or updating CHANGELOG.md, adding release entries, categorising changes (Added, Changed, Deprecated, Removed, Fixed, Security), managing an Unreleased section, cutting a new release version, or integrating changelog updates with SemVer bumps. Triggers on changelog, CHANGELOG.md, keep a changelog, release notes, semver, semantic versioning, unreleased, version bump, yanked release.
---

# Keep a Changelog — Structured Release History (v1.1.0)

## Quick Start

Create `CHANGELOG.md` at the project root:

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Added

- Initial project scaffold

## [0.1.0] - 2025-01-15

### Added

- Core feature implementation
- CLI entry point
- Configuration loading from TOML

[unreleased]: https://github.com/user/project/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/user/project/releases/tag/v0.1.0
```

## Key Patterns

### Change types

Use exactly these six categories — omit any that are empty for a given release:

| Category | What belongs here |
|----------|------------------|
| **Added** | New features, new endpoints, new CLI commands |
| **Changed** | Modifications to existing behaviour or APIs |
| **Deprecated** | Features still present but marked for future removal |
| **Removed** | Features deleted in this release |
| **Fixed** | Bug fixes and corrections |
| **Security** | Vulnerability patches, dependency bumps for CVEs |

### Version header format

```
## [MAJOR.MINOR.PATCH] - YYYY-MM-DD
```

- Versions in square brackets, linked to comparison URL at bottom of file
- Date in ISO 8601 (`YYYY-MM-DD`), no regional formats
- Latest version first (reverse chronological)

### Unreleased workflow

```
1. Day-to-day: add entries under ## [Unreleased]
2. Release time: rename [Unreleased] → [X.Y.Z] - YYYY-MM-DD
3. Add a fresh empty ## [Unreleased] above
4. Update comparison links at bottom of file
5. Commit, tag, push
```

### Yanked releases

Mark withdrawn versions inline — don't delete them:

```markdown
## [1.2.1] - 2025-03-10 [YANKED]
```

### SemVer mapping

| Changelog has... | Version bump |
|-----------------|--------------|
| Only **Fixed** / **Security** | PATCH (x.y.Z) |
| **Added** / **Changed** / **Deprecated** (backwards-compatible) | MINOR (x.Y.0) |
| **Removed** or any breaking **Changed** | MAJOR (X.0.0) |

## References

- **[format.md](references/format.md)** — Complete format specification, section structure, comparison links, anti-patterns
- **[semver.md](references/semver.md)** — Semantic Versioning rules, changelog-to-version mapping, release workflow, pre-release conventions

## Grep Patterns

- `^## \[` — Find version entries in a changelog
- `^## \[Unreleased\]` — Find the unreleased section
- `^### (Added|Changed|Deprecated|Removed|Fixed|Security)` — Find change type headers
- `\[YANKED\]` — Find yanked releases
- `CHANGELOG\.md` — Find references to the changelog file
