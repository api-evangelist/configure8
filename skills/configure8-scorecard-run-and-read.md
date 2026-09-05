---
name: configure8-scorecard-run-and-read
description: Trigger a configure8 scorecard re-evaluation and read the metric definitions and per-service results.
generated: '2026-09-05'
method: generated
source: openapi/configure8-c8-public-api-openapi.json
api: Configure8 REST API
base_url: https://app.configure8.io
operations:
  - ScorecardController_getScorecards
  - ScorecardController_getScorecardDefinitionById
  - ScorecardController_createScorecardSchedule
  - ScorecardController_getMetricsByScorecardId
  - ScorecardController_getScorecardMetricResults
  - ScorecardController_updateScorecardById
---

# Run a scorecard and read its results

Drives configure8's production-readiness scorecards from CI or from an agent: find the
scorecard, trigger a re-evaluation, then read the metrics and the per-service results.

## Before you start

- `Api-Key` header on every call. Reading needs only the default **read** scope; steps 3 and
  5 write and need **write**. RBAC applies: a user can only edit scorecards they own.
- Triggering a run is not free — it re-evaluates every service in scope. There is no
  idempotency key, so a retried trigger schedules a second run.

## Steps

1. **List the scorecards.**
   `GET /public/v1/scorecards` (`ScorecardController_getScorecards`). Filter down to the one
   you want and keep its `id`.

2. **Read the definition.**
   `GET /public/v1/scorecards/{id}` (`ScorecardController_getScorecardDefinitionById`) to see
   the checks and levels before you act on any score.

3. **Trigger a re-evaluation.**
   `POST /public/v1/scorecards/{id}/run` (`ScorecardController_createScorecardSchedule`).
   The response is a `ScheduleDto`. This is asynchronous — results are not ready when the
   call returns.

4. **Read the metrics.**
   `GET /public/v1/scorecards/{id}/metrics` (`ScorecardController_getMetricsByScorecardId`)
   returns `ScorecardMetricDto` records, each with a `measurementId`.

5. **Read the results.**
   `GET /public/v1/scorecards/{id}/results`
   (`ScorecardController_getScorecardMetricResults`) returns `ScorecardMetricResultDto`
   records keyed by `scorecardId`, `metricId` and `serviceId`. Poll here after step 3 rather
   than assuming the run has finished. Paginate with `pageNumber` / `pageSize`.

6. **Update the definition only if you must.**
   `PUT /public/v1/scorecards/{id}` (`ScorecardController_updateScorecardById`) is a full
   replace, not a patch. Read the definition in step 2 first and send it back whole, or you
   will drop checks. There is no version history and no undo.

## Errors

`403` means the key's user is not an owner of this scorecard. `404` means the scorecard id
does not exist in this organization. Aggregation and sorting defects on the results history
were fixed in release 2.157.0 — see `changelog/configure8-changelog.yml`.
