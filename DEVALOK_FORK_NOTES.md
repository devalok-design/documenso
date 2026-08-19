# Devalok Fork Notes

This is `devalok-design/documenso`, a fork of `documenso/documenso` used to self-host Documenso on Railway at `sign.devalok.in`.

It is **not** a real GitHub fork — Railway's eject pushed a squashed initial commit. Upstream sync is done via `git remote`, not the GitHub "Sync fork" button.

## Sync history

| Date | Upstream HEAD | Notes |
|---|---|---|
| 2026-05-04 | (initial Railway eject) | Squashed initial commit `af5d802` |
| 2026-06-04 | `0ecde7a` | First upstream sync. 86 commits absorbed (~v2.10.x → v2.11.x territory). 4 additive Prisma migrations applied. Guards re-applied to all 14 upstream-only workflows. |
| 2026-08-19 | `7533016` (v2.17.0) | Second upstream sync. 120 commits, v2.11 → v2.17.0. 4 additive Prisma migrations. **Motivation:** deployed commit predated upstream `583e35c7` ("fix: ensures new expire on setSessionCookie", #2708), which froze every session cookie's `Expires` at process-start + 30 days — all logins silently broke on 2026-07-04. Guards re-applied to 13 workflows (upstream deleted `issue-assignee-check.yml` and `pr-review-reminder.yml`). |

## Upstream sync

Histories are **unrelated** (squashed eject), so `git merge upstream/main` fails with
`refusing to merge unrelated histories`. Every sync is a squash-import of upstream's tree.

Import the tree wholesale — do **not** hand-apply a diff. The 2026-06-04 sync applied upstream's
additions and modifications but silently kept files upstream had **deleted** (`packages/eslint-config/`,
`packages/prettier-config/`, `prettier.config.cjs`, `.prettierignore`, `.eslintrc.cjs`,
`packages/lib/server-only/document/send-completed-email.ts`, stale embedding docs). `read-tree` propagates
deletions; a diff does not.

```bash
git remote add upstream https://github.com/documenso/documenso.git   # one-time
git fetch upstream
git checkout -b chore/upstream-merge-<yymm>

# fork-only delta = workflow guards + this file + the Dockerfile NODE_ENV line.
# Capture it against the PREVIOUS sync base.
git diff <previous-sync-base> HEAD -- .github/workflows DEVALOK_FORK_NOTES.md docker/Dockerfile > ../fork-delta.patch

# working tree becomes upstream/main exactly (adds, mods AND deletes)
git read-tree -u --reset upstream/main

# re-apply fork guards; --exclude any workflow upstream has since deleted
git apply -3 --index ../fork-delta.patch
```

Then audit guard coverage per job before pushing (see below), and check
`git diff <previous-sync-base> upstream/main -- packages/prisma/migrations` for pending migrations.

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
4. Audit every job, not every file — a guarded file can still gain an unguarded job:

```bash
python - <<'EOF'
import re, glob
for f in sorted(glob.glob('.github/workflows/*.yml')):
    lines = open(f, encoding='utf-8').read().splitlines()
    i = next((n for n, l in enumerate(lines) if l.rstrip() == 'jobs:'), None)
    if i is None: continue
    cur = None
    print(f)
    for l in lines[i + 1:]:
        m = re.match(r'^  ([A-Za-z0-9_-]+):\s*$', l)
        if m:
            cur = {'n': m.group(1), 'g': False}
            print('   ', cur)
            continue
        if cur and "github.repository == 'documenso/documenso'" in l:
            print('    ^ guarded')
EOF
```

Expected open (intentional): `ci.yml/build_docker`, `codeql-analysis.yml/analyze`. Everything else guarded.

### Currently guarded workflows

| File | Why |
|---|---|
| `ci.yml` (build_app job only) | Upstream npm cache hides a lock-file/babel mismatch; cold-cache fork CI fails. `build_docker` (the real prod signal) still runs on fork. |
| `deploy.yml` | Pushes `main` → `release` using upstream `GH_TOKEN` |
| `e2e-tests.yml` | Uses `warp-ubuntu-*` runners only available to upstream |
| `first-interaction.yml` | Welcome message linking to upstream Discord |
| `issue-labeler.yml` | Upstream issue triage |
| `issue-opened.yml` | Upstream issue triage |
| `pr-labeler.yml` | Upstream PR triage |
| `publish.yml` | Publishes to `documenso/*` on DockerHub + `ghcr.io/documenso` |
| `semantic-pull-requests.yml` | Conventional-commit PR title enforcement |
| `stale.yml` | Upstream community issue/PR stale-bot |
| `translations-force-pull.yml` | Crowdin (no fork credentials) |
| `translations-pull.yml` | Crowdin (no fork credentials) |
| `translations-upload.yml` | Crowdin (no fork credentials) |

### Kept (run on fork)

- `ci.yml` — only the `build_docker` job (Docker build = real prod signal). `build_app` job guarded above.
- `codeql-analysis.yml` — security scan (uses only `GITHUB_TOKEN`)

## Fork-local code changes

Keep this list short — every entry is a merge conflict waiting to happen. Anything here MUST be in the
`fork-delta.patch` capture above, or the next `read-tree` import silently drops it.

| File | Change | Why |
|---|---|---|
| `docker/Dockerfile` | `ENV NODE_ENV="production"` in the **runner** stage | Upstream never sets `NODE_ENV`, so self-hosters run React Router's *development* bundle and `useSecureCookies` (`packages/lib/constants/auth.ts`) stays false — session cookies get no `Secure` flag and no `__Secure-` prefix. **Must be runner-stage only**: as a Railway service variable it also reaches the build, where `npm ci` (installer stage) would omit devDependencies and break `turbo run build`. Worth upstreaming; until then it lives here. |

## Railway deploy

- Project: `documenso` (`9c5f62f8-2820-4ae0-862f-7c39db14e3f6`)
- Service: `documenso` builds from this repo's `main`
- Custom domain: `sign.devalok.in` (Cloudflare DNS-only / grey cloud)

Do **not** use the Railway MCP `updateService` call — it wipes networking config. UI only for env-var and networking changes. Read-only MCP calls (status, logs) are fine.
