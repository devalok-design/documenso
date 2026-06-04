# Devalok Fork Notes

This is `devalok-design/documenso`, a fork of `documenso/documenso` used to self-host Documenso on Railway at `sign.devalok.in`.

It is **not** a real GitHub fork — Railway's eject pushed a squashed initial commit. Upstream sync is done via `git remote`, not the GitHub "Sync fork" button.

## Sync history

| Date | Upstream HEAD | Notes |
|---|---|---|
| 2026-05-04 | (initial Railway eject) | Squashed initial commit `af5d802` |
| 2026-06-04 | `0ecde7a` | First upstream sync. 86 commits absorbed (~v2.10.x → v2.11.x territory). 4 additive Prisma migrations applied. Guards re-applied to all 14 upstream-only workflows. |

## Upstream sync

```bash
git remote add upstream https://github.com/documenso/documenso.git   # one-time
git fetch upstream
git merge upstream/main
git push origin main
```

## CI / workflow conventions

Every workflow that depends on upstream-only secrets or infrastructure (Crowdin, DockerHub, Warp runners, ghcr.io/documenso, upstream community automation) is **guarded** with:

```yaml
jobs:
  <job-name>:
    if: github.repository == 'documenso/documenso'
```

This is intentional. The workflows are left in place so upstream merges stay clean (no merge conflicts on `.github/workflows/*`), but the jobs short-circuit on the fork.

**After every `git merge upstream/main`:**

1. Diff workflow changes: `git diff HEAD~1 .github/workflows/`
2. If upstream added a new job that depends on their secrets/infra, add the same `if:` guard before pushing.
3. If upstream added a new `.yml` file that's entirely upstream-only, add the guard to every job in it.

### Currently guarded workflows

| File | Why |
|---|---|
| `ci.yml` (build_app job only) | Upstream npm cache hides a lock-file/babel mismatch; cold-cache fork CI fails. `build_docker` (the real prod signal) still runs on fork. |
| `deploy.yml` | Pushes `main` → `release` using upstream `GH_TOKEN` |
| `e2e-tests.yml` | Uses `warp-ubuntu-*` runners only available to upstream |
| `first-interaction.yml` | Welcome message linking to upstream Discord |
| `issue-assignee-check.yml` | Upstream community management |
| `issue-labeler.yml` | Upstream issue triage |
| `issue-opened.yml` | Upstream issue triage |
| `pr-labeler.yml` | Upstream PR triage |
| `pr-review-reminder.yml` | Upstream reviewer reminder |
| `publish.yml` | Publishes to `documenso/*` on DockerHub + `ghcr.io/documenso` |
| `semantic-pull-requests.yml` | Conventional-commit PR title enforcement |
| `stale.yml` | Upstream community issue/PR stale-bot |
| `translations-force-pull.yml` | Crowdin (no fork credentials) |
| `translations-pull.yml` | Crowdin (no fork credentials) |
| `translations-upload.yml` | Crowdin (no fork credentials) |

### Kept (run on fork)

- `ci.yml` — only the `build_docker` job (Docker build = real prod signal). `build_app` job guarded above.
- `codeql-analysis.yml` — security scan (uses only `GITHUB_TOKEN`)

## Railway deploy

- Project: `documenso` (`9c5f62f8-2820-4ae0-862f-7c39db14e3f6`)
- Service: `documenso` builds from this repo's `main`
- Custom domain: `sign.devalok.in` (Cloudflare DNS-only / grey cloud)

Do **not** use the Railway MCP `updateService` call — it wipes networking config. UI only for env-var and networking changes. Read-only MCP calls (status, logs) are fine.
