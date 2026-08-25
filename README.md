# platform-demo-gitops

Single source of truth for everything running inside the cluster. ArgoCD
watches this repo; nothing here is ever `kubectl apply`'d by a human except
the one-time bootstrap.

## How it's wired

```
bootstrap/               ArgoCD itself + the AppProject + the "root" Application
clusters/dev/            Two ApplicationSets: one discovers apps/**/config.json
                          (platform components), the other services/**/config.json
                          (application workloads)
apps/<name>/              One platform component: a kustomization.yaml that
                          Helm-inflates the upstream chart (kustomize --enable-helm)
                          plus any raw manifests (CRs, policies) it needs
services/<name>/config.json  One deployed application — points ArgoCD at that
                          service's OWN repo + Helm chart, see services/README.md
```

Adding a new platform component is: create `apps/<name>/{config.json,
kustomization.yaml,values.yaml}`, open a PR. The ApplicationSet in
`clusters/dev` picks it up automatically — no change to bootstrap or to any
other app's files.

## Components

| Component | Namespace | What it does |
|---|---|---|
| [istio](apps/istio) | `istio-system` | Service mesh: mTLS (STRICT), ingress gateway, telemetry export |
| [argo-rollouts](apps/argo-rollouts) | `argo-rollouts` | Canary/blue-green progressive delivery, with a Prometheus-backed AnalysisTemplate |
| [crossplane](apps/crossplane) | `crossplane-system` | Self-service AWS resource provisioning from inside the cluster (sample: `S3Bucket` claim) |
| [kyverno](apps/kyverno) | `kyverno` | Admission policy engine — PSS *restricted*, signed-image + SBOM verification, and more |
| [karpenter](apps/karpenter) | `kube-system` | Just-in-time node autoscaling, spot-first |
| [opencost](apps/opencost) | `opencost` | Real-time Kubernetes cost allocation by namespace/team |
| [kube-prometheus-stack](apps/observability/kube-prometheus-stack) | `monitoring` | Prometheus + Grafana + Alertmanager |
| [otel-collector](apps/observability/otel-collector) | `observability` | OTLP gateway (traces→Tempo, metrics→Prometheus) + node-level log agent (→Elasticsearch) |
| [tempo](apps/observability/tempo) | `observability` | Trace storage; Jaeger-compatible query API for Kiali |
| [kiali](apps/observability/kiali) | `istio-system` | Service mesh topology + trace visualization |
| [efk](apps/observability/efk) | `logging` | Elasticsearch + Kibana for centralized log search |

## Observability data flow

```
App (OTel SDK) ──OTLP──┐
Istio sidecars ──OTLP──┼─► otel-collector (gateway) ──► Tempo ──► Kiali (traces)
                        └─► otel-collector (gateway) ──► Prometheus ──► Grafana (metrics)
Container stdout ──────────► otel-collector (agent/DaemonSet) ──► Elasticsearch ──► Kibana (logs)
```

Kiali needs a Jaeger-API-compatible trace store; Tempo provides that while
staying fully OTLP-native end to end, so it's the natural pairing rather
than standing up a separate Jaeger.

## Bootstrap (one time, per cluster)

```bash
# 0. platform-demo-terraform-modules/envs/dev must already be applied — this repo assumes
#    the cluster, OIDC provider, and IRSA roles already exist.
aws eks update-kubeconfig --name platform-demo

# 1. Install ArgoCD itself
helm repo add argo https://argoproj.github.io/argo-helm
helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace \
  -f bootstrap/argocd-values.yaml

# 2. Create the AppProject and the root Application — everything else
#    cascades from here automatically
kubectl apply -f bootstrap/project.yaml
kubectl apply -f bootstrap/root-app.yaml
```

Before step 2, replace the placeholder `REPLACE_ACCOUNT_ID` / role ARNs in
`apps/crossplane/runtime-config.yaml`, `apps/karpenter/values.yaml`, and
`apps/opencost/values.yaml` with the real outputs of
`terraform -chdir=platform-demo-terraform-modules/envs/dev output`, and the placeholder
`bernadin-kabore` repo URLs throughout with your actual GitHub org.

## Why Kustomize + Helm inflation instead of ArgoCD Helm sources directly

Every app directory is plain Kustomize with `helmCharts:` inflation rather
than ArgoCD's native Helm source type, so extra raw manifests (Kyverno
policies, Crossplane compositions, Karpenter NodePools) can live
side-by-side with the Helm release in one Application and one sync — no
juggling multi-source Applications per component.
