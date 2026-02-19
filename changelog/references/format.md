# Keep a Changelog — Format Specification

> Part of the [changelog](../SKILL.md) skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [File Naming and Location](#file-naming-and-location)
- [Document Header](#document-header)
- [Version Entries](#version-entries)
- [Change Type Sections](#change-type-sections)
- [The Unreleased Section](#the-unreleased-section)
- [Comparison Links](#comparison-links)
- [Yanked Releases](#yanked-releases)
- [Formatting Rules](#formatting-rules)
- [Anti-Patterns](#anti-patterns)
- [Complete Example](#complete-example)

---

## File Naming and Location

The file must be named `CHANGELOG.md` and placed at the project root. This is a widely recognised convention — tools, CI systems, and humans all look for this specific name.

Do not use alternatives like `HISTORY.md`, `NEWS.md`, `RELEASES.md`, or `changes.txt`. A single, consistent name across all projects maximises discoverability.

---

## Document Header

Every changelog starts with a title and a brief preamble declaring the format:

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).
```

This preamble serves two purposes:
1. Tells readers the conventions in use
2. Links to the full spec for anyone unfamiliar with the format

---

## Version Entries

Each release gets a second-level heading:

```markdown
## [1.2.0] - 2025-06-15
```

**Rules:**
- Version number in square brackets: `[X.Y.Z]`
- Followed by a space, hyphen, space: ` - `
- Date in ISO 8601 format: `YYYY-MM-DD`
- Latest version appears first (reverse chronological order)
- The version in brackets becomes a link via comparison URLs at the bottom of the file

**What counts as a version:**
- Every tagged release should have an entry
- Pre-releases (`1.0.0-alpha.1`) get entries if publicly distributed
- Internal builds or CI-only versions can be omitted

---

## Change Type Sections

Within each version entry, changes are grouped under third-level headings. There are exactly six categories:

### Added

New functionality that did not exist before. New API endpoints, CLI commands, configuration options, integrations.

```markdown
### Added

- User authentication via OAuth 2.0
- `--verbose` flag to CLI for debug output
- Support for PostgreSQL as a storage backend
```

### Changed

Modifications to existing behaviour. API signature changes, default value changes, performance improvements that alter observable behaviour.

```markdown
### Changed

- Default timeout increased from 30s to 60s
- `GET /users` now returns paginated results (default page size 20)
- Migrated from `requests` to `httpx` for HTTP client
```

### Deprecated

Features that still work but are marked for removal in a future version. Always include what to use instead.

```markdown
### Deprecated

- `Config.from_file()` — use `Config.load()` instead, will be removed in 3.0
- Python 3.9 support — minimum will be 3.11 in next major release
```

### Removed

Features or capabilities that have been deleted. If these were previously deprecated, reference that.

```markdown
### Removed

- `Config.from_file()` (deprecated in 2.1, use `Config.load()`)
- Python 3.8 support
- Legacy XML export format
```

### Fixed

Bug fixes. Reference issue numbers where applicable.

```markdown
### Fixed

- Connection pool exhaustion under high concurrency (#142)
- Incorrect timezone handling for dates before 1970
- CLI crash when config file is empty
```

### Security

Vulnerability patches, dependency updates for CVEs, security hardening. Reference CVE identifiers where applicable.

```markdown
### Security

- Upgrade `cryptography` to 42.0.0 (CVE-2024-12345)
- Sanitize user input in template rendering to prevent XSS
- Enforce TLS 1.2 minimum for all outbound connections
```

**Important:** Only include sections that have entries. Empty sections add noise. If a release only has fixes, only include `### Fixed`.

---

## The Unreleased Section

The `[Unreleased]` section sits at the top, above all versioned entries:

```markdown
## [Unreleased]

### Added

- New endpoint for bulk user import

### Fixed

- Memory leak in connection pool recycling
```

**Purpose:**
- Captures changes as they're merged, before a release is cut
- Gives users a preview of what's coming
- Makes release preparation trivial — rename and date-stamp

**Workflow for cutting a release:**

1. Determine the version number based on the changes (see [semver.md](./semver.md))
2. Replace `## [Unreleased]` heading with `## [X.Y.Z] - YYYY-MM-DD`
3. Add a fresh `## [Unreleased]` section above it (empty)
4. Update comparison links at the bottom of the file
5. Commit the changelog update as part of the release commit

---

## Comparison Links

At the bottom of the changelog, add reference-style links that make each version header clickable:

```markdown
[unreleased]: https://github.com/user/project/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/user/project/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/user/project/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/user/project/releases/tag/v1.0.0
```

**Rules:**
- `[unreleased]` always compares the latest tag to `HEAD`
- Each version compares to its predecessor
- The first version links to the tag itself (no comparison)
- Use your repository's URL format (GitHub, GitLab, Bitbucket all support comparison URLs)

**GitLab format:**
```
[unreleased]: https://gitlab.com/user/project/-/compare/v1.2.0...main
[1.2.0]: https://gitlab.com/user/project/-/compare/v1.1.0...v1.2.0
```

---

## Yanked Releases

When a release must be withdrawn (critical bug, security issue, accidental publish):

```markdown
## [1.2.1] - 2025-03-10 [YANKED]

### Fixed

- Attempted fix for connection pool issue (introduced data corruption)
```

**Rules:**
- Append `[YANKED]` to the version header
- Do not delete the entry — the history should remain visible
- Explain why it was yanked in the content or in the next release's notes
- Yanked versions should not be installed by default by package managers

---

## Formatting Rules

### Entry format

Each change is a bullet point starting with `- `:

```markdown
- Description of the change
```

For multi-line entries, indent continuation lines:

```markdown
- Long description that needs more context
  to fully explain the change
```

### Writing good entries

| Do | Don't |
|----|-------|
| Write for humans, not machines | Paste commit messages verbatim |
| Describe the user-visible effect | Describe implementation details |
| Use present tense ("Add", "Fix") | Use past tense ("Added", "Fixed") in entries |
| Reference issue/PR numbers | Leave readers guessing about context |
| Group related changes into one entry | Create separate entries for every commit |
| Be specific: "Fix crash when config file is empty" | Be vague: "Fix bug" |

### Ordering within sections

- Most important changes first within each category
- No strict rule, but consistency helps — some teams prefer alphabetical

---

## Anti-Patterns

### Don't dump the git log

```markdown
# BAD — this is not a changelog
- fix typo in readme
- Merge branch 'feature/auth' into main
- wip
- more wip
- actually fix the thing
- bump version
```

The commit log documents code evolution. The changelog documents user-facing changes. These are different audiences with different needs.

### Don't ignore deprecations

If you're removing something in a future version, deprecate it first and document it. Users need time to migrate. Surprising removals erode trust.

### Don't use regional date formats

```markdown
# BAD — ambiguous
## [1.0.0] - 03/04/2025   # Is this March 4 or April 3?

# GOOD — unambiguous
## [1.0.0] - 2025-03-04
```

### Don't keep empty sections

```markdown
# BAD — noise
### Added

### Changed

### Fixed

- The one actual change

# GOOD — clean
### Fixed

- The one actual change
```

### Don't use GitHub Releases as your only changelog

GitHub Releases are useful but platform-specific. A `CHANGELOG.md` file travels with the code, works offline, and is portable across forges.

---

## Complete Example

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Added

- Bulk import endpoint for user data

## [1.1.0] - 2025-06-15

### Added

- OAuth 2.0 authentication support
- `--verbose` flag to CLI

### Changed

- Default request timeout increased from 30s to 60s

### Deprecated

- `Config.from_file()` — use `Config.load()` instead

### Fixed

- Connection pool exhaustion under high concurrency (#142)

## [1.0.1] - 2025-05-01

### Fixed

- CLI crash when config file is empty
- Incorrect timezone handling for pre-1970 dates

### Security

- Upgrade `cryptography` to 42.0.0 (CVE-2024-12345)

## [1.0.0] - 2025-04-01

### Added

- Core API with CRUD endpoints
- CLI with `run`, `config`, and `status` commands
- PostgreSQL and SQLite storage backends
- Structured logging with structlog
- OpenTelemetry tracing integration
- Docker and Helm chart for deployment

[unreleased]: https://github.com/user/project/compare/v1.1.0...HEAD
[1.1.0]: https://github.com/user/project/compare/v1.0.1...v1.1.0
[1.0.1]: https://github.com/user/project/compare/v1.0.0...v1.0.1
[1.0.0]: https://github.com/user/project/releases/tag/v1.0.0
```
