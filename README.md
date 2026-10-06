PromQL Canonical Data Model
----------------------------

## Overview

Prometheus metrics arrive from multiple sources, each using a different technology and
data model:

- Prometheus client libraries
- OpenTelemetry instrumentation
- service mesh naming conventions
- span-to-metrics pipelines deriving RED metrics from traces

Left alone, that proliferates into separate dashboards, alerts, and ad-hoc queries per
source and per service, each answering the same basic question (what's the request
rate? the error ratio? the latency?) in its own way. That proliferation prevents
building generic features as part of a centralized observability platform.

This repo addresses this problem with a **canonical data model**: a small, fixed set of
labels every source gets mapped into once, using nothing more than stock Prometheus
`metric_relabel_configs`/`receive_relabel_configs`. Alerts, dashboards, and other
generic tooling are then written against the model instead of each source's own naming
— adding a new source means mapping it into the model once, not touching anything
downstream.

Three principles guide the mapping:

- **Embrace the differences.** Accept each source's own naming convention rather than
  forcing one schema at the point of instrumentation — harmonize centrally instead.
- **Non-destructive, full fidelity.** A source's own labels and `__name__` are
  preserved, never dropped or rewritten away.
- **Zero additional cardinality.** Enrichment only adds labels to series that already
  exist; it never materializes new ones.

Built on top of [`grafana/saafe-model`](https://github.com/grafana/saafe-model) (the
OpenTelemetry Demo, Prometheus, Grafana, the SAAFE alert taxonomy, and the
anomaly-detection recording rules), this repo adds the canonical model on top and
repoints the existing SAAFE alerts and anomaly rules to consume it — the same rule
definitions, now source-agnostic.

## Quick start

```bash
cd demo
make start
```

Requires Docker. Then:

- `http://localhost:8080/grafana/` — Grafana. See the **Canonical Data Model** folder for
  the generic per-service RED dashboard, **SAAFE** for the Assertions dashboard, and
  **Node Exporter** for node-level saturation.
- `http://localhost:8080/prometheus/` — Prometheus, to query `{cdm_metric="requests_total"}` /
  `{cdm_metric="request_duration"}` directly — see "Concepts" below for why it's a
  label selector rather than a metric name.
- `http://localhost:8080/loadgen/` — the load generator, to drive traffic.

![Canonical Data Model — RED dashboard: inbound request rate/error ratio/latency with anomaly-detection bands, a request-rate-by-operation breakdown, outbound request rate/error ratio/latency broken out by destination, and a SAAFE assertions timeline, all for one selected service/operation/source](docs/sources/assets/red-dashboard.png)

`make stop` (from `demo/`) tears everything down.

## Concepts

**`__name__` is just a label.** `my_requests_total{}` and
`{__name__="my_requests_total"}` are the same series — the metric name is stored as an
ordinary label, not special syntax. That means a query can skip the name entirely and
match on labels alone, like `{cdm_metric="requests_total"}` does throughout this repo,
and a relabeling rule can match on `__name__` itself exactly like any other label.

**Enrichment, not a rename.** `__name__` is left exactly as each source emits it — a
`traces_span_metrics_calls_total` series stays named that, an
`envoy_cluster_upstream_rq_xx` series stays named that. A new label, `cdm_metric`,
carries the canonical identity instead.

## The labels

| Label | Values | Why |
|---|---|---|
| `cdm_metric` | `requests_total`, `request_duration` | The canonical identity to query by, in place of a renamed metric name. See "Enrichment, not a rename" above. |
| `cdm_source` | `spanmetrics`, `otel_semconv`, `envoy`, `grafana`, `prometheus` | Which source type produced this series — one value per relabeling section in `prometheus-config.yaml`. |
| `service` | free text | The entity this series is about. Renamed from whatever the source calls it (`service_name` for spanmetrics, a static value derived from `instance` for Envoy/Grafana). |
| `cdm_request_type` | `http`, `rpc`, `db`, `internal` | *What kind* of request. Derived from *which* semantic-convention attribute is present on the source, not from a fixed source-specific value — this is what lets the same query work across HTTP, gRPC and future sources without listing them all. |
| `cdm_request_context` | free text | *Which* request/operation — an HTTP route, an RPC method, a DB operation, falling back to whatever the source's own operation identifier is (e.g. a span name) when nothing more specific exists. |
| `cdm_direction` | `inbound`, `outbound` | Which side of a call this series represents, from OTel's `span.kind` (SERVER/CONSUMER vs. CLIENT/PRODUCER) or the source's equivalent. Without this, summing a service's inbound and outbound calls together double-counts the same logical request from both ends. |
| `cdm_status` | `ok`, `error` | Collapsed from whatever the source's own error signal is (span status, HTTP status class, response code class, ...). |
| `cdm_unit` | `ms`, `s`, ... | The unit a `request_duration` series' value is actually in. Not normalized away — relabeling can rewrite labels but never a sample's *value*, so a source reporting milliseconds can't be silently forced to look like seconds. |

The formal version of this table, independent of any one source, is
[`model/schema.json`](model/schema.json) (a [JSON Schema](https://json-schema.org/));
[`model/README.md`](model/README.md) is its full prose reference, including the known
v1 gaps and exactly how each source type gets mapped.

### How to use them

A few representative queries against the live demo (see Quick start):

```promql
# Request rate for checkoutservice's inbound calls, across every source that reports them
sum by (cdm_source) (rate({cdm_metric="requests_total", service="checkoutservice", cdm_direction="inbound"}[5m]))

# Error ratio, grouped by source to avoid double-counting the same call reported twice
# (see "Lessons learned" below)
sum by (service, cdm_source) (rate({cdm_metric="requests_total", cdm_status="error", cdm_direction="inbound"}[5m]))
/
sum by (service, cdm_source) (rate({cdm_metric="requests_total", cdm_direction="inbound"}[5m]))

# p95 latency, accounting for cdm_unit explicitly rather than mixing differently-scaled
# buckets into one histogram_quantile() call
histogram_quantile(0.95, sum by (le) (rate({cdm_metric="request_duration", le=~".+", cdm_unit="ms", service="checkoutservice"}[5m]))) / 1000
```

## Lessons learned

Relabeling is a sharp tool. A few things worth knowing before applying this pattern to a
real deployment, beyond what's in this demo:

- **Sources change versions out from under you.** The OTel HTTP semantic conventions
  alone moved from `http.server.duration` (ms) to `http.server.request.duration` (s)
  between v1.20.0 and the stable v1.23.1 — and different services upgrade at different
  times, so both can be live simultaneously. Map each version's metric name and
  attributes separately into the same `cdm_metric`, and let `cdm_unit` carry the
  difference rather than assuming a single source-wide convention.
- **A relabeled label set can break query matching silently.** A binary operation
  (`errors / requests`) between two series with different label sets drops non-matching
  series with no error, not a query failure — pin down the matching explicitly with
  `on(...)`/`ignoring(...)` rather than relying on both sides happening to match.
- **Choose output labels explicitly with `by(...)`.** `without(...)` keeps whatever new
  labels relabeling just added, which can silently split aggregation groups and change an
  alert's identity when the mapping changes.
- **New label values mean new series.** A relabeling rollout that changes a label's value
  creates new series, which costs extra memory and can produce a `rate()` gap right at
  the rollout boundary — stage it and give the TSDB headroom.
- **One oversized sample fails the whole scrape.** Exceeding `label_limit` on a single
  series fails ingestion for every series in that scrape, not just the offending one —
  map only the metric families you actually need rather than everything a source exposes.

## Architecture

![Architecture: OTel Demo services and Envoy/Grafana/Prometheus metrics flow into Prometheus via push (OTLP) and scrape respectively, enriched by metric_relabel_configs and receive_relabel_configs, then consumed by the generic dashboard, anomaly detection, and SAAFE insights](docs/sources/assets/architecture.png)

Envoy's admin stats and the observability stack's own metrics (Grafana, Prometheus) are
scraped directly, enriched with stock `metric_relabel_configs`. Everything from the OTel
Collector (the spanmetrics connector, OTel's own RPC semantic-convention metrics, and the
`docker_stats`/`redis`/`httpcheck` receivers) arrives by OTLP push instead, the same way
the upstream OpenTelemetry Demo already ingests it. Stock Prometheus has no relabeling
stage on that path, so `receive_relabel_configs` fills the gap — an experimental addition
to Prometheus ([prometheus/prometheus#19675](https://github.com/prometheus/prometheus/pull/19675),
open and unmerged) that applies the exact same relabeling rules to push-ingested samples
before they're stored; `demo/src/prometheus/Dockerfile.pr19675` builds Prometheus from
the PR's branch to run it. Either way, consumers query `{cdm_metric=...}` directly — no
recording-rule layer in between.

The mapping itself — how each source's raw series gets enriched with the labels in
[`model/schema.json`](model/schema.json) — is hand-written `metric_relabel_configs`/
`receive_relabel_configs` directly in `demo/src/prometheus/prometheus-config.yaml`; a
schema-to-rules generator was tried and dropped (see [`model/README.md`](model/README.md))
once the remaining regex patterns turned out not to be complex enough to justify it.

## Repository layout

- `demo/` — the runnable stack (OTel Demo services, Collector, Prometheus, Grafana),
  from `saafe-model`.
- `model/` — the canonical data model: the formal schema (`schema.json`) and its prose
  reference (`README.md`), including the known v1 gaps and how each source type is
  mapped. No mapping logic and no recording rules live here — the mapping is hand-written
  `metric_relabel_configs`/`receive_relabel_configs` in
  `demo/src/prometheus/prometheus-config.yaml`, and consumers query
  `{cdm_metric=...}` inline.
- `rules/saafe/`, `rules/anomaly/` — the SAAFE alert taxonomy and the
  `promql-anomaly-detection` recording rules, from `saafe-model`, repointed at the
  canonical model.
- `docs/` — supporting material carried over from `saafe-model`.

## Attribution

Built on [`grafana/saafe-model`](https://github.com/grafana/saafe-model) (Apache-2.0),
which itself builds on the
[OpenTelemetry Demo](https://github.com/open-telemetry/opentelemetry-demo) (Apache-2.0).
The anomaly-detection recording rules are imported from
[`grafana/promql-anomaly-detection`](https://github.com/grafana/promql-anomaly-detection),
presented at PromCon 2024. SAAFE was presented at PromCon 2025 as *A Prioritized Alerting
Model to Troubleshoot Your Incidents*. See `AUTHORS` and `LICENSE`.
