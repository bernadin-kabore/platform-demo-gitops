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
                          plus any raw manifests (CRs, policies) it needs — or,
                          for first-party components with no upstream chart
                          (ai-platform-agent), raw manifests alone
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
| [ai-platform-agent](apps/ai-platform-agent) | `ai-platform` | The AI Platform Agent: turns a natural-language request from Backstage into pull requests across the four platform repos, gated by its own eval suite |
| [istio](apps/istio) | `istio-system` | Service mesh: mTLS (STRICT), ingress gateway, telemetry export |
| [argo-rollouts](apps/argo-rollouts) | `argo-rollouts` | Canary/blue-green progressive delivery, with a Prometheus-backed AnalysisTemplate |
| [crossplane](apps/crossplane) | `crossplane-system` | Self-service AWS resource provisioning from inside the cluster (sample: `S3Bucket` claim) |
| [kyverno](apps/kyverno) | `kyverno` | Admission policy engine — PSS *restricted*, signed-image + SBOM verification, and more |
| [karpenter](apps/karpenter) | `kube-system` | Just-in-time node autoscaling, spot-first |
| [opencost](apps/opencost) | `opencost` | Real-time Kubernetes cost allocation by namespace/team |
| [kube-prometheus-stack](apps/observability/kube-prometheus-stack) | `monitoring` | Prometheus + Grafana + Alertmanager |
| [otel-collector](apps/observability/otel-collector) | `observability` | OTLP gateway (traces→Tempo, metrics→Prometheus, logs→Elasticsearch) + node-level log agent that ships container stdout to the gateway |
| [tempo](apps/observability/tempo) | `observability` | Trace storage; Jaeger-compatible query API for Kiali |
| [kiali](apps/observability/kiali) | `istio-system` | Service mesh topology + trace visualization |
| [efk](apps/observability/efk) | `logging` | Elasticsearch + Kibana for centralized log search |

## Observability data flow

All three signals converge on the **gateway**, so exactly one place in the
platform knows which backends exist. Applications depend on the
OpenTelemetry contract alone and never name Tempo, Prometheus or
Elasticsearch.

```
traces    app (OTel SDK)     --OTLP-->  gateway  -->  Tempo          -->  Grafana / Kiali
metrics   app (OTel SDK)     --OTLP-->  gateway  -->  Prometheus     -->  Grafana        (remote write)
logs      app stdout (JSON)  --> agent (DaemonSet) --OTLP--> gateway --> Elasticsearch --> Kibana
```

**Log/trace correlation.** Services emit structured JSON containing
`trace_id` and `span_id`; the agent's `filelog` receiver promotes those to
first-class log-record IDs, which is what lets Grafana pivot from an error
log straight to its trace. Non-JSON lines - panics, JVM stack traces,
pre-logger startup output - pass through unparsed rather than being dropped.

**Why logs are collected rather than pushed by the SDK:** records buffered
inside a process are lost when it crashes, and crash logs matter most. The
container runtime has already written the line to disk before anything dies.

**Why metrics use remote write rather than a scraped exporter:** with two
gateway replicas a scrape-based exporter gives Prometheus two targets each
holding a partial view, and series migrate between them as applications
rebalance, churning the instance label. Remote write presents one logical
destination instead.

Kiali needs a Jaeger-API-compatible trace store; Tempo provides that while
staying fully OTLP-native end to end, so it's the natural pairing rather
than standing up a separate Jaeger.

The reasoning behind each of these choices, including the options rejected
and what they cost, is recorded in
[`docs/observability/README.md`](docs/observability/README.md).

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
kubectl apply -f clusters/dev/workloads-project.yaml
kubectl apply -f bootstrap/root-app.yaml
```

The IRSA role ARNs in `apps/crossplane/runtime-config.yaml`,
`apps/karpenter/values.yaml` and `apps/opencost/values.yaml` are populated
for the current dev account. **Deploying into a different AWS account means
replacing that account ID** with the real outputs of
`terraform -chdir=platform-demo-terraform-modules/envs/dev output`.

Forking also means replacing the `bernadin-kabore` repo URLs throughout with
your own GitHub org - including `targetRevision` in
`bootstrap/root-app.yaml` and `clusters/dev/applicationset.yaml`, which pin
the branch ArgoCD actually reads.

## Why Kustomize + Helm inflation instead of ArgoCD Helm sources directly

Every app directory is plain Kustomize with `helmCharts:` inflation rather
than ArgoCD's native Helm source type, so extra raw manifests (Kyverno
policies, Crossplane compositions, Karpenter NodePools) can live
side-by-side with the Helm release in one Application and one sync — no
juggling multi-source Applications per component.
