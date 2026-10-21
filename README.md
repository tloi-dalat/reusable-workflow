# reusable-workflow

Shared GitHub Actions workflows for TLOI. App repos keep thin callers; everything that builds, tags or deploys
lives here. CI never holds a cluster credential: its only write is a commit to control-plane.

```
push to main          build-push → :main-<sha7>   dispatch → control-plane bot commit → ArgoCD (development)
main → release        release-tag → X.(Y+1).0     build-push → :X.Y.Z   production-values → Release notes
push to releases/*    release-tag → X.Y.(Z+1)     production: a reviewed control-plane PR sets image.tag
```

- `docs/workflows.md`: every workflow, its permissions and how to call it
