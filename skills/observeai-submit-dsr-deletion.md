---
name: Submit and track an Observe.AI DSR deletion request
description: Programmatically delete call audio, transcripts and screen recordings matched by metadata rules to satisfy a data subject request, then poll the deletion job to a terminal state.
api: openapi/observeai-reporting-apis-openapi.yml
operations:
  - Create Auth Token
  - Create Dsr Delete Request
  - Get Dsr Delete Status
generated: '2026-08-14'
method: generated
source: openapi/observeai-reporting-apis-openapi.yml + https://api-docs.observe.ai/#tag/DSR
---

# Submit and track an Observe.AI DSR deletion request

Data Subject Request (DSR) deletion is the one **destructive** operation on
Observe.AI's public API. It permanently removes recorded conversation data
matched by metadata rules.

## Stop before you call this

This operation is irreversible. There is no dry-run mode, no preview of what a
rule set will match, no soft delete and no undo published anywhere in the API or
the docs. A rule that is broader than intended will delete production
conversation data — including the transcripts every QA evaluation, coaching
record and report in the platform was derived from.

**An autonomous agent must not invoke `Create Dsr Delete Request` without
explicit human approval of the exact rule set.** Show the operator the literal
`entity_type` and `rules` payload and get confirmation on that payload, not on
a summary of it. Log the submitted request and the returned `job_id`.

There is also no idempotency key. Re-submitting after a timeout queues a second
deletion job rather than returning the first.

## Steps

1. **Mint a token.** `Create Auth Token` (`POST /v1/oauth/token`) with the App ID
   and App Secret. The DSR operations are among the only two in the spec that
   explicitly declare `security: [bearerAuth]`.

2. **Build the rule set.** The `DsrDeleteRequest` body requires `entity_type`
   and `rules`.

   `entity_type` is exactly one of:
   - `AUDIO_TRANSCRIPT` — both the audio and its transcript
   - `AUDIO` — the recording only
   - `TRANSCRIPT` — the transcript only
   - `SCREEN_RECORDING` — agent screen capture

   Each rule in `rules` is a `DsrRule` with a `type` of `OAI_METADATA` or
   `CUSTOMER_METADATA`, a `name`, and a list of `values`. For `OAI_METADATA` the
   `name` must be `DURATION`, `ENTITYTYPE` or `STATUS`. For `CUSTOMER_METADATA`
   the `name` is one of your own metadata fields — this is the path you will
   normally use to target one data subject.

   Scope the rules as narrowly as the request allows. Prefer a
   `CUSTOMER_METADATA` match on a specific customer identifier over any
   `OAI_METADATA` filter, which matches on platform attributes and will fan out
   far wider than a single subject.

3. **Submit.** `Create Dsr Delete Request`
   (`POST /v1/dsr/delete/on-demand`). The `DsrDeleteResponse` returns a
   `job_id`, a `status`, a `message`, `requested_at` and
   `expected_completion_by`. Persist all of it — `expected_completion_by` is
   your evidence of the SLA on the request, and the `job_id` is the only handle
   you will get.

4. **Poll to a terminal state.** `Get Dsr Delete Status`
   (`GET /v1/dsr/delete/{job_id}/status`) returns a `DsrStatusResponse` with
   `request_id`, `status` and `status_message`.

   `status` is `QUEUED`, `CREATED`, `PROGRESS`, `COMPLETED`, `STOPPED` or
   `FAILED`. Only `COMPLETED` is proof of deletion. `STOPPED` and `FAILED` are
   terminal and mean the data is still present — escalate rather than assuming
   the request succeeded. Keep the terminal response as your compliance record.

## Handling failures

- **401** — token expired after its 2-hour life. Re-mint and retry the *status*
  call. Do not blindly re-submit the deletion.
- **403** — the account is not entitled to the DSR API. It was released
  2026-02-10 and may not be enabled on older contracts; contact your CSM.
- **400** — the rule set failed validation, commonly an `OAI_METADATA` rule with
  a `name` outside `DURATION`/`ENTITYTYPE`/`STATUS`. No job was created.
- **500** — do not retry a submission on a 500 without first checking whether a
  job was created; there is no idempotency key to protect you.
