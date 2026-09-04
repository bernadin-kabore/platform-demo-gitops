# applications/

One `<application>.json` per logical application the platform deploys — never
hand-written. The **New Application** Backstage template opens a pull request
here as the last step of scaffolding; merging it is what starts deployment.

```json
{
  "application": "checkout-platform",
  "gitopsRepoURL": "https://github.com/bernadin-kabore/checkout-platform-gitops.git",
  "owner": "team-checkout"
}
```

That is the whole registration. Three facts, and none of them describe a
service: which application exists, where its deployment state lives, and who
owns it.

## Why so little

Everything about *how* an application deploys — which services it has, which
environments they run in, what image each one is on — lives in the
application's own GitOps repository, in `argocd/applicationset.yaml` and
`environments/`. This file just tells ArgoCD to go and read it.

That split is the point of the model. An application team adds a service,
changes a memory limit, or promotes a release without ever touching this
repository. The platform team decides which applications exist at all, and that
decision is a reviewed pull request here.

`clusters/dev/applicationset-applications.yaml` turns each of these files into
one ArgoCD Application that applies the target repository's ApplicationSet,
under the deliberately tiny `app-delivery` project — see
`clusters/dev/app-delivery-project.yaml` for what that project can and cannot
do.

## Not to be confused with `apps/`

`apps/` holds *platform components* — Istio, Kyverno, the observability stack,
the AI Platform Agent. Their Helm values live in this repository because the
platform team owns them. Application deployment state does not.
