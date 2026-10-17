# One-time setup

1. **Keep this repo public.** The app repos that call it are public, and a public repo can't call workflows in a
   private one. This repo holds no secrets.
2. **CI GitHub App** (the only credential CI uses):
   1. Org settings → Developer settings → GitHub Apps → New, e.g. `tloi-ci`. Webhook off. Repository
      permissions: Contents read/write, Deployments read/write. Installable only on this account.
   2. Install it on `tloi-dalat`, all repositories.
   3. Generate a private key. Add org secrets for all repositories: `CI_APP_ID` and `CI_APP_PRIVATE_KEY` (the
      whole `.pem`). Delete the downloaded `.pem`.
   4. Callers pass them with `secrets: inherit`. Each run mints a one-hour token with only what the job needs.
3. **GHCR packages**: the first push creates `ghcr.io/tloi-dalat/<image>`. Make it public: the clusters pull
   without credentials and `control-plane-deployments.yaml` reads the labels anonymously.

## Conventions

- Image tags are immutable: `<branch>-<sha7>` or `X.Y.Z`. Nothing publishes `latest`.
- Protect release tags and the `release` / `releases/**` branches with rulesets (terraform-github).
- Callers pin `@main`. Once stable, tag `v1` and pin to it.
