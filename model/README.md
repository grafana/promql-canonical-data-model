# The Canonical Data Model

A small set of labels that every metric source in this demo gets enriched with, via
plain Prometheus `metric_relabel_configs` — no new series, no new ingestion pipeline,
just extra labels added to samples you already scrape. Every raw label the source
originally had (`service_name`, `span_kind`, `envoy_cluster_name`, `otel_scope_name`,
...) stays right where it was — nothing here ever drops or overwrites the original
data, only adds to it. See the root [README](../README.md) for the overall architecture
and why relabeling (not an OTel-side transform) is the mechanism.

## `cdm_metric`: enrichment, not a rename

`__name__` is left exactly as each source emits it — a `traces_span_metrics_calls_total`
series stays named that; an `envoy_cluster_upstream_rq_xx` series stays named that.
Canonical identity is carried by a new label, `cdm_metric`, instead:

| `cdm_metric` value | Meaning |
|---|---|
| `requests_total` | One request/call, however the source defines "a call" (a completed span, a proxied HTTP request, ...) |
| `request_duration` | How long a request took |

Query it with a name-less selector — `{cdm_metric="requests_total", service=~"..."}`.
This was a deliberate choice over renaming
`__name__` to something like `cdm_requests_total`: renaming is a destructive
transformation (the source's own metric name is gone, unrecoverable, and two sources
could collide into an identical labelset), whereas tagging `cdm_metric` is purely
additive. The cost is that querying no longer looks like "pick a metric name" — it
looks like "pick a label" — which is why this file exists: nobody discovers
`{cdm_metric="requests_total"}` by browsing a metric list, they need to know the
convention.

Duration has no unit baked into its label value on purpose: relabeling can rewrite
labels but never a sample's *value*, so a source reporting milliseconds can't be
silently forced to look like seconds. Instead every source tags its own unit via
`cdm_unit` (`ms` for both spanmetrics and Envoy in this demo). Because bucket, sum and
count series all share the same `cdm_metric="request_duration"` value, a query that
wants only the histogram buckets filters on `le=~".+"` (only bucket series carry `le`)
rather than on the metric name — see `rules/saafe/anomaly.yml` for a live example.

There's no separate recording-rule layer computing rates/ratios/saturation on top of
this — that would materialize new stored series, which is exactly the extra
cardinality this model exists to avoid. Consumers (SAAFE alerts, the anomaly-detection
framework, dashboards) query `{cdm_metric=...}` directly with inline
`rate()`/`histogram_quantile()`, same as they'd query any other Prometheus metric.

## Labels

The formal version of this table is [`schema.json`](schema.json) — a JSON Schema,
independent of how any source maps into it. This section is its prose form.

`service` and `env` are deliberately *not* `cdm_`-prefixed, unlike everything else here.
The prefix's job is to mark labels this model *invents* — new classification vocabulary
a source doesn't natively have (`cdm_request_type`, `cdm_direction`, ...). `service` and
`env` aren't inventions; they're near-universal ecosystem conventions already (OTel's
`service.name`/`deployment.environment.name`, Grafana/Loki/Tempo cross-signal
correlation, most APM tooling) that already expect a bare label — prefixing them would
opt this model *out* of that convention for no benefit.

| Label | Values | Why |
|---|---|---|
| `cdm_metric` | `requests_total`, `request_duration` | The canonical identity to query by, in place of a renamed metric name. See above. |
| `cdm_source` | `spanmetrics`, `otel_semconv`, `envoy`, `node_exporter`, `grafana` | Which source type produced this series — one value per relabeling section in `prometheus-config.yaml`. Every section adds it, including `node_exporter` (which sets no `cdm_metric`). Makes the source type queryable/selectable instead of only living in a comment. |
| `service` | free text | The entity this series is about. Renamed from whatever the source calls it (`service_name` for spanmetrics, a static value for Envoy, the scrape `instance` for node_exporter). |
| `env` | free text | Deployment environment, from `deployment.environment.name` (or the older `deployment.environment`). Not populated in this demo (single environment) but included so the schema is complete for real deployments. |
| `cdm_entity_type` | `service`, `node` | Which level a series describes — a service dashboard and a node/host dashboard need different label sets, this is what tells you which one you're looking at. |
| `cdm_request_type` | `http`, `rpc`, `db`, `internal` | *What kind* of request. Derived from *which* semantic-convention attribute is present on the source, not from a fixed source-specific value — this is what lets the same query work across HTTP, gRPC and future sources without listing them all. |
| `cdm_request_context` | free text | *Which* request/operation — an HTTP route, an RPC method, a DB operation, falling back to whatever the source's own operation identifier is (e.g. a span name) when nothing more specific exists. |
| `cdm_direction` | `inbound`, `outbound` | Which side of a call this series represents, from OTel's `span.kind` (SERVER/CONSUMER vs. CLIENT/PRODUCER) or the source's equivalent. Without this, summing a service's inbound and outbound calls together double-counts the same logical request from both ends. |
| `cdm_status` | `ok`, `error` | Collapsed from whatever the source's own error signal is (span status, HTTP status class, response code class, ...). |
| `cdm_unit` | `ms`, `s`, ... | The unit a `request_duration` series' value is actually in. Not normalized away — see above. |

Saturation doesn't get a `cdm_metric` value in v1: node_exporter is the only node-level
source in this demo, so there's nothing to unify across sources for it (see "Known v1
gaps"). It's tagged `cdm_entity_type: node` for dashboard classification only.

## Why these labels and not more

This is deliberately a small, OTel-semantic-convention-first set (see the root README's
talk context). It does **not** yet include things like entity-relationship/service-graph
edges or rootcause/entity-graph labels — those are real, useful ideas, but adding them
before the OTel-native layer is solid would be designing in the wrong order. They're
natural extensions once you need them for your own organization, not because the demo
needs to prove it can do everything at once.

## Schema vs. mapping — two separate things on purpose

[`schema.json`](schema.json) is the *formal* definition of the canonical model:
which labels exist, what values they take, why — as a plain
[JSON Schema](https://json-schema.org/), independent of any particular source. It
doesn't know spanmetrics or Envoy exist. It's the machine-readable counterpart to the
"Labels" table above (in fact it's the source of truth that table is written from) —
useful as documentation, and as something a validator or another tool could check
label values against, without pulling in anything about how the mapping happens.

The *mapping* — how each source's raw series gets enriched with these labels — is
hand-written `metric_relabel_configs` directly in
[`demo/src/prometheus/prometheus-config.yaml`](../demo/src/prometheus/prometheus-config.yaml).
There's no generator or template standing between the schema and the rules: Prometheus
has no native way to `include` a separate relabeling file, and a source-mapping DSL
compiling down to `metric_relabel_configs` was tried and dropped here — the regex
patterns involved (mostly "gate on a guard label, then override a default with
progressively more specific matches") aren't complex enough to justify a code-generation
step and the Go toolchain dependency it required. Adding a new source means writing its
relabel rules directly in `prometheus-config.yaml`, following the pattern of the
existing ones (each section is commented) and enriching with the labels `schema.json`
defines.

## Source types, not source instances

Every rule in `prometheus-config.yaml` is written to be generic for a *type* of source,
not a fact about this one demo's topology. A rule that hardcodes something only true of
this specific deployment (a literal service name, say) isn't mapping a source type —
it's papering over one instance of it. Each type also tags itself with `cdm_source`,
so which relabeling section produced a series is queryable/selectable directly, not
just documented in a comment. Five types are mapped:

- **OTel spanmetrics** (traces → RED metrics via the connector) — HTTP/RPC/DB, keyed on
  `service_name` being present.
- **node_exporter** — host saturation, single source in this demo (see "Known v1 gaps").
- **Envoy admin stats** — a service-mesh-adjacent naming convention. `service` comes
  from the scrape target's own `instance`, not a hardcoded name, so the rule works for
  any Envoy deployment scraped this way, not just this demo's one proxy.
- **OTel RPC semantic-convention metrics** (`cdm_source="otel_semconv"`) —
  `rpc_client_duration_milliseconds` / `rpc_server_duration_milliseconds`, emitted
  *directly* by RPC client/server instrumentation libraries, not derived from spans. A
  genuinely different source from spanmetrics sharing the same scrape endpoint,
  confirmed present from both a Java and a Go SDK in this demo, mapped with the exact
  same `cdm_*` labels — the point of defining the schema independent of any one source.
- **Grafana's own HTTP metrics** (`grafana_http_request_duration_seconds`) — classic
  Prometheus client-library instrumentation, not OTel-derived at all: no resource
  attributes, no `service_name` of its own. `service` comes from `instance`, same
  technique as Envoy. The observability stack's own tooling is a source type like any
  other, not a special case — and it's the one native-seconds source in this demo,
  where every other source reports `ms`, a live example of why `cdm_unit` exists.

## Push ingestion (experimental)

This demo scrapes rather than pushes because stock Prometheus only ever runs relabeling
from the scrape loop — push ingestion (remote-write, OTLP) has no relabeling stage at all.
[prometheus/prometheus#19675](https://github.com/prometheus/prometheus/pull/19675),
open and unmerged, adds `receive_relabel_configs` to close that gap: relabeling applied to
remote-write/OTLP-ingested samples before storage. `prometheus-config.yaml`'s
`receive_relabel_configs` block and `otelcol-config.yml`'s `metrics/push` pipeline
validate it against this demo's own spanmetrics rules — same rules, same output as the
scrape path, confirmed live, only *where* they run changes.

## Known v1 gaps

- **HTTP semantic-convention metrics aren't mapped.** Unlike RPC, this one's a genuine
  mess: this demo alone has three different metric names for "how long did the server
  take" from different SDKs/semconv versions —
  `http_server_duration_milliseconds` (old semconv, ms), `http_server_duration_seconds`
  (transitional, seconds despite the older attribute names alongside it),
  `http_server_request_duration_seconds` (current stable semconv, seconds, new
  attribute names like `http_request_method`/`http_route`). A single generic rule
  across all three would be forcing elegance where the ecosystem itself hasn't
  converged yet — left unmapped rather than papered over. This is the talk's "changes
  over time / different versions" lesson, live.
- **Cross-unit aggregation is a per-query responsibility.** `cdm_unit` tells you what
  unit a series is in; relabeling can't convert it (rewriting a bucket boundary's
  numeric meaning needs arithmetic, not a label rewrite). Consumers that aggregate
  across sources normalize explicitly at query time instead — the Canonical Data Model
  dashboard's latency panels compute the quantile per `cdm_unit`, divide the
  millisecond branch by 1000, and `or` the branches together, rather than mixing
  differently-scaled `le` buckets into one `histogram_quantile()` call (which would
  silently produce nonsense). A query that skips this and groups by `le` alone across
  multiple `cdm_unit` values is a bug, not a feature.
- **Saturation isn't unified across sources.** There's only node_exporter in this demo,
  so `rules/saafe/saturation.yml` and the anomaly framework's resource rules query
  node_exporter's own metric names directly rather than through `cdm_metric` — nothing
  to gain from indirecting through a label when there's only one source.
- **redis and docker_stats sources aren't mapped yet.** They already flow through the
  same collector endpoint Prometheus scrapes; adding them is "more of the same" rather
  than new design work.
- **No service-graph/entity-relationship modeling.** An open question from the talk
  outline (see the Tempo service-graph PR referenced there) — not attempted here.
