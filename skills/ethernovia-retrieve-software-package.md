---
name: Retrieve an Ethernovia software package and record the download
description: Find the software release (drivers, binaries, header-file APIs) attached to an
  Ethernovia product, resolve its file, and write the per-user download record.
api: openapi/ethernovia-customer-portal-openapi.yml
base_url: https://portal-admin.ethernovia.com/api
operations:
  - get/ec-products
  - get/ec-software-packages
  - get/ec-software-packages/{id}
  - get/ec-document-types
  - get/my-downloads
  - post/my-downloads
generated: '2026-08-04'
method: generated
source: openapi/ethernovia-customer-portal-openapi.yml
---

# Retrieve an Ethernovia software package

Ethernovia's software binaries, source-code drivers, header files and plug-in APIs are
delivered as `EcSoftwarePackage` records in the Customer Portal. Their use is governed by a
separate EULA that accompanies the download, not by the public Terms of Use — surface that
to the user before you fetch anything on their behalf.

## Before you start

- Authenticate first. Every operation requires `Authorization: Bearer <jwt>`; anonymous
  calls return `403 ForbiddenError`.
- Tokens come from the Ethernovia Auth0 tenant (`GET /connect/auth0` →
  `https://ethernovia.us.auth0.com/authorize`) or from `POST /auth/local`.

## Steps

1. **Identify the product.** Call `get/ec-products` with
   `filters[slug][$eq]=<slug>` (or `filters[name][$containsi]=<text>`) and read its
   `documentId`.
2. **List the packages for that product.** Call `get/ec-software-packages` with
   `filters[ec_products][documentId][$eq]=<productDocumentId>`,
   `populate=file,ec_document_type,ec_status` and `sort=date:desc`. The newest release is
   the first row; `revision` distinguishes releases that share a name.
3. **Read the package.** Call `get/ec-software-packages/{id}` with the package
   `documentId` and `populate=file`. The populated `file` (`UploadFile`) carries `url`,
   `ext`, `mime` and `size` — that `url` is the download target.
4. **Check what the user already has.** Call `get/my-downloads` with
   `filters[ec_software_package][documentId][$eq]=<packageDocumentId>` and
   `populate=ec_software_package,users` before creating a new record, so you do not
   duplicate the ledger entry.
5. **Record the download.** Call `post/my-downloads` with a body of
   `{"data": {"description": "<package name>", "type": "software", "revision": "<revision>",
   "ec_software_package": "<packageDocumentId>", "users": "<userDocumentId>"}}`.
   Resolve `<userDocumentId>` from `GET /users/me`.
6. **Confirm.** Re-read `get/my-downloads` filtered on the package to verify exactly one
   record exists.

## Conventions you must follow

- **Write bodies are wrapped.** Strapi expects `{"data": { ... }}` on `POST`/`PUT`; a bare
  attribute object is a `400 BadRequestError`.
- **Relations are set by identifier**, not by nested object.
- **There is no idempotency key on this API.** `post/my-downloads` is not safe to blind-retry
  — a retry after a timeout can create a second ledger row. Always re-read step 4 before
  retrying, and never retry a write automatically.
- **Use `documentId`, not `id`.**
- **Paginate.** `pagination[page]` / `pagination[pageSize]`, plus
  `pagination[withCount]=true` when you need the total.
- **Entitlement filters the catalog.** A product with no visible packages usually means the
  account's `ec_groups` do not grant that product, not that no release exists.

## Errors

| Status | `error.name` | What to do |
|---|---|---|
| 400 | BadRequestError | Missing `data` wrapper, unknown attribute, or a bad relation id. |
| 401 | UnauthorizedError | Re-authenticate. |
| 403 | ForbiddenError | No token, or the role lacks `create` on `my-download`. |
| 404 | NotFoundError | Wrong `documentId`, or the package is unpublished. |
| 500 | InternalServerError | Do **not** auto-retry the POST — re-read `get/my-downloads` first. |

Error envelope: `{"data":null,"error":{"status","name","message","details"}}`, plain
`application/json`, not RFC 9457.
