---
name: configure8-sync-services-safely
description: Use configure8's diff-then-apply pair to preview a bulk service sync before committing it — the only rehearsal mechanism this API offers.
generated: '2026-09-05'
method: generated
source: openapi/configure8-c8-public-api-openapi.json
api: Configure8 REST API
base_url: https://app.configure8.io
operations:
  - SyncController_getServiceDiff
  - SyncController_applyServiceDiff
  - CatalogEntityBatchController_createCatalogEntitiesBulk
  - CatalogEntityBatchController_deleteCatalogEntitiesBulk
  - CatalogEntityController_getCatalogEntities
---

# Sync services into the catalog without breaking it

Synchronising a source-of-truth system into the configure8 catalog is the highest-risk thing
this API does: it can create and remove many entities at once, nothing is idempotent, and
nothing can be undone. configure8 gives you exactly one safety net, and this skill is about
using it.

## The safety net

`POST /public/v1/sync/services/diff` (`SyncController_getServiceDiff`) returns what
`POST /public/v1/sync/services` (`SyncController_applyServiceDiff`) *would* do. It is the
only rehearse-then-commit pair in the API — there is no global dry-run flag, and the bulk
endpoints have no preview. **Always call the diff first.**

## Steps

1. **Snapshot the current state.**
   `POST /public/v1/catalog/entities`
   (`CatalogEntityController_getCatalogEntities`) and page through it with `pageNumber` /
   `pageSize`. Keep the response. This snapshot is your only rollback material — there is no
   restore endpoint.

2. **Compute the diff.**
   `POST /public/v1/sync/services/diff` (`SyncController_getServiceDiff`) with the payload
   you intend to apply.

3. **Read the diff before applying it.** Check the deletion side specifically. A source
   system that failed to return records looks identical to a source system that legitimately
   removed them, and the apply will act on the difference either way.

4. **Apply.**
   `POST /public/v1/sync/services` (`SyncController_applyServiceDiff`) with the same payload.

5. **Bulk endpoints, if you use them directly.**
   `POST /public/v1/catalog/batch/entities/resource`
   (`CatalogEntityBatchController_createCatalogEntitiesBulk`) and
   `DELETE /public/v1/catalog/batch/entities`
   (`CatalogEntityBatchController_deleteCatalogEntitiesBulk`). The documented maximum batch
   size is 1000 elements. Batch responses report `success` and `failed` counts — read the
   counts, because a batch can partially succeed while returning a 2xx.

## Cautions

- **No idempotency.** A retry after a timeout re-applies the whole payload.
- **No reversal.** Nothing here can be undone through the API.
- A cache-flush defect after batch catalog deletion was fixed in release 2.158.0; if you are
  running an older self-hosted build, verify the catalog after a bulk delete rather than
  trusting an immediate read.
