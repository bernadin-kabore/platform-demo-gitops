# The AI layer, from this repository's point of view

Sibling to [`docs/observability/README.md`](../observability/README.md): the
decisions, and what they cost, rather than the manifests.

The agent's own source, prompts and eval suite live in
[`platform-demo-ai-agent`](https://github.com/bernadin-kabore/platform-demo-ai-agent).
This document covers only the part that touches the cluster.

## What actually changed in the cluster

One namespace, one Deployment, and a service account with an IRSA annotation.
That is the entire cluster-side footprint of "the platform is now AI-assisted".

It is small because the agent does not do anything to the cluster. It reads
repositories over the GitHub API and writes pull requests over the GitHub API.
It has no kubeconfig, no ServiceAccount token mounted anywhere but its own pod,
no RBAC beyond the default, and no ArgoCD credentials. If it were deleted
tomorrow, every pull request it has ever opened would still be there and every
gate they passed through would still work.

## Decisions

### The agent is subject to the platform's own admission policies

`apps/ai-platform-agent/namespace.yaml` labels the namespace
`platform.acme.io/managed: "true"`, which opts it into `pod-security-restricted`,
`require-labels`, `require-probes` and `require-resource-limits` — the policies
that apply to application workloads and that most platform components in this
repository are excluded from.

Considered and rejected: adding `ai-platform` to those policies' exclude lists,
the way `monitoring`, `observability` and `crossplane-system` are excluded.
Those exclusions exist because the components in them genuinely need elevated
access to manage the cluster. This one does not need any, and exempting the
component that can open pull requests across the whole platform from the
platform's own rules would be exactly backwards.

Cost: the Deployment has to be written properly — non-root, read-only root
filesystem, dropped capabilities, both probes, explicit requests and limits.
That is not really a cost.

### A Deployment, not an Argo Rollout

Every application workload on this platform deploys as an Argo Rollout with
canary analysis. This one does not.

Canary analysis works by splitting request traffic between two revisions and
comparing error rates over a pause window. This service handles a handful of
long-running requests a day. There is no traffic to split, and the
`istio-success-rate` AnalysisTemplate would evaluate against approximately zero
samples and pass regardless of whether the new revision works. A Rollout here
would be ceremony that produces a strictly worse deploy than a rolling update.

The honest way to catch a bad agent revision is the eval suite in its own
repository, before the image is ever built.

### The image bump arrives as a pull request, not a push

Scaffolded services push their image-tag bump straight to their own `main`,
because `platform-deploy-bot` has a ruleset bypass on those repositories. This
component deploys out of *this* repository, whose `main` is protected by
`envs/github-repos` in Terraform with no bypass actor at all — so its CI opens a
pull request that a human merges.

Keep that asymmetry. An agent that can open pull requests across the platform
should not also be able to ship a new version of itself while nobody is looking.

### Egress is constrained, but not precisely

`networkpolicy.yaml` allows ingress only from the Backstage namespace, and
egress only to DNS, the OpenTelemetry gateway, and HTTPS outside the cluster's
private ranges — with IMDS explicitly blocked, since IRSA uses a projected
service account token rather than the node's instance role.

"HTTPS to anywhere public" is a real narrowing over "all egress" and is not a
precise control. Bedrock and the GitHub API have no address range worth
encoding in a NetworkPolicy. Doing it properly means an Istio egress gateway
with hostname-based rules, which is a mesh change rather than a NetworkPolicy
change. Recorded here rather than faked in the manifest.

## The dependency this makes urgent

The agent's GitHub App private key is the second concrete need for secrets
management on this platform, after Grafana's admin password.

There is no delivery mechanism, so `deployment.yaml` references a Secret that is
not in Git and must be created by hand once, with the exact command written in a
comment beside the reference. That is the honest state: a manual step, marked as
one, in the place someone will look.

It is also the strongest argument yet for
[`PLATFORM_ROADMAP.md`](../../../PLATFORM_ROADMAP.md) Part 2 item 1. When
External Secrets Operator lands, those three `secretKeyRef` blocks become one
`ExternalSecret` pointing at Secrets Manager and the manual step disappears.

Note what is *not* in that Secret: there is no model API key. Claude is reached
through Bedrock with IRSA, so the credential that would otherwise have been the
most sensitive thing in this namespace does not exist. That was the deciding
factor in choosing Bedrock over the first-party API — see the agent
repository's README.

## What is not built

- **No alerting on the agent itself.** It emits traces and metrics over OTLP
  like everything else, and there is no `PrometheusRule` watching them — because
  there is not a single `PrometheusRule` anywhere in this repository yet
  (roadmap Part 2 item 2). A run that fails silently is currently visible only
  in Grafana, if someone looks.
- **No dashboard.** Same reason; the natural first one is agent runs by outcome,
  eval score distribution, and Bedrock token spend.
- **No cost attribution for Bedrock.** OpenCost attributes Kubernetes cost by
  the `team` label, which this component carries, but the model spend lands on
  the AWS bill outside anything OpenCost sees.
