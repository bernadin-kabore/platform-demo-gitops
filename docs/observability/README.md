# Observability architecture decisions

A record of the design choices made for this platform's observability stack —
**what was chosen, what was rejected, and why**. The rejected options are
documented as fully as the chosen ones, because the reasoning is the part worth
keeping.

Each decision below has the same shape: the question, the options, the call, what
it costs, when to revisit, and what to study to understand it properly.

**Status:** decided, not yet implemented. Implementation begins at Phase 3.

---

## Decision 1 — Logs leave the pod via `filelog`, not the OTel SDK

**Chosen:** the application writes structured JSON to stdout; the OpenTelemetry
Collector agent (a DaemonSet, one pod per node) tails the container log file and
forwards records into the OTel pipeline.

**Rejected:** the OpenTelemetry SDK inside the application holds log records and
pushes them over OTLP itself.

### The concept

The two options differ only in the **first hop**. Everything after it is identical:
records enter an OTel Collector pipeline, get enriched by the `k8sattributes`
processor, get batched, and leave through an OTel exporter.

This matters because it is easy to assume `filelog` means "not doing
OpenTelemetry." It does not. `filelog` is a first-class OpenTelemetry Collector
receiver, and logs collected that way are OTel logs with OTel semantic
conventions. The application still depends on nothing but OpenTelemetry
conventions either way. The real question is narrower:

> Does the application push its own logs, or does the platform pick them up?

### Why `filelog` won

The deciding argument is **reliability, not effort**. Log records buffered by an
SDK live in the application's memory until they are flushed. If the process dies
from a panic, an OOM kill, or a crash during startup, those buffered records die
with it — and those are precisely the logs you most want to read. With `filelog`,
the container runtime has already written the line to disk before anything can
crash, so the collector picks it up regardless.

Secondary benefits: it is language-agnostic and needs no logging code in four
different language templates.

### What was given up

The SDK path produces natively structured records with trace context already
attached by the SDK, with no dependency on the application formatting its own
JSON correctly. Choosing `filelog` means **the application is now responsible for
emitting well-formed JSON containing `trace_id` and `span_id`** — the collector
cannot invent trace context from a plain text line.

A third option was considered and rejected: run both, using the SDK for normal
logs and `filelog` as a crash net. It removes the crash blind spot but produces
duplicate ingest that then has to be de-duplicated. Not worth the complexity at
this scale.

### Cost of this choice

Every language template must be changed to emit structured JSON with trace
context. Java and Python get trace injection nearly free from their existing
auto-instrumentation agents. **Go and Node need real work** — a custom `slog`
handler and a winston or pino formatter respectively.

### When to revisit

If the platform ever ingests logs from somewhere with no filesystem to tail —
serverless functions, or an external service pushing OTLP directly — the SDK path
becomes necessary for those sources. The two can coexist; the collector accepts
both.

### To understand this deeply

- OpenTelemetry Collector `filelog` receiver, and the OTel logs data model
- The OTel logging specification's distinction between *log appender* and *log
  collection*, and why OTel treats file-based collection as a supported path
- Kubernetes container log file layout and rotation (`/var/log/pods/...`), which
  is what makes `filelog` work at all
- Trace context injection: MDC in Java, `OTEL_PYTHON_LOG_CORRELATION` in Python,
  `slog` handlers in Go

---

## Decision 2 — Metrics leave the app via OTLP push, not Prometheus scrape

**Chosen:** the application pushes metrics over OTLP to the collector, exactly as
it already pushes traces. The collector exposes them for Prometheus.

**Rejected:** the application exposes a `/metrics` HTTP endpoint that Prometheus
scrapes directly every 15 seconds.

### The concept

Pull versus push. Prometheus was built around **pull**: it discovers targets and
scrapes them on a schedule, which gives it a built-in liveness signal (a target
that cannot be scraped is down) and makes the data source self-describing.
**Push** inverts this: the application decides when to emit, and something
downstream receives.

### Why OTLP push won

This was decided on **principle, not mechanics**. The platform's stated contract
is that OpenTelemetry is the single application-facing observability API, so that
backends can be replaced without touching applications.

That contract was fiction as originally implemented. Every language template
imported a Prometheus client library — `promauto` in Go, `prom-client` in Node,
`prometheus_client` in Python, Micrometer in Java — which means applications
depended directly on Prometheus, the exact coupling the contract forbids.

Choosing OTLP push makes the stated principle true rather than aspirational.

An initial recommendation to keep scrape was withdrawn: its only justification was
avoiding rework, which is not an engineering argument.

### What was given up

- **Scrape-as-liveness.** A failed scrape is itself a signal. With push, a silent
  application and a healthy-but-idle one look similar; liveness has to come from
  elsewhere (Kubernetes probes, Istio metrics).
- **Prometheus-native ergonomics.** Metric names move to OpenTelemetry semantic
  conventions — `http.server.request.duration` rather than `http_requests_total` —
  so any existing query, dashboard, or alert written against the old names must be
  rewritten.
- **A simpler mental model.** Scrape is easier to debug: you can curl the endpoint.

### A coupling that was checked, not assumed

Progressive delivery uses an Argo Rollouts `AnalysisTemplate` that aborts a canary
when the error rate regresses. That template queries **`istio_requests_total`,
emitted by the Istio sidecar — not by the application**. So changing the
application's own metrics does not affect automatic rollback.

This was verified before deciding. Worth noting as a habit: the risk in a change
like this is rarely where you expect it.

Related observation: the application's `http_requests_total` was largely redundant
with Istio's sidecar metrics for request-rate purposes anyway.

### Cost of this choice

All four templates lose their Prometheus client library, their `/metrics`
endpoint, and the shared chart's `ServiceMonitor`. `OTEL_METRICS_EXPORTER` flips
from `none` to `otlp`. The collector's Prometheus exporter becomes the single
scrape target.

### To understand this deeply

- Pull versus push monitoring, and why Prometheus chose pull
- OpenTelemetry metrics semantic conventions, and how they differ from Prometheus
  naming idioms
- The collector's `prometheus` exporter versus `prometheusremotewrite`, and when
  each is appropriate
- OTel delta versus cumulative temporality — the usual source of surprise when
  moving to OTLP metrics

---

## Decision 3 — Kafka runs in-cluster via Strimzi, not Amazon MSK

**Chosen:** Strimzi, the Kafka operator, running as pods on EKS.

**Rejected for now:** Amazon MSK, documented as the production-like variant but
not provisioned.

### Why

Cost, and the fact that this is a lab. MSK is roughly **$70–110/month** for the
smallest sensible provisioned cluster and runs continuously. Strimzi consumes node
capacity that Karpenter provisions on demand and can scale to zero between working
sessions.

Strimzi is also the better learning artifact: operating Kafka — brokers, topics,
partitions, consumer groups, rebalancing — teaches far more than clicking a
managed cluster into existence.

### What was given up

Managed durability, patching, and broker replacement. An in-cluster Kafka on a
demo EKS cluster is genuinely less reliable than MSK, and its storage is only as
durable as the underlying EBS volumes and PVC configuration.

### Sequencing note — this matters more than the choice itself

Kafka is deliberately scheduled **after** the log pipeline works without it, and
**after** a failure test.

The OTel Collector already has a persistent `sending_queue` with retry. That
covers "the log store is briefly unavailable," which is the failure Kafka is
nominally there to absorb. So the plan is: build the pipeline without Kafka, kill
the log store deliberately, and measure where log loss actually begins. Then add
Kafka to fix a measured problem.

The difference between *"I added Kafka because the architecture diagram had
Kafka"* and *"I measured log loss at N events and introduced Kafka to eliminate
it"* is the whole value of the exercise.

### To understand this deeply

- Kafka partitions, replication factor, retention, and consumer groups — and how
  each maps to a Strimzi custom resource
- Consumer lag as the primary health signal of a log pipeline
- The OTel Collector's `sending_queue`, `retry_on_failure`, and the
  `file_storage` extension — that is, what Kafka is competing against
- Backpressure: what a producer does when the downstream cannot keep up

---

## Decision 4 — Elasticsearch is kept; OpenSearch is not adopted

**Chosen:** keep the existing Elasticsearch 8.5.1 + Kibana deployment and its
index lifecycle policy.

**Rejected:** replacing it with OpenSearch.

### Why

The Elasticsearch stack already exists in this repository, works, and includes a
retention policy. OpenSearch is a fork of Elasticsearch and does an identical job
at this scale. Replacing it would mean deleting the EFK application, dropping
Kibana, rewriting retention from ILM to ISM, and repointing the Grafana
datasource — real work for no new capability.

### What was given up

The main argument for OpenSearch was **the AWS-managed path**: AWS sells a managed
*OpenSearch* service and does not sell managed Elasticsearch. Keeping
Elasticsearch means that if this platform ever wants a managed log store on AWS,
the migration still has to happen then.

Licensing was a secondary factor — OpenSearch is Apache-2.0, Elasticsearch is not
— which matters for commercial redistribution but not for a lab.

### A constraint that applies either way

**Do not run both.** The cluster cannot afford two log stores; each wants
gigabytes of memory and persistent volumes. Whichever is chosen, the other must be
deleted.

### To understand this deeply

- The 2021 Elasticsearch licence change and the OpenSearch fork — why the
  ecosystem split
- Index lifecycle management (ILM in Elasticsearch, ISM in OpenSearch): hot, warm,
  and delete phases
- Index templates, field mappings, and why log fields must be mapped deliberately
  rather than auto-detected
- Shards and replicas, and why single-node demo clusters usually sit yellow

---

## An honest inconsistency worth being able to explain

Decisions 1 and 2 pull in opposite directions.

For **logs**, the platform picks records up (`filelog`) — the pragmatic,
platform-owned path. For **metrics**, the application pushes (OTLP) — the
principled, application-owned path.

That is a real asymmetry in the application contract, and anyone reviewing this
architecture should notice it. The justification is that the two cases are not
symmetrical:

- The log case has a **reliability argument**: SDK-buffered logs are lost on
  crash, and crash logs are the highest-value logs.
- The metrics case has **no equivalent argument**. Losing the last few seconds of
  metrics before a crash costs almost nothing, because metrics are aggregates
  sampled over time rather than discrete events.

So the asymmetry is deliberate: each signal was decided on its own failure
characteristics rather than forced into one pattern for tidiness. Being able to
articulate that is more valuable than having been consistent.

---

## Still open

- **Structured log format.** The exact JSON field names, and whether to follow OTel
  log semantic conventions strictly or a house format.
- **Vector's dead-letter strategy.** Where malformed records go. The requirement is
  that they are never silently dropped.
- **S3 archive layout.** Prefix hierarchy and lifecycle transitions.
- **Alert thresholds.** What consumer lag or delivery-failure rate is actually
  worth waking someone for.

## Related

- `../../DEFERRED_CHANGES.md` at the workspace root — infrastructure debt and
  deferred fixes, including the Kyverno namespace-exclusion requirement that
  applies to every new observability component added here.
