# services/

One `<name>/config.json` per application deployed by the platform — never
hand-written. One of the Backstage "hello-world-*" templates opens a PR
here as the last step of scaffolding a new service; merging that PR is what
makes ArgoCD start deploying it (see `clusters/dev/applicationset-services.yaml`).

```json
{
  "name": "checkout-api",
  "namespace": "checkout-api",
  "repoURL": "https://github.com/bernadin-kabore/checkout-api.git",
  "path": "chart",
  "owner": "checkout-team"
}
```

Unlike `apps/` (platform components, whose Helm values live in *this*
repo), a service's chart and values live in *its own* repository — this
file only tells ArgoCD where to find them.
