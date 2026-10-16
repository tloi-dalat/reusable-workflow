# reusable-workflow

Shared GitHub Actions workflows for every TLOI service. App repos keep only thin
callers; everything that builds, pushes or deploys lives here.

```
app repo (push to main)
  └─ build-push.yaml   → ghcr.io/tloi-dalat/<image>:main-<sha7>
  └─ dispatch.yaml     → repository_dispatch "sync-version" to tloi-dalat/control-plane
                            └─ control-plane-repo-dispatch.yaml
                                 → bot commit: clusters/<env>/<ns>/<app>/values.yaml image.tag
                                 → ArgoCD syncs it
```

```
app repo (merge main → release, or push to releases/<name>)
  └─ release-tag.yaml  → GitHub Release + tag X.Y.Z  (release: minor+1, releases/*: patch+1)
                       → workflow_dispatch workflow.yaml on the tag
                            └─ build-push.yaml → ghcr.io/tloi-dalat/<image>:X.Y.Z  (never overwritten)
production: a reviewed PR in control-plane sets clusters/production/<ns>/<app>/values.yaml image.tag: X.Y.Z
```

CI never holds a cluster credential. Its only write is a git commit to control-plane.

## Workflows

| Workflow | Called from | Purpose |
|---|---|---|
| `build-push.yaml` | app repos | Build an image; on push, publish `ghcr.io/<owner>/<image>:<branch>-<sha7>`. Multi-arch on native runners, merged by digest |
| `release-tag.yaml` | app repos | On a push to `release` / `releases/*`, tag the next `X.Y.Z` (minor / patch), create the GitHub Release, dispatch the image build on the tag |
| `production-values.yaml` | app repos | On a tag build, add the images and the `image:` blocks to paste into control-plane's production values to the run summary **and the GitHub Release** (production is deployed by PR, not CI) |
| `dispatch.yaml` | app repos | Ask control-plane to deploy a version to `<env>/<ns>/<apps>` |
| `control-plane-repo-dispatch.yaml` | control-plane | Receive `sync-version`, validate it, bump tags, commit, push (serialised, with retry) |
| `control-plane-deployments.yaml` | control-plane | For each image tag changed on control-plane main, create a Deployment (environment `development` / `production`) in the app repo the image was built from, at the exact commit (OCI labels) |
| `control-plane-lint.yaml` | control-plane | `kustomize build --enable-helm` every app, `kubeconform` the output, reject plaintext secrets |
| `django-migration-check.yaml` | Django app repos | Fail on missing migrations; put the SQL of new migrations in the PR summary |
| `django-i18n-check.yaml` | Django app repos | Fail when .po files are invalid or don't compile, out of date with the code, or have untranslated / fuzzy / obsolete entries; optionally (`template-globs`) when a template `_('...')` is one makemessages cannot extract. A locale dir that another command generates sets `regenerate` to run it instead of makemessages |
| `django-template-check.yaml` | Django app repos | Compile every Django / Jinja template (no database), annotating syntax errors with file and line |
| `docker-build-check.yaml` | Repos with images | Build the images on pull requests without pushing, reading `build-push.yaml`'s cache; require its `docker` job |
| `python-audit.yaml` | Python repos | pip-audit the requirement files (resolved from scratch); run on PRs and weekly; `ignore` takes advisory IDs with reasons |
| `pr-title-check.yaml` | Any repo | Pull request title must be Conventional Commits (`type(scope): summary`); caller must also trigger on `edited` |
| `workflow-lint.yaml` | Any repo | actionlint (+ shellcheck) on the calling repo's own workflows |
| `codeql.yaml` | Any repo | CodeQL code scanning (Python, JavaScript/TypeScript, Actions by default); results go to the repo's Security tab and PR alerts |
| `lint.yaml` | this repo | actionlint (+ shellcheck) on these workflows |

### build-push.yaml

```yaml
jobs:
  core:
    permissions: { contents: read, packages: write }
    uses: tloi-dalat/reusable-workflow/.github/workflows/build-push.yaml@main
    with:
      image: oj-core                     # → ghcr.io/tloi-dalat/oj-core
      dockerfile: Dockerfile
      context: .
      platforms: '["linux/amd64"]'       # JSON list; arm64 runs on ubuntu-24.04-arm
      submodules: recursive              # passed to actions/checkout
      build-args: |
        KEY=value
      push: ${{ github.event_name != 'pull_request' }}
```

Outputs: `version` (the tag), `image`, `digest` (empty when not pushed).

### release-tag.yaml

```yaml
# .github/workflows/release.yaml in the app repo
on:
  push:
    branches: [release, 'releases/**']
jobs:
  tag:
    permissions: { contents: write, actions: write }
    uses: tloi-dalat/reusable-workflow/.github/workflows/release-tag.yaml@main
    # with:
    #   release-branch: release        # defaults
    #   hotfix-prefix: releases/
    #   initial-version: 1.0.0
    #   workflow: workflow.yaml        # dispatched on the new tag; needs `workflow_dispatch`
```

| Push to | Tag it builds on | Tag created |
|---|---|---|
| `release` | none | `1.0.0` |
| `release` | `1.0.0` / `1.0.99` / `1.1.1` | `1.1.0` / `1.1.0` / `1.2.0` |
| `releases/x` cut from `1.0.0` | `1.0.0` (even if `1.3.0` exists) | `1.0.1` |
| `releases/x` cut from `1.1.1` | `1.1.1` | `1.1.2` |
| a second `releases/y` cut from `1.1.0` | `1.1.1` already taken | `1.1.2` (patches never collide) |
| any, commit already tagged | | nothing |

Outputs: `version`, `created`. Tags are plain `X.Y.Z`; anything else (`v4.0.1`, `1.2.0-rc1`) is ignored
when computing the next one. A new major is pushed by hand (`git tag 2.0.0 && git push origin 2.0.0`),
which the caller's `on: push: tags` builds.

The caller's build workflow must: trigger on `workflow_dispatch` (and `push: tags: ['[0-9]+.[0-9]+.[0-9]+']`
for hand-pushed tags), and keep its dev deploy to `refs/heads/main`, or a tag build would deploy to dev.

### production-values.yaml

```yaml
  production-values:
    if: github.ref_type == 'tag'
    needs: [core, wsevent]
    permissions: { contents: write }      # edits (or creates) the GitHub Release for the tag
    uses: tloi-dalat/reusable-workflow/.github/workflows/production-values.yaml@main
    with:
      apps: core,wsevent                  # directories under clusters/production/<namespace>/
      images: oj-core,oj-wsevent          # listed in the Release
      namespace: online-judge
      version: ${{ needs.core.outputs.version }}
      # environment: production           # default
```

### control-plane-deployments.yaml

Called by control-plane (`deployments.yaml` on push to main, and `repo-dispatch.yaml` after the bot
commit, since a commit pushed with `GITHUB_TOKEN` triggers no workflow). The app repo comes from the
image's `org.opencontainers.image.source` label and the commit from `org.opencontainers.image.revision`,
both set by `build-push.yaml`. Apps of one repo changed together (`core` + `wsevent`) become one
Deployment. Status `success` means "on control-plane main"; ArgoCD applies it within minutes.

### dispatch.yaml

```yaml
  deploy-dev:
    needs: [core]
    uses: tloi-dalat/reusable-workflow/.github/workflows/dispatch.yaml@main
    with:
      apps: core,wsevent                  # directories under clusters/<env>/<namespace>/
      namespace: online-judge
      environment: development            # development | production
      version: ${{ needs.core.outputs.version }}
    secrets:
      CONTROL_PLANE_TOKEN: ${{ secrets.CONTROL_PLANE_TOKEN }}
```

### codeql.yaml

Call it from its own workflow, on pull requests **and** pushes to the default branch (pull request
results are compared against the default branch's analysis), plus a schedule for new queries:

```yaml
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  schedule:
    - cron: '43 2 * * 1'

jobs:
  codeql:
    permissions: { contents: read, security-events: write, actions: read }
    uses: tloi-dalat/reusable-workflow/.github/workflows/codeql.yaml@main
    with:
      languages: '["python", "javascript-typescript", "actions"]'   # the default
      queries: security-and-quality         # default | security-extended (the default) | security-and-quality
      paths-ignore: |                       # vendored code
        resources/mathjax
```

Turn off CodeQL *default setup* (Settings → Code security) first, or the upload is rejected. The jobs
pass even with alerts; to block merges, add a "Require code scanning results" rule to the branch
ruleset rather than requiring the `analyze` jobs.

## One-time setup

1. **Keep this repo public.** The app repos that call it (`online-judge`, `judge-server`) are public,
   and GitHub doesn't let a public repo call workflows stored in a private one. The org-access
   setting only helps private callers. This repo holds no secrets.
2. **`CONTROL_PLANE_TOKEN`**: an org secret, or a secret in each app repo. Use a fine-grained PAT
   (or GitHub App) scoped to **`tloi-dalat/control-plane` only**, with **Contents: Read and write**.
   That's all `repository_dispatch` needs.
3. **GHCR packages**: the first push creates `ghcr.io/tloi-dalat/<image>`, linked to the source repo
   through the `org.opencontainers.image.source` label. Either make each package **public**
   (the cluster then pulls with no credentials, which is the simplest option for AGPL code), or keep them private and add an
   image pull secret in control-plane. `control-plane-deployments.yaml` reads the image labels
   anonymously, so it needs the packages public (or a registry login added to it).
4. **`DEPLOYMENTS_TOKEN`** (control-plane secret): a fine-grained PAT (or GitHub App) on the **app repos**
   (`TLOJ`, `judge-server`) with **Deployments: Read and write**. It only creates Deployments there.
5. **`ENVIRONMENT_URLS`** (control-plane repository *variable*, optional): JSON shown as each
   Deployment's link, e.g. `{"development": "https://oj.tloi.shop", "production": "https://oj.tloi.vn"}`.

## Conventions

- Tags are immutable: `<branch>-<sha7>` from branches, the git tag (`X.Y.Z`) from tag refs. `build-push.yaml`
  refuses to overwrite an `X.Y.Z` that already exists. Nothing publishes `latest`.
- Release tags (and the `release` / `releases/**` branches) should be protected by a ruleset: no deletion,
  no force-push, creation limited to maintainers + GitHub Actions.
- Callers pin `@main` today. Once these settle, tag a release (`v1`) and pin to it so a change here
  can't break every repo at once.
