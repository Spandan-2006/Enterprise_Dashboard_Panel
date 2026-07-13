# 15 — Git Workflow

## 1. Branching Strategy

| Branch | Purpose |
|---|---|
| `main` | Always deployable; protected branch (requires PR + CI pass) |
| `develop` | Integration branch; requires CI pass to push |
| `feature/issue-number-short-description` | New features — e.g., `feature/42-add-rollback-endpoint` |
| `bugfix/issue-number-short-description` | Bug fixes — e.g., `bugfix/57-fix-jwt-expiry-validation` |
| `hotfix/issue-number-short-description` | Production hotfixes; branch from `main`, merge to both `main` and `develop` |
| `release/v1.0.0` | Release preparation and final QA before tagging |

### Workflow Summary

```
main ──────────────────────────────────────────────── (tagged releases)
  └── develop ──────────────────────────────────────── (integration)
        ├── feature/42-add-rollback-endpoint
        ├── bugfix/57-fix-jwt-expiry-validation
        └── release/v1.0.0 ─────────────────────────── (PR to main)

hotfix/63-patch-auth-bypass (branches from main, merges to main AND develop)
```

---

## 2. Branch Protection Rules

| Branch | Rules |
|---|---|
| `main` | Require 1 PR review; require all status checks pass (CI, tests, coverage); no direct push; require linear history |
| `develop` | Require CI status checks pass; no direct push |
| `feature/*, bugfix/*` | No protection; merged via PR to `develop` |

**Configuring protection rules (GitHub):**

1. Navigate to **Settings → Branches → Add rule**.
2. For `main`: enable *Require a pull request before merging*, *Require status checks to pass* (select `build`, `test`, `coverage`), *Require linear history*, and *Restrict who can push directly*.
3. For `develop`: enable *Require status checks to pass* (select `build`, `test`) and *Restrict who can push directly*.

---

## 3. Commit Message Convention (Conventional Commits)

**Format:** `type(scope): description`

- **`type`**: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `ci`, `perf`
- **`scope`** (optional): `auth`, `deployment`, `rollback`, `audit`, `user`, `dashboard`, `security`, `db`
- **`description`**: imperative mood, lowercase, maximum 72 characters, no trailing period

### Examples

| Type | Example Commit Message |
|---|---|
| `feat` | `feat(deployment): add rollback endpoint for failed deployments` |
| `fix` | `fix(auth): resolve JWT expiry comparison using UTC timestamps` |
| `docs` | `docs(api): update deployment status endpoint to PATCH method` |
| `test` | `test(service): add DeploymentService unit tests for status transition` |
| `refactor` | `refactor(audit): extract AuditActions constants class` |
| `chore` | `chore: update Spring Boot to 3.3.2` |
| `ci` | `ci: add JaCoCo coverage gate to GitHub Actions workflow` |
| `perf` | `perf(db): add composite index on environment and status columns` |

### Extended Commit Body (optional)

For commits that require more context, add a blank line after the subject and a longer body:

```
feat(deployment): add rollback endpoint for failed deployments

Introduces POST /api/deployments/{id}/rollback which creates a Rollback
entity linked to the given deployment. Only deployments in FAILED status
may be rolled back; other statuses result in 422 Unprocessable Entity.

Closes #42
```

---

## 4. Pull Request Template

The file `.github/pull_request_template.md` in the repository root pre-fills new PRs with this template:

```markdown
## Summary

<!-- What does this PR do? Why? (2-3 sentences) -->

## Type of Change

- [ ] New feature
- [ ] Bug fix
- [ ] Refactoring (no behavior change)
- [ ] Documentation
- [ ] CI/CD change

## Testing Done

- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manually tested locally

## Checklist

- [ ] Tests pass (`./gradlew test`)
- [ ] Coverage gate passes (`./gradlew jacocoTestCoverageVerification`)
- [ ] Docs updated if new endpoint added (`docs/06-api-spec.md`)
- [ ] Database migration added if schema changed (`docs/07-database.md`)
- [ ] No hardcoded secrets or URLs
- [ ] No business logic in controllers
- [ ] PR is against the correct target branch (`develop`, not `main`)
```

---

## 5. Release Tagging

### Versioning Scheme

Semantic versioning: `vMAJOR.MINOR.PATCH`

| Segment | Increment when |
|---|---|
| MAJOR | Breaking API or data model change |
| MINOR | New backward-compatible feature |
| PATCH | Bug fix, patch, or documentation only |

### Tagging Command

```bash
git tag -a v1.0.0 -m "Release 1.0.0: initial public release"
git push origin v1.0.0
```

### CHANGELOG.md Format

```markdown
## [1.0.0] - 2026-07-13

### Added
- Initial deployment management API
- JWT authentication and refresh token flow
- Role-based access control (ADMIN, MANAGER, DEVELOPER, VIEWER)
- Rollback endpoint for failed deployments
- Audit trail with full event history

### Changed
- ...

### Fixed
- ...

### Removed
- ...
```

### Release Process

1. All feature/bugfix branches are merged to `develop` via PR.
2. When ready to release, create `release/v1.0.0` from `develop`.
3. Finalize version numbers, update `CHANGELOG.md`, run full test suite.
4. Open PR from `release/v1.0.0` → `main`.
5. After PR is approved and merged, tag `main`:
   ```bash
   git tag -a v1.0.0 -m "Release 1.0.0: initial public release"
   git push origin v1.0.0
   ```
6. GitHub Actions `release.yml` workflow triggers on the tag push and builds the release artifact.
7. Merge `main` back into `develop` (or merge the release branch into `develop`) to bring version bump and changelog back.

---

## 6. Code Review Checklist (for Reviewers)

When reviewing a PR, verify all of the following before approving:

- [ ] No business logic in controllers — controllers delegate to service layer only
- [ ] All new endpoints secured with the appropriate `@PreAuthorize` annotation
- [ ] Database migration present if any schema change was introduced (see `docs/07-database.md`)
- [ ] Tests added for new code — both unit tests and integration tests
- [ ] No hardcoded secrets, URLs, or passwords in source code or configuration files
- [ ] API spec (`docs/06-api-spec.md`) updated for new or changed endpoints
- [ ] Error handling follows the domain exception pattern (`ResourceNotFoundException`, `InvalidStatusTransitionException`, etc. — see `docs/17-error-handling.md`)
- [ ] Commit messages follow Conventional Commits format (see Section 3 above)
- [ ] PR targets `develop`, not `main` (unless it is a release or hotfix PR)
- [ ] No `.env` file or secrets file accidentally included in the diff
