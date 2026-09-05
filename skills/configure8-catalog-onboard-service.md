---
name: configure8-catalog-onboard-service
description: Register a new service in the configure8 catalog, link it to its repository and environments, and attach ownership metadata.
generated: '2026-09-05'
method: generated
source: openapi/configure8-c8-public-api-openapi.json
api: Configure8 REST API
base_url: https://app.configure8.io
operations:
  - CatalogEntityController_createServiceEntity
  - CatalogEntityController_createCatalogRepository
  - CatalogRelationController_createCatalogEntityRelation
  - CatalogEntityMetadataController_updateCatalogEntityMetadataById
  - CatalogEntityController_getCatalogEntityById
  - TemplateController_getAllTemplates
---

# Onboard a service into the configure8 catalog

Adds a service to the universal catalog, points it at its repository, relates it to the
environments it runs in, and stamps the metadata scorecards will later read.

## Before you start

- Authenticate every call with the `Api-Key` header. Keys begin with `c8ak` and inherit the
  permissions of the user who created them; this flow writes, so the key needs the **write**
  scope. API access is an Enterprise-plan feature.
- All calls are HTTPS against `https://app.configure8.io` and the paths below already carry
  their `/public/v1` prefix.
- **There is no idempotency mechanism on this API.** Do not blind-retry a create. If a POST
  returns 409 Conflict on a duplicate name, the previous attempt succeeded — go find the
  entity, do not create a second one.
- **There is no undo.** Nothing in this flow can be reversed by an API call; see
  `conventions/configure8-conventions.yml`.

## Steps

1. **Check the service is not already catalogued.**
   `POST /public/v1/catalog/entities` (`CatalogEntityController_getCatalogEntities`) with a
   query-builder body. The filter `name` field accepts `id`, `name`, `description`, `type`,
   `provider`, `providerResourceKey`, `providerResourceType` and `providerAccountId`.
   Note this list read is a POST, because the query builder travels in the body. Paginate
   with `pageNumber` (from 0) and `pageSize` (default 20).

2. **Pick a template, if the organization uses them.**
   `GET /public/v1/templates` (`TemplateController_getAllTemplates`). Keep the template `id`
   — passing an id that does not fit the entity type is a documented cause of 409 Conflict.

3. **Create the repository entity first, if it is not catalogued yet.**
   `POST /public/v1/catalog/entities/repository`
   (`CatalogEntityController_createCatalogRepository`, body `CreateCatalogRepositoryDto`).
   Set `providerAccountId` to the key/ID of the account that owns the repository. Keep the
   returned `id`.

4. **Create the service.**
   `POST /public/v1/catalog/entities/service` (`CatalogEntityController_createServiceEntity`,
   body `CreateCatalogServiceDto`). Set `repositoryId` to the id from step 3 and
   `templateId` to the id from step 2. Keep the returned service `id`.

5. **Relate the service to its environments.**
   `POST /public/v1/catalog/relations`
   (`CatalogRelationController_createCatalogEntityRelation`, body `CreateCatalogRelationDto`)
   with `sourceEntityId` and `targetEntityId`. For an Environment-to-Resource relation also
   set `serviceId`, which is the only case that field applies to. Resource-to-Resource
   relations are only available for resources that were **not** created by discovery.

6. **Attach metadata.**
   `PUT /public/v1/catalog/metadata/{id}`
   (`CatalogEntityMetadataController_updateCatalogEntityMetadataById`) against the service id.
   This is a replace, not a merge. On the entity PATCH endpoints, `replaceMetadataTags`
   controls whether tags are overwritten or merged.

7. **Verify.**
   `GET /public/v1/catalog/entities/{id}` (`CatalogEntityController_getCatalogEntityById`)
   and confirm the repository link, relations and metadata are present.

## Errors

`400` malformed body — validate against the DTO for the entity type.
`401` bad or missing key. `403` the key lacks the write scope, or RBAC ownership blocks the
edit. `404` an id you referenced does not exist in this organization. `409` duplicate name or
wrong template. `422` the request is well-formed but the operation is not legal in the
current state — a disallowed relation is the common case. See
`errors/configure8-problem-types.yml`.
