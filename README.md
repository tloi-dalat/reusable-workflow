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

CI never holds a cluster credential. Its only write is a git commit to control-plane.

## Workflows

| Workflow | Called from | Purpose |
|---|---|---|
| `build-push.yaml` | app repos | Build an image; on push, publish `ghcr.io/<owner>/<image>:<branch>-<sha7>`. Multi-arch on native runners, merged by digest |
| `dispatch.yaml` | app repos | Ask control-plane to deploy a version to `<env>/<ns>/<apps>` |
| `control-plane-repo-dispatch.yaml` | control-plane | Receive `sync-version`, validate it, bump tags, commit, push (serialised, with retry) |
| `control-plane-lint.yaml` | control-plane | `kustomize build --enable-helm` every app, `kubeconform` the output, reject plaintext secrets |
| `django-migration-check.yaml` | Django app repos | Fail on missing migrations; put the SQL of new migrations in the PR summary |
| `lint.yaml` | this repo | actionlint (+ shellcheck) on these workflows |

### build-push.yaml

```yaml
jobs:
  site:
    permissions: { contents: read, packages: write }
    uses: tloi-dalat/reusable-workflow/.github/workflows/build-push.yaml@main
    with:
      image: oj-site                     # → ghcr.io/tloi-dalat/oj-site
      dockerfile: Dockerfile
      context: .
      platforms: '["linux/amd64"]'       # JSON list; arm64 runs on ubuntu-24.04-arm
      submodules: recursive              # passed to actions/checkout
      build-args: |
        KEY=value
      push: ${{ github.event_name != 'pull_request' }}
```

Outputs: `version` (the tag), `image`, `digest` (empty when not pushed).

### dispatch.yaml

```yaml
  deploy-dev:
    needs: [site]
    uses: tloi-dalat/reusable-workflow/.github/workflows/dispatch.yaml@main
    with:
      apps: django,wsevent                # directories under clusters/<env>/<namespace>/
      namespace: online-judge
      environment: dev                    # dev | prd
      version: ${{ needs.site.outputs.version }}
    secrets:
      CONTROL_PLANE_TOKEN: ${{ secrets.CONTROL_PLANE_TOKEN }}
```

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
   image pull secret in control-plane.

## Conventions

- Tags are immutable: `<branch>-<sha7>` from branches, the git tag from tag pushes. Nothing publishes `latest`.
- Callers pin `@main` today. Once these settle, tag a release (`v1`) and pin to it so a change here
  can't break every repo at once.
