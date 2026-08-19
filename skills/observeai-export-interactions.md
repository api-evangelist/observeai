---
name: Export Observe.AI interactions for a date range
description: Authenticate, register an asynchronous interactions report, poll it to completion, and download the resulting file before its pre-signed URL expires.
api: openapi/observeai-reporting-apis-openapi.yml
operations:
  - Create Auth Token
  - Create Interaction Request
  - Get Interactions
generated: '2026-08-14'
method: generated
source: openapi/observeai-reporting-apis-openapi.yml + https://api-docs.observe.ai/
---

# Export Observe.AI interactions for a date range

Pulls interaction records — Moments, transcripts and metadata for voice calls,
webchat and email — out of Observe.AI. This is the platform's primary export
flow and the one every other reporting flow is shaped like.

## Before you start

- You need an **App ID** and **App Secret**. These are not self-serve: request
  them from `help@observe.ai` or your CSM, generate them in the Observe.AI app,
  and download them once — they cannot be retrieved again.
- You need your **tenant base URL**. The spec declares `https://{base_url}`
  because each cluster differs; Observe.AI issues yours via CSM or support
  ticket. Do not guess it.

## Steps

1. **Mint a token.** Call `Create Auth Token` (`POST /v1/oauth/token`) with the
   App ID and App Secret in the `OauthTokenRequest` body. The response is an
   `OauthTokenResponse` carrying a bearer JWT.

   **Cache this token.** It is valid for 7200 seconds (2 hours), and the token
   endpoint is itself capped at **1500 requests/day** per app credential. Minting
   a token per request will exhaust that budget long before the reporting budget.

2. **Register the report.** Call `Create Interaction Request`
   (`POST /v1/data/reports/interactions`) with
   `Authorization: Bearer <auth_token>` and the `InteractionsRequest` body.

   The window between `start_date` and `end_date` **must not exceed 90 days**.
   For a longer history, chunk into consecutive 90-day windows and register one
   request per chunk. There is no limit on how far back the earliest window may
   start, subject to the account's data retention settings.

   The response is a `PostJobResponse` containing a `request_id`. This POST is
   **not idempotent** — there is no Idempotency-Key header. If you retry it you
   create a second job and spend another unit of the 2000/day POST budget.
   Persist the `request_id` before doing anything else.

3. **Poll to completion.** Call `Get Interactions`
   (`GET /v1/data/reports/interactions/{request_id}`) with the same bearer token.
   The `GetJobResponse` carries a `status` of `QUEUED`, `CREATED`, `PROGRESS`,
   `COMPLETED`, `STOPPED` or `FAILED`.

   Poll no faster than **20 requests per minute** per app credential — that is
   the published GET limit. A 3-second interval sits safely under it. Treat
   `STOPPED` and `FAILED` as terminal; do not keep polling them.

4. **Download before it expires.** On `COMPLETED` the response carries a
   pre-signed S3 URL. It is valid for **24 hours**. Reporting payloads are never
   returned inline — fetch the URL and persist the contents. If it has expired,
   re-poll the same `request_id`; if that no longer resolves, register a new
   request.

## Paging

Responses page with `page` and `size`, and report `total_pages` and
`total_size`. Maximum page size is **1000** for interactions requested without
transcripts. Note the field names are snake_case — the pre-2023 Calls Report API
used `totalPages`/`totalResults`, which no longer exist.

## Handling failures

- **401** — the token expired (2-hour lifetime). Re-run step 1 and retry once.
- **429** — you exceeded 2000 POSTs/day, 20 GETs/minute or 1500 tokens/day. The
  body is `{"message": "API rate limit exceeded"}`. **No `Retry-After` or
  `RateLimit-*` header is published**, so implement exponential backoff with
  jitter rather than reading a reset time.
- **400** — usually a date window over 90 days, a malformed date, or a page size
  over the maximum. The job is not created; fix and resubmit.
- **404** — the `request_id` does not exist or has aged out. Register again.
- **403** — the credential is valid but not entitled to this account or product.
  Escalate to the CSM; retrying will not help.

Errors are a flat `{"message": "..."}` object served as `*/*`. There is no error
code to branch on — branch on the HTTP status only.
