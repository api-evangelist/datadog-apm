---
name: datadog-apm-manage-slos
description: >-
  Create, inspect, correct and safely delete Datadog Service Level Objectives through the Datadog API
  v1. Use when asked to add an SLO, check whether an SLO can be deleted, read SLI history, or exclude
  a maintenance window from an error budget.
api: Datadog APM Service Level Objectives API
spec: openapi/datadog-apm-slos-api-openapi.yml
operations:
  - ListSLOs
  - SearchSLO
  - GetSLO
  - CreateSLO
  - UpdateSLO
  - CheckCanDeleteSLO
  - DeleteSLO
  - GetSLOHistory
  - GetSLOCorrections
  - CreateSLOCorrection
  - UpdateSLOCorrection
  - DeleteSLOCorrection
generated: '2026-09-05'
method: generated
source: openapi/datadog-apm-slos-api-openapi.yml, openapi/datadog-apm-slo-corrections-api-openapi.yml
---

# Managing Datadog SLOs

Every operationId below was read out of the specs named in the frontmatter. Do not invent others.

## Before the first call

- Send **both** headers: `DD-API-KEY` and `DD-APPLICATION-KEY`. A missing application key is a `403`,
  not a `401`.
- Pick the host for the org's **Datadog site**. The spec's `servers[]` is templated
  (`https://{subdomain}.{site}`); the default resolution is `https://api.datadoghq.com`. A key used
  against the wrong site host fails with `403` or `404` and never redirects.
- The application key's role needs `slos_read` to read and `slos_write` to change anything.

## Find an SLO

- `SearchSLO` — `GET /api/v1/slo/search`, paginated with `page[size]` and `page[number]`.
- `ListSLOs` — `GET /api/v1/slo`, paginated with `offset` and `limit`, and filterable by `ids`,
  `query`, `tags_query`, `metrics_query`.

These two page **differently**. Do not carry a cursor from one to the other.

## Read the objective and its history

- `GetSLO` — `GET /api/v1/slo/{slo_id}`; pass `with_configured_alert_ids=true` to see which monitors
  reference it.
- `GetSLOHistory` — `GET /api/v1/slo/{slo_id}/history` with `from_ts` and `to_ts` (epoch seconds).
  `apply_correction` decides whether corrections are folded into the returned SLI.

## Create and update

- `CreateSLO` — `POST /api/v1/slo`. **There is no idempotency key.** If the call times out, do NOT
  blindly retry: call `SearchSLO` for the name first, or you will create a second SLO.
- `UpdateSLO` — `PUT /api/v1/slo/{slo_id}` is a full-document replace. Read with `GetSLO`, modify,
  write back the whole object.

## Delete safely

1. `CheckCanDeleteSLO` — `GET /api/v1/slo/can_delete?ids=...`. This is the only dry run in the
   surface. It returns the same `409` conflict payload the delete would, and changes nothing.
2. Only if it comes back clean, `DeleteSLO` — `DELETE /api/v1/slo/{slo_id}`.

`DeleteSLO` accepts a `force` parameter. Treat `force` as an escalation that needs explicit human
confirmation: it deletes an SLO that dashboards or monitors still reference.

**There is no restore.** Datadog publishes no undelete operation and no restoration window for SLOs.
A deleted SLO's history does not come back when an identically-configured SLO is recreated. Say so
before deleting.

## Exclude a maintenance window instead of deleting

A correction is the reversible alternative to changing or deleting an SLO.

- `GetSLOCorrections` — `GET /api/v1/slo/{slo_id}/corrections` lists what already applies.
- `CreateSLOCorrection` — `POST /api/v1/slo/correction` with `category`, `start`, `end` (or
  `duration`), `timezone`, and optionally an `rrule` for a recurring window.
- `DeleteSLOCorrection` — removes the correction and restores the uncorrected SLI. This is a genuine
  undo, unlike an SLO delete.

`CreateSLOCorrection` has no idempotency key either — a retried create applies the correction twice.

## Errors

- `400` bad request, `403` forbidden / missing permission, `404` unknown `slo_id`,
  `409` conflict on delete, `429` rate limited.
- The v1 body is `{"errors": ["..."]}` — plain strings, no machine-readable code.
- On `429`, read `X-RateLimit-Reset` (seconds until reset) and back off. `X-RateLimit-Name` names the
  bucket to quote if the org needs a limit raise.

See `errors/datadog-apm-problem-types.yml`, `conventions/datadog-apm-conventions.yml` and
`rate-limits/datadog-apm-rate-limits.yml`.
