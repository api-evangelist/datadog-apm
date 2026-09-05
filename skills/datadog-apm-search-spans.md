---
name: datadog-apm-search-spans
description: >-
  Search, aggregate and shape Datadog APM span data through the Datadog API v2 — span search,
  analytics rollups, span-based metric definitions, and the retention filters that decide which spans
  exist to be searched at all. Use when asked to investigate latency or errors from traces, define a
  metric from spans, or change APM indexing.
api: Datadog APM Spans API
spec: openapi/datadog-apm-spans-api-openapi.yml
operations:
  - ListSpans
  - ListSpansGet
  - AggregateSpans
  - ListSpansMetrics
  - GetSpansMetric
  - CreateSpansMetric
  - UpdateSpansMetric
  - DeleteSpansMetric
  - ListApmRetentionFilters
  - GetApmRetentionFilter
  - CreateApmRetentionFilter
  - UpdateApmRetentionFilter
  - DeleteApmRetentionFilter
  - ReorderApmRetentionFilters
  - GetServiceList
generated: '2026-09-05'
method: generated
source: >-
  openapi/datadog-apm-spans-api-openapi.yml, openapi/datadog-apm-spans-metrics-api-openapi.yml,
  openapi/datadog-apm-retention-filters-api-openapi.yml, openapi/datadog-apm-services-api-openapi.yml
---

# Searching and shaping Datadog APM spans

Every operationId below was read out of the specs named in the frontmatter. Do not invent others.

## Auth and host

`DD-API-KEY` + `DD-APPLICATION-KEY`, against the org's own Datadog site host (default
`https://api.datadoghq.com`). Read operations need the `apm_read` scope / APM Read permission.

## Search spans

- `ListSpans` — `POST /api/v2/spans/events/search`. Body carries `filter.query` (Datadog APM search
  syntax), `filter.from` / `filter.to`, `sort`, and a `page` object with `cursor` and `limit`
  (default 10, **max 1000**).
- `ListSpansGet` — `GET /api/v2/spans/events`, the same search as query parameters:
  `filter[query]`, `filter[from]`, `filter[to]`, `sort`, `page[cursor]`, `page[limit]`.

Page by following `meta.page.after` into the next request's cursor. Do not increment an offset — the
v2 spans surface is cursor-paginated, unlike the v1 SLO surface.

A returned span is `{type: "spans", id, attributes}` where `attributes` carries `service`, `env`,
`trace_id`, `span_id`, `parent_id`, `resource_name`, `start_timestamp`, `end_timestamp`, `host`,
`type`, `tags`, `custom`, `retained_by` and `ingestion_reason`. **There is no `duration` field** —
compute it from the two timestamps.

## Aggregate instead of paging

`AggregateSpans` — `POST /api/v2/spans/analytics/aggregate` takes `compute[]` (aggregation function
over a span attribute), `group_by[]` and a filter, and returns buckets as single numbers, single
strings or timeseries. Prefer this over paging thousands of spans to compute a number.

## Which services exist

`GetServiceList` — `GET /api/v2/apm/services`. **`filter[env]` is mandatory**; it became required in
client release 2.55.0 (2026-02-17). A call without it fails.

## Define a metric from spans

- `ListSpansMetrics`, `GetSpansMetric`, `CreateSpansMetric`, `UpdateSpansMetric`, `DeleteSpansMetric`
  under `/api/v2/apm/config/metrics`.
- The metric id is the caller-chosen name, so a duplicate create fails loudly rather than silently
  creating a second definition — one of the few writes here that is safe to retry.
- Deleting a span metric does **not** delete already-collected values, and recreating it does **not**
  backfill the gap.

## Retention filters — the highest-consequence write in this API

`ListApmRetentionFilters`, `GetApmRetentionFilter`, `CreateApmRetentionFilter`,
`UpdateApmRetentionFilter`, `DeleteApmRetentionFilter` and `ReorderApmRetentionFilters` under
`/api/v2/apm/config/retention-filters`.

Read this before touching them:

- A retention filter decides **which spans Datadog keeps**. Disabling, deleting or reordering one
  stops spans being indexed from that moment on.
- **This is not reversible.** Recreating the filter restores future retention only. The spans that
  went unretained during the gap are gone, and no operation brings them back. Datadog publishes no
  restoration window.
- Filters are **ordered**. `ReorderApmRetentionFilters` rewrites the entire execution order in one
  call — send the complete desired list, not a delta.
- Some filters are marked `editable: false`. Check before attempting a write.

Treat every retention-filter write as requiring explicit human confirmation, and state the data-loss
consequence in that confirmation.

## Retries and errors

- **No idempotency key exists on any of these operations.** A retried `CreateApmRetentionFilter`
  adds a duplicate filter to the evaluation order.
- The v2 spans surface returns JSON:API error objects: `{"errors":[{"status","title","detail",
  "source","meta"}]}` — richer than the v1 string array, and the field to read is `detail`.
- `429` is declared on every operation. Back off on `X-RateLimit-Reset`.

See `data-model/datadog-apm-data-model.yml`, `conventions/datadog-apm-conventions.yml` and
`errors/datadog-apm-problem-types.yml`.
