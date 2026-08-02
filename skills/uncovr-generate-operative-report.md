---
name: Generate an operative report from surgical embeddings
description: Submit a surgical-video embeddings file to the Uncovr inference
  API for a supported operation and handle the report / accepted / error
  response union correctly.
api: openapi/uncovr-api-openapi-original.json
operations:
  - process_embeddings_compat_v1_inference__operation__process_embeddings_post
---

# Generate an operative report from surgical embeddings

Uncovr's public API surface (https://api.uncovr.ai, OpenAPI 3.1 served at
/openapi.json) exposes one inference operation. It is bearer-authenticated
and not publicly documented — credentials come from Uncovr directly.

## Steps

1. **Authenticate.** Send `Authorization: Bearer <token>` on every request
   (securityScheme `HTTPBearer`; see
   `authentication/uncovr-authentication.yml`). There is no OAuth or API-key
   flow.
2. **Pick the operation.** The path parameter `operation` must be one of the
   supported enum values: `cholecystectomy` or `hiatal-hernia`. Any other
   value fails validation.
3. **Call the endpoint.** `POST
   /v1/inference/{operation}/process_embeddings` (operationId
   `process_embeddings_compat_v1_inference__operation__process_embeddings_post`)
   with `multipart/form-data` containing the required binary `file` field
   (the embeddings file). Optional query parameters: `case_id` (UUID),
   `video_id` (integer, default 1), `user_id`, `device_id`.
4. **Discriminate the 200 union.** A 200 response is one of three shapes —
   inspect the fields:
   - `report` / `assessments` present → `IngestionResponse`: the operative
     report text plus `CaseAssessment[]` (name, finding, severity
     normal|mild|moderate|severe, evidence_frame, rationale).
   - `code: ingestion_accepted` → `IngestionAcceptedResponse`: input was
     queued for asynchronous processing; the report is not returned inline.
   - `error_code` present → `IngestionErrorResponse`: a domain error carried
     in-band; surface `error_code` and `detail`, do not retry blindly.
5. **Handle 422.** Validation failures return FastAPI's
   `HTTPValidationError` envelope (`detail[]` of loc/msg/type). Fix the
   request rather than retrying (see `errors/uncovr-problem-types.yml`).

## Cautions

- No idempotency contract, rate-limit signaling, or pagination exists on
  this surface (`conventions/uncovr-conventions.yml`).
- This is a clinical-intelligence API: treat outputs as draft documentation
  requiring surgeon review, and keep PHI out of requests — Uncovr's
  architecture is zero-PHI by design.
