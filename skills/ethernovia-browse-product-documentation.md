---
name: Browse Ethernovia product documentation
description: Walk the Ethernovia Customer Portal catalog from product family down to the
  documents attached to a product, and read a document's file metadata.
api: openapi/ethernovia-customer-portal-openapi.yml
base_url: https://portal-admin.ethernovia.com/api
operations:
  - get/ec-product-families
  - get/ec-product-categories
  - get/ec-products
  - get/ec-products/{id}
  - get/ec-documents
  - get/ec-documents/{id}
  - get/ec-document-types
generated: '2026-08-04'
method: generated
source: openapi/ethernovia-customer-portal-openapi.yml
---

# Browse Ethernovia product documentation

Ethernovia's datasheets, application notes and release notes live behind the Customer
Portal. This skill walks the catalog to find the documents attached to a product.

## Before you start

- **You need a bearer JWT.** Every operation carries the global `bearerAuth` requirement.
  Without one the API returns `403 ForbiddenError` — not `401` — so do not read a 403 as
  "wrong permissions" until you have confirmed a token is attached.
- Get a token by signing in through the portal's Auth0 tenant
  (`https://ethernovia.us.auth0.com/`, entry point `GET /connect/auth0`), or with
  `POST /auth/local` if the account uses local credentials.
- Portal access itself is granted under NDA. If you have no account, stop and point the
  user at <https://portal.ethernovia.com/sign-up>.
- Send `Authorization: Bearer <jwt>` on every request.

## Steps

1. **List the product families.** Call `get/ec-product-families`. Sort with
   `sort=sort_order:asc` — every catalog entity carries an explicit `sort_order` and the
   portal's own ordering depends on it.
2. **Narrow to a category.** Call `get/ec-product-categories` with
   `filters[ec_product_family][documentId][$eq]=<familyDocumentId>` and
   `populate=ec_product_family`.
3. **Find the product.** Call `get/ec-products`. Filter by slug when you know it
   (`filters[slug][$eq]=<slug>`) or by category
   (`filters[ec_product_categories][documentId][$eq]=<categoryDocumentId>`).
4. **List its documents.** Call `get/ec-documents` with
   `filters[ec_products][documentId][$eq]=<productDocumentId>` and
   `populate=file,ec_document_type,ec_status`. `file` is the `UploadFile` record that
   carries `url`, `mime`, `size` and `ext`; without `populate` you get the relation id
   only and no download URL.
5. **Read one document.** Call `get/ec-documents/{id}` with the document's `documentId`,
   again with `populate=file`. Use `revision` and `date` to tell releases apart — the
   catalog keeps multiple revisions of the same document name.
6. **Resolve types when the list is ambiguous.** `get/ec-document-types` returns the
   controlled vocabulary that `ec_document_type` points at.

## Conventions you must follow

- **Identifiers.** Use `documentId` (a string) in the `{id}` path parameter and in
  filters. The numeric `id` is the legacy Strapi 4 identifier and is not stable here.
- **Pagination.** Collections default to a page; pass `pagination[page]` and
  `pagination[pageSize]`, and set `pagination[withCount]=true` when you need
  `meta.pagination.total`. Never assume the first page is the whole catalog.
- **Relations are not returned by default.** Anything you need from `file`,
  `ec_document_type`, `ec_status`, `ec_products` or `ec_groups` must be named in
  `populate`.
- **Visibility is entitlement-scoped.** `ec_groups` and `ec_status` gate what a given
  account can see. An empty result set is a normal, correct answer for an account that is
  not entitled to that product — do not retry it as an error.
- **No idempotency contract.** This skill is read-only, so this does not bite here, but do
  not assume safe retries on the write operations of this API.

## Errors

| Status | `error.name` | What to do |
|---|---|---|
| 400 | BadRequestError | Malformed filter or populate syntax — check the `filters[...]` nesting. |
| 401 | UnauthorizedError | Token missing, expired or malformed. Re-authenticate. |
| 403 | ForbiddenError | No token at all, or the role lacks the grant for this controller action. |
| 404 | NotFoundError | Wrong `documentId`, or the record is unpublished. |
| 500 | InternalServerError | Retry once, then escalate to <https://support.ethernovia.com/support/home>. |

Errors arrive as `{"data":null,"error":{"status","name","message","details"}}` with
`Content-Type: application/json`. This is **not** RFC 9457 — do not parse it as
`application/problem+json`.
