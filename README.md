PromCon 2026: Canonical Data Model
----------------------------------

Most organizations ingest metrics from more than one convention at once — a Prometheus
SDK here, OpenTelemetry there, a service mesh's own naming scheme, spans turned into RED
metrics via span-to-metrics pipelines. This repo shows a pragmatic way to unify them:
define a small canonical label schema once, and map every source into it using nothing
but stock Prometheus relabeling rules:

- `metric_relabel_configs` for sources scraped directly.
- `receive_relabel_configs` for sources that only ever arrive by push (OTLP) — an
  experimental addition to Prometheus
  ([prometheus/prometheus#19675](https://github.com/prometheus/prometheus/pull/19675),
  open and unmerged) that applies the same relabeling rules before push-ingested samples
  are stored.

Relabeling rewrites labels on series already being collected rather than materializing
new ones, so the canonical model comes with **zero additional cardinality**.

## What's here

This repo starts from [`grafana/saafe-model`](https://github.com/grafana/saafe-model) —
the OpenTelemetry Demo app, Prometheus, Grafana, the SAAFE alerting taxonomy, and the
[`promql-anomaly-detection`](https://github.com/grafana/promql-anomaly-detection)
recording-rule framework — and adds one thing on top: a canonical data model, with the
existing SAAFE alerts and anomaly-detection rules repointed to consume it.

The payoff: the same alert and recording-rule definitions now work identically no matter
which naming convention the underlying series came from — OTel spanmetrics, Envoy's admin
stats, OTel's own RPC instrumentation, or classic Prometheus client-library metrics. Each
source type is mapped with a rule generic to that *type* of source — see
`model/README.md`'s "Source types, not source instances".

See [`model/README.md`](model/README.md) for the label schema itself and the reasoning
behind each label.

## Architecture

```
OTel Demo services          ┌────────────────────────┐
(OTLP: traces, RPC ───────▶ │     OTel Collector     │
semconv metrics)            │ spanmetrics connector  │
                            └────────────┬───────────┘
                                         │ push (OTLP)
                                         ▼
Envoy admin stats  ──scrape─▶ ┌───────────────────────────────┐
Grafana metrics    ──scrape─▶ │          Prometheus           │
Prometheus metrics ──scrape─▶ │                               │
                              │    metric_relabel_configs     │
                              │    (scrape-based sources)     │
                              │                               │
                              │    receive_relabel_configs    │
                              │     (push-based sources,      │
                              │ prometheus/prometheus#19675)  │
                              └───────────────┬───────────────┘
                                              │
                                              ▼
          ┌───────────────────────────────────┴───────────────────┐
          ▼                                   ▼                   ▼
          SAAFE alerts                        anomaly detection   RED dashboard
          (rules/saafe)                       (rules/anomaly)     (per service)
```

Envoy's admin stats and the observability stack's own metrics (Grafana, Prometheus) are
scraped directly, enriched with stock `metric_relabel_configs`. Everything from the OTel
Collector (the spanmetrics connector, OTel's own RPC semantic-convention metrics, and the
`docker_stats`/`redis`/`httpcheck` receivers) arrives by OTLP push instead, the same way
the upstream OpenTelemetry Demo already ingests it. Stock Prometheus has no relabeling
stage on that path, so `receive_relabel_configs` fills the gap — see
[`model/README.md`](model/README.md)'s "Push ingestion" section for the details. Either
way, consumers query `{cdm_metric=...}` directly — no recording-rule layer in between.

The canonical model is defined formally, independent of the mapping, in
[`model/schema.json`](model/schema.json) (a [JSON Schema](https://json-schema.org/) —
which labels exist, what values they take, why). The mapping itself — how each source's
raw series gets enriched with those labels — is hand-written `metric_relabel_configs`/
`receive_relabel_configs` directly in `demo/src/prometheus/prometheus-config.yaml`; a
schema-to-rules generator was tried and dropped (see [`model/README.md`](model/README.md))
once the remaining regex patterns turned out not to be complex enough to justify it.

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
  `{cdm_metric="request_duration"}` directly — see [`model/README.md`](model/README.md)
  for why it's a label selector rather than a metric name.
- `http://localhost:8080/loadgen/` — the load generator, to drive traffic.

`make stop` (from `demo/`) tears everything down.

## Repository layout

- `demo/` — the runnable stack (OTel Demo services, Collector, Prometheus, Grafana),
  from `saafe-model`.
- `model/` — the canonical data model: the formal schema (`schema.json`) and its prose
  reference (`README.md`). No mapping logic and no recording rules live here — the
  mapping is hand-written `metric_relabel_configs`/`receive_relabel_configs` in
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
