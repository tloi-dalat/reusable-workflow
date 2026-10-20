# Workflows

Each section: what it does, the permissions the calling job needs, and how to call it. Inputs are described
in each workflow's `inputs:`.

## build-push.yaml

Builds an image and, when `push` is true, publishes `ghcr.io/<owner>/<image>:<tag>`. The tag is
`<branch>-<sha7>`, or the git tag on a tag ref. An existing `X.Y.Z` is never overwritten. Multi-arch builds run
each platform on its own native runner, push by digest, then merge one manifest list. Sets the OCI labels
`org.opencontainers.image.source` and `.revision`.

Permissions: `contents: read, packages: write`. Outputs: `version`, `image`, `digest`.

```yaml
  core:
    permissions: { contents: read, packages: write }
    uses: tloi-dalat/reusable-workflow/.github/workflows/build-push.yaml@main
    with:
      image: oj-core
      dockerfile: Dockerfile
      platforms: '["linux/amd64"]'
      build-args: |
        KEY=value
      push: ${{ github.event_name != 'pull_request' }}
```

## release-tag.yaml

On a push to `release` or `releases/*`, tags the next version, creates the GitHub Release (notes from the
previous tag), then dispatches `workflow` on the tag. A dispatch is needed because a tag created with
`GITHUB_TOKEN` never triggers `on: push: tags`.

Permissions: `contents: write, actions: write`. Outputs: `version`, `created`.

| Push to | Builds on | Tag created |
|---|---|---|
| `release`, no tag yet | | `0.0.0` |
| `release` | `0.0.0` / `1.0.99` / `1.1.1` | `0.1.0` / `1.1.0` / `1.2.0` |
| `releases/x` cut from `1.1.1` | `1.1.1` | `1.1.2` |
| `releases/y` also cut from `1.1.0` | `1.1.1` taken | `1.1.2` |
| commit already tagged | | nothing |

Tags are plain `X.Y.Z`. A new major is pushed by hand (`git tag 2.0.0 && git push origin 2.0.0`). The caller's
build workflow needs `workflow_dispatch` and `push: tags: ['[0-9]+.[0-9]+.[0-9]+']`, and must deploy to dev only
from `refs/heads/main`.

```yaml
on:
  push:
    branches: [release, 'releases/**']
jobs:
  tag:
    permissions: { contents: write, actions: write }
    uses: tloi-dalat/reusable-workflow/.github/workflows/release-tag.yaml@main
```

## production-values.yaml

After a tag build, writes the images and one `image:` block per app (to paste into control-plane's production
`values.yaml`) to the run summary and to the tag's GitHub Release. Creates the Release if the tag was pushed by
hand. Production is deployed by PR, never by CI.

Permissions: `contents: write`.

```yaml
  production-values:
    if: github.ref_type == 'tag'
    needs: [core, wsevent]
    permissions: { contents: write }
    uses: tloi-dalat/reusable-workflow/.github/workflows/production-values.yaml@main
    with:
      apps: core,wsevent
      images: oj-core,oj-wsevent
      namespace: online-judge
      version: ${{ needs.core.outputs.version }}
```

## dispatch.yaml

Sends `sync-version` to `<owner>/control-plane`, which commits the new `image.tag` for each app. Uses a one-hour
CI App token with Contents: write on control-plane only.

```yaml
  deploy-dev:
    needs: [core]
    uses: tloi-dalat/reusable-workflow/.github/workflows/dispatch.yaml@main
    with:
      apps: core,wsevent
      namespace: online-judge
      environment: development
      version: ${{ needs.core.outputs.version }}
    secrets: inherit
```

## control-plane-repo-dispatch.yaml

The receiver, called by control-plane on `repository_dispatch: [sync-version]`. Validates the payload, bumps
`image.tag`, commits and pushes to `main` with the CI App (serialised, with retry). The App is `main`'s only
bypass, and its pushes trigger control-plane's workflows (a `GITHUB_TOKEN` push wouldn't).

Permissions: `contents: read`, `secrets: inherit`. Output: `commit` (empty when nothing changed).

## control-plane-deployments.yaml

Called by control-plane on every push to `main`. For each changed `image.tag`, reads the image's OCI labels and
creates a Deployment in the source repo, at the built commit, in environment `<env>` (`development`,
`production`). Apps of one repo changed together become one Deployment; older ones turn inactive. Third-party
images are skipped. The link is the app's `url:` in `values.yaml`, else `https://<env.SITE__DOMAIN>`.
`success` means the change is on `main`; ArgoCD applies it within minutes.

Permissions: `contents: read`, `secrets: inherit` (token with Deployments: write only).

## control-plane-lint.yaml

On control-plane PRs: `kustomize build --enable-helm` every app as ArgoCD does, validate the output with
`kubeconform` (strict, with CRD schemas), reject secret-looking values in any `values.yaml`, and check that every
image in the rendered manifests exists in its registry (anonymously, so packages must be public). Images are every
string `image` or `imageName` field (and `image.reference`), in any resource except CRDs, ConfigMaps and Secrets:
containers, init containers, and operator resources such as `MariaDB` or a CloudNativePG `Cluster`.

Permissions: `contents: read`.

## terraform-check.yaml

On terraform repo PRs, for each directory in `workspaces`: `terraform fmt`, `init -backend=false`, `validate` and
`test` with mocked providers. No secrets: plans and applies run in Atlantis. The check is named
`<caller job> / terraform (<workspace>)`.

Permissions: `contents: read`.

## django-migration-check.yaml

Fails when models changed without a migration, and puts the SQL of every new migration in the job summary.
MySQL has no transactional DDL, so reading the SQL before merging is the cheapest protection.

Permissions: `contents: read`.

## django-i18n-check.yaml

Fails when `.po` files are invalid or don't compile, are out of date with the code (re-running makemessages
changes them; line references ignored), or have untranslated, fuzzy or (unless allowed) obsolete entries in
target locales. A locale dir generated by another command sets `regenerate`. Optional `template-globs` also
checks that every `_('...')` in templates was extracted, since django_jinja's makemessages skips some.

Permissions: `contents: read`.

## django-template-check.yaml

Compiles every Django / Jinja template through Django's loader, without rendering or a database, and annotates
syntax errors.

Permissions: `contents: read`.

## docker-build-check.yaml

Builds the images on pull requests without pushing, reading `build-push.yaml`'s cache. Require only the final
`docker` job. `images` is a JSON list of `{image, dockerfile, context?, build-args?}`.

Permissions: `contents: read`.

## python-audit.yaml

pip-audit on the requirement files, resolved from scratch. Run on PRs and weekly. `ignore`: one advisory ID per
line, with `# reason` allowed.

Permissions: `contents: read`.

## pr-title-check.yaml

The PR title must follow Conventional Commits (`type(scope): summary`). The caller must also trigger on `edited`.

Permissions: `{}`.

## pr-source-check.yaml

Only `main` of this repo may be merged into `release` (inputs `base` / `head` change the rule). Skipped on PRs to
other branches. Call it from a workflow that triggers on `edited`.

Permissions: `{}`.

## workflow-lint.yaml / lint.yaml

actionlint, with shellcheck on every `run:` block: `workflow-lint.yaml` for the calling repo, `lint.yaml` for
this repo.

Permissions: `contents: read`.

## codeql.yaml

CodeQL scanning (Python, JavaScript/TypeScript, Actions by default) to the repo's Security tab. Call it on PRs and
pushes to the default branch, plus a schedule. Turn off CodeQL default setup first. The jobs pass with alerts; to
block merges, use a "Require code scanning results" ruleset rule.

Permissions: `contents: read, security-events: write, actions: read`.

```yaml
  codeql:
    permissions: { contents: read, security-events: write, actions: read }
    uses: tloi-dalat/reusable-workflow/.github/workflows/codeql.yaml@main
    with:
      queries: security-and-quality
      paths-ignore: |
        resources/mathjax
```
