PromCon 2026: Canonical Data Model
----------------------------------

## Why this exists

Most organizations run more than one way of describing the same thing: a Prometheus SDK
here, OpenTelemetry there, a service mesh's own naming scheme, spans turned into RED
metrics by a span-to-metrics pipeline. Each convention is reasonable on its own — the
problem shows up a layer up, where dashboards, alerting, anomaly detection, and SLOs all
want to ask the same questions (what's the request rate? the error ratio? the latency?)
and get a different answer depending on which naming convention the underlying metric
happens to use. Multiply that by every service and every source, and the generic
capabilities on top either get rebuilt per source, or never get built at all.

A **canonical data model** is the fix: a small, fixed set of labels that every source
gets mapped into once, so dashboards, alerting, anomaly detection, and
[SAAFE](https://github.com/grafana/saafe-model) insights only need to be built once too
— against the model, not against each source.

This repo is the third installment of a multi-year series exploring how far stock
Prometheus's own rule engines can be pushed to solve exactly this kind of problem,
without standing up a new system:

| PromCon | What it pushed | Rule engine |
|---|---|---|
| 2024 — [Anomaly Detection](https://github.com/grafana/promql-anomaly-detection) | Statistical baselines, in PromQL | Recording rules |
| 2025 — [SAAFE](https://github.com/grafana/saafe-model) | An opinionated alert taxonomy | Alerting rules |
| 2026 — Canonical Data Model (this repo) | A shared label schema across sources | **Relabeling rules** |

The insight that makes this one work: Prometheus stores a series' metric name as nothing
more than its `__name__` label internally — `my_requests_total{}` and
`{__name__="my_requests_total"}` are the same series. Because of that, relabeling rules
can match and enrich *any* series by its existing labels, `__name__` included, with no
new ingestion path and no new system to run. And because relabeling rewrites labels on
series that already exist rather than creating new ones, it comes for free,
cardinality-wise. Three properties fall out of taking that seriously:

- **Embrace the differences.** Accept each source's own naming convention rather than
  forcing one schema at the point of instrumentation — harmonize centrally instead.
- **Non-destructive, full fidelity.** The source's own labels and `__name__` are
  preserved, never dropped or rewritten away.
- **Zero additional cardinality.** Enrichment only adds labels to series that already
  exist; it never materializes new ones.

One gap surfaced while building this: stock Prometheus only ever runs relabeling from
the scrape loop, so anything arriving by push (remote-write, OTLP) — which is how most
modern pipelines, including the upstream OpenTelemetry Demo, actually ingest — gets none
of this. The "Architecture" section below covers the experimental patch this repo carries
to close that gap.

See [`model/README.md`](model/README.md) for how these principles turn into the actual
label schema, and the reasoning behind each label.

## What's here

This repo starts from [`grafana/saafe-model`](https://github.com/grafana/saafe-model) —
the OpenTelemetry Demo app, Prometheus, Grafana, the SAAFE alert taxonomy, and the
anomaly-detection recording rules — and adds the canonical data model on top, repointing
the existing SAAFE alerts and anomaly-detection rules to consume it.

The payoff: the same alert and recording-rule definitions now work identically no matter
which naming convention the underlying series came from — OTel spanmetrics, Envoy's admin
stats, OTel's own RPC instrumentation, or classic Prometheus client-library metrics. Each
source type is mapped with a rule generic to that *type* of source — see
`model/README.md`'s "Source types, not source instances".

See [`model/README.md`](model/README.md) for the label schema itself and the reasoning
behind each label.

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
before they're stored. See [`model/README.md`](model/README.md)'s "Push ingestion"
section for the details. Either way, consumers query `{cdm_metric=...}` directly — no
recording-rule layer in between.

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

![Canonical Data Model — RED dashboard: inbound request rate/error ratio/latency with anomaly-detection bands, a request-rate-by-operation breakdown, outbound request rate/error ratio/latency broken out by destination, and a SAAFE assertions timeline, all for one selected service/operation/source](docs/sources/assets/red-dashboard.png)

`make stop` (from `demo/`) tears everything down.

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
