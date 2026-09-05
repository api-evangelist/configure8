---
name: configure8-record-deployment
description: Record a deployment against a repository and environment in configure8 from a CI pipeline, then amend or remove it.
generated: '2026-09-05'
method: generated
source: openapi/configure8-c8-public-api-openapi.json
api: Configure8 REST API
base_url: https://app.configure8.io
operations:
  - DeploymentController_create
  - DeploymentController_getDeployments
  - DeploymentController_update
  - DeploymentController_delete
---

# Record a deployment in configure8

Feeds the Deployment API from a CI pipeline so the catalog shows what shipped where.

## Before you start

- `Api-Key` header, **write** scope. The Deployment API is Enterprise-only — the pricing
  table's "Deployment API" row reads "No" for the Free plan.
- Resolve `repositoryId` (and the environment) from the catalog first; a deployment is
  attached to entities, not to free text.

## Steps

1. **Resolve the repository entity.**
   `POST /public/v1/catalog/entities` with a query-builder filter on `name` or
   `providerResourceKey`, or use an id your pipeline already holds.

2. **Create the deployment.**
   `POST /public/v1/deployments` (`DeploymentController_create`, body `CreateDeploymentDto`)
   with `repositoryId`. Keep the returned id — you cannot look a deployment up by an
   external build id.

3. **List deployments to confirm.**
   `GET /public/v1/deployments` (`DeploymentController_getDeployments`), paginated with
   `pageNumber` / `pageSize`.

4. **Amend a deployment.**
   `PATCH /public/v1/deployments/{id}` (`DeploymentController_update`) — for example to move
   it from in-progress to succeeded or failed.

5. **Remove a deployment only if it was recorded in error.**
   `DELETE /public/v1/deployments/{id}` (`DeploymentController_delete`). **This cannot be
   undone.** There is no restore endpoint and no retention window; if you may need the
   record, GET it and keep the response before deleting.

## Errors

A 500 on step 2 is genuinely ambiguous: with no idempotency key you cannot tell whether the
deployment was written. List with step 3 before retrying, or you will create a duplicate.
