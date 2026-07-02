# ADR-007 — Deployment Issues Found During Initial Pipeline Setup

## Status

Resolved

## Date

2026

## Context

During the first end-to-end pipeline run following the implementation of
stages 5–9 (Docker build, registry push, and Render deploy), three
distinct problems were encountered that blocked successful execution.
This record documents each problem, its root cause, and the fix applied.

---

## Decision

Apply three targeted fixes: cross-platform build flag for local image
pushes, workflow-level `permissions` block for `GITHUB_TOKEN`, and a
non-root user in the backend Dockerfile final stage.

---

## Rationale

### (a) Images built for linux/arm64 instead of linux/amd64

**Problem:** Initial images pushed manually to ghcr.io from a MacBook
with Apple Silicon (M-series) were built for `linux/arm64`. Render's
infrastructure runs on `linux/amd64`. The containers failed to start on
Render with an architecture mismatch error.

**Root cause:** `docker build` and `docker buildx build` without
`--platform` default to the host machine's architecture. On Apple
Silicon, that is `arm64`.

**Fix:** All local builds intended for deployment must use
`--platform linux/amd64` explicitly:
```bash
docker buildx build --platform linux/amd64 --push ...
```
The CI pipeline is unaffected — GitHub Actions `ubuntu-latest` runners
are `amd64` by default, so `docker/build-push-action` produces the
correct architecture without any additional configuration.

---

### (b) GITHUB_TOKEN denied write access to ghcr.io

**Problem:** The `docker-build-push` pipeline job failed with
`permission_denied: write_package` when attempting to push images to
ghcr.io, even after adding `permissions: packages: write` at the job
level in `ci.yml`.

**Root cause:** Two compounding issues:

1. GitHub Actions requires `permissions: packages: write` at the
   **workflow level** (not only at the job level) to grant `GITHUB_TOKEN`
   write access to the package registry.
2. Packages created via a personal PAT are not automatically linked to
   any repository. `GITHUB_TOKEN` scoped to a repository can only write
   to packages that are explicitly linked to that repository.

**Fix:**
- Added `permissions: contents: read / packages: write` at the top-level
  `permissions` block in `ci.yml` (in addition to the job-level block).
- Linked both packages (`forum-app-cloud-deploy-backend` and
  `forum-app-cloud-deploy-frontend`) to the repository via
  Package settings → Connect repository in the GitHub UI.

---

### (c) Container running as root flagged by SonarCloud (docker:S6471)

**Problem:** SonarCloud blocked the pipeline with a security vulnerability
on `backend/Dockerfile`: the final stage did not specify a `USER`
instruction, causing the application to run as `root` inside the
container.

**Root cause:** The initial Dockerfile omitted user configuration in the
final `alpine:3.19` stage. Containers running as root violate the
principle of least privilege and are flagged by SonarCloud rule
`docker:S6471`.

**Fix:** Added a non-root system user and group to the final stage, with
ownership transferred at copy time:
```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder --chown=appuser:appgroup /app/server .
USER appuser
```
The builder stage is unaffected — `gcc` and the Go compiler require root
during compilation.

---

### (d) SQLite unable to create database.db — appuser has no write permission on /app

**Problem:** When cloning the repository from scratch and running the backend
container locally, the application failed at startup with:

```
failed to initialize database: unable to open database file: no such file or directory
```

This blocked both local testing and deployment on Render.

**Root cause:** The `WORKDIR /app` instruction in the final stage creates the
directory as `root:root` with mode `755`. The `COPY --chown=appuser:appgroup`
instruction transfers ownership of the binary but not of the directory itself.
When SQLite attempts to create `database.db` in `/app` at runtime, `appuser`
lacks write permission on the directory.

This is the materialisation of the consequence already noted in section (c):
*"any future feature that requires writing to the filesystem must ensure the
target path is writable by that user."*

**Fix:** Added a `RUN chown appuser:appgroup /app` instruction between `WORKDIR`
and `COPY` in the final stage. It executes as `root` (before `USER appuser` is
set) and transfers ownership of the directory itself:

```dockerfile
WORKDIR /app
RUN chown appuser:appgroup /app
COPY --from=builder --chown=appuser:appgroup /app/server .
USER appuser
```

`appuser` can now create new files under `/app`, including `database.db`.

---

### (e) Render API key hardcoded in docs/COMMANDS.md

**Problem:** The Render API key used for manual PROD rollback instructions
(`docs/COMMANDS.md`, section 10 — "Rollback in Render") was committed in
plain text as a real value (`export RENDER_API_KEY="rnd_..."`) instead of a
placeholder. Since the repository remote is public, the key was exposed to
anyone with read access from the commit that introduced it onward.

**Root cause:** The rollback instructions were written with a real API key
value pasted in for a "verified, copy-pasteable" example, rather than a
placeholder. No secret-scanning step in the pipeline or pre-commit hook
caught the plain-text credential before it was pushed.

**Fix:**
- Replaced the hardcoded value with a placeholder in `docs/COMMANDS.md`:
  ```bash
  export RENDER_API_KEY="your-render-api-key"
  ```
- Rotated (regenerated) the exposed API key in the Render dashboard,
  invalidating the leaked value regardless of its continued presence in
  git history.
- Updated the corresponding GitHub Actions secret (`RENDER_API_KEY`) used
  by the `deploy-qa` and `deploy-prod` jobs with the new key value.

**Detected during:** manual code review ahead of using this repository as a
professional portfolio piece, not by any automated pipeline gate.

---

## Alternatives considered

| Alternative | Reason not chosen |
| --- | --- |
| Use `latest` tag in pipeline to avoid registry linking issues | Violates the non-negotiable constraint that images are tagged with commit SHA. Rejected without further consideration. |
| Use a PAT instead of GITHUB_TOKEN for registry push in CI | Would require storing a long-lived credential as a secret. GITHUB_TOKEN is ephemeral and scoped to the workflow run — the correct approach once packages are linked. |
| Suppress SonarCloud rule docker:S6471 | Does not fix the underlying security issue. A non-root user is straightforward to add and has no functional downside. |
| Mount a separate writable volume for the database | Correct for production persistence but does not fix the immediate startup failure when no volume is mounted. The directory still needs to be writable. |

## Consequences

- Local image builds for deployment must always specify `--platform linux/amd64`.
- Both ghcr.io packages must remain linked to the repository for `GITHUB_TOKEN` to push successfully in future pipeline runs.
- The backend container runs as `appuser` — any future feature that requires writing to the filesystem must ensure the target path is writable by that user.
- `/app` is owned by `appuser:appgroup` in the final image — SQLite can create `database.db` there both in local runs and on Render.
- `docs/COMMANDS.md` must only ever contain placeholder credential values —
  no real key, even for a "verified working example", is committed to a
  file tracked by Git.
