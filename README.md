PromCon 2026: Canonical Data Model
----------------------------------

Accompanying material for the PromCon 2026 talk: *Bring Order to Metrics Chaos: Building
a Canonical Data Model with Prometheus Relabeling*.

Most organizations ingest metrics from more than one convention at once — a Prometheus
SDK here, OpenTelemetry there, a service mesh's own naming scheme, spans turned into
RED metrics via span-to-metrics pipelines. This repo shows a pragmatic way to unify them:
define a small canonical label schema once, and map every source into it using nothing
but stock **Prometheus relabeling rules** — `metric_relabel_configs`. Because relabeling
rewrites labels on series you already scrape rather than materializing new ones, the
canonical model comes with **zero additional cardinality**.

## What's here

Starting from [`grafana/saafe-model`](https://github.com/grafana/saafe-model) — the
OpenTelemetry Demo app, Prometheus, Grafana, and the SAAFE alerting taxonomy plus the
[`promql-anomaly-detection`](https://github.com/grafana/promql-anomaly-detection)
recording-rule framework already layered on top of it — this repo adds one thing: a
canonical data model, and repoints the existing SAAFE alerts and anomaly-detection
recording rules to consume it. The result: the *same* alert and recording rule
definitions now work identically whether the underlying series came from OTel
spanmetrics, Envoy's admin stats, or OTel's own RPC client/server instrumentation —
three genuinely different naming conventions, one query surface. Each is mapped as a
generic rule for that *type* of source, not a fact hardcoded about this one demo's
topology (see `model/README.md`'s "Source types, not source instances").

See [`model/README.md`](model/README.md) for the label schema itself and the reasoning
behind each label.

## Architecture

```
                       ┌───────────────────────────┐
  OTel Demo services   │      OTel Collector        │
  (OTLP: traces,    ──▶│  spanmetrics connector     │──┐
  RPC semconv metrics) │  docker_stats / redis /     │  │  scrape (:9464,
                       │  httpcheck receivers        │  │  Prometheus exposition)
                       └───────────────────────────┘  │
                                                        ▼
  node_exporter ─────────────────────scrape──────▶ ┌─────────────┐
  Envoy admin stats ─────────────────scrape──────▶ │  Prometheus │
                                                    │             │
                                                    │ metric_     │  ◀── canonical
                                                    │ relabel_    │      mapping lives
                                                    │ configs     │      here, per source
                                                    └──────┬──────┘
                                                           │ __name__ untouched, enriched
                                                           │ with cdm_metric="requests_total"
                                                           │ / "request_duration" + cdm_* labels
                                                           ▼
                                    ┌──────────────────────┼───────────────────────┐
                                    ▼                      ▼                       ▼
                           SAAFE alerts (rules/saafe)   anomaly detection      generic RED
                           query {cdm_metric=...}       (rules/anomaly)        dashboard,
                           inline, no recording-rule                          per service
                           layer in between
```

Metrics reach Prometheus by **scrape**, not push, on purpose: Prometheus's OTLP and
remote-write receive paths have no relabeling stage at all (confirmed against
Prometheus's own source — see the talk for details), so relabeling only exists as a
scrape-time thing. The collector keeps centralizing ingestion exactly as `saafe-model`
already did; it just exposes a `prometheus`-format scrape endpoint instead of pushing.

The canonical model is defined formally, independent of the mapping, in
[`model/schema.json`](model/schema.json) (a [JSON Schema](https://json-schema.org/) —
which labels exist, what values they take, why). The mapping itself — how each source's
raw series gets enriched with those labels — is hand-written `metric_relabel_configs`
directly in `demo/src/prometheus/prometheus-config.yaml`; a schema-to-rules generator
was tried and dropped (see [`model/README.md`](model/README.md)) once the remaining
regex patterns turned out not to be complex enough to justify it.

## Quick start

```bash
cd demo
make start
```

Requires Docker. Then:

- `http://localhost:8080/grafana/` — Grafana. See the **Canonical Data Model** folder for
  the generic per-service RED dashboard, **SAAFE** for the Assertions dashboard, and
  **Node Exporter** for node-level saturation (a different entity type — see
  `model/README.md`).
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
  mapping is hand-written `metric_relabel_configs` in
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
