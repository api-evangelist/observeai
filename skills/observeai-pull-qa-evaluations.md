---
name: Pull Observe.AI QA evaluations and their ack/dispute state
description: Export Auto QA and manual evaluation scores for a period, then join each evaluation to its acknowledgement/dispute record to see which scores the agent accepted or contested.
api: openapi/observeai-reporting-apis-openapi.yml
operations:
  - Create Auth Token
  - Create Evaluation Request
  - Get Evaluations
  - Create Ack Dispute Request
  - Get Ack Dispute Details
  - Get Ack Dispute Details By Ids
generated: '2026-08-14'
method: generated
source: openapi/observeai-reporting-apis-openapi.yml + https://api-docs.observe.ai/
---

# Pull Observe.AI QA evaluations and their ack/dispute state

Quality assurance in Observe.AI is two datasets, not one. The evaluation carries
the score; the ack/dispute record carries what happened after the agent saw it.
Reporting on QA without the second half will overstate how settled your scores
are.

## Steps

1. **Mint a token.** `Create Auth Token` (`POST /v1/oauth/token`) with the App ID
   and App Secret. Cache the JWT — 7200-second lifetime, 1500 mints/day.

2. **Register the evaluations report.** `Create Evaluation Request`
   (`POST /v1/data/reports/evaluations`). The date window must be **90 days or
   less**. Returns a `request_id`.

3. **Poll it.** `Get Evaluations` (`GET /v1/data/reports/evaluations/{request_id}`)
   at no more than 20 requests/minute until `status` is `COMPLETED`, then fetch
   the pre-signed S3 URL within 24 hours.

4. **Register the ack/dispute report.** `Create Ack Dispute Request`
   (`POST /v1/data/reports/evaluation/ackdisputedetails`), same 90-day cap.

5. **Poll it.** `Get Ack Dispute Details`
   (`GET /v1/data/reports/evaluation/ackdisputedetails/{request_id}`).

   If you only need state for a known set of evaluations rather than a whole
   window, use `Get Ack Dispute Details By Ids`
   (`POST /v1/data/reports/evaluation/ackdisputedetails/ids`) instead — it skips
   the poll cycle.

6. **Join.** Ack/dispute records key on `evaluation_id` (and `evaluation_form_id`
   for the template). Join them to the evaluation's `id`.

## Reading the data

The evaluations payload nests four levels deep: `evaluation_forms` ->
`template` (with `sections` -> `questions`, each carrying `max_score`) ->
`evaluations` -> `evaluation_list` -> `response` and `scores_obtained`.

`scores_obtained` gives `final_score`, `total_points`, `percent_score`,
`total_bonus`, `total_penalties`, `grade_assigned` and `auto_fail`.

**Separate Auto QA from manual before you aggregate.** `evaluation_type` is
`Manual` or `Auto QA`. On Auto QA rows the evaluator fields are placeholders —
`evaluator_id` is the literal string `"NA"` and `evaluator_name` is
`"Observe.AI"`. Treating those as a real evaluator will produce a phantom
top-performing reviewer in any per-evaluator rollup.

Also filter on `evaluation_purpose`, which is `AGENT_PERFORMANCE`, `CALIBRATION`,
`FINAL_CALIBRATION`, or null. Calibration evaluations are quality-team exercises,
not agent scores — including them inflates evaluation volume and distorts agent
averages. The field is only populated for `evaluation_type = Manual`.

`ack_info.ack_status` is one of `NOT_INITIATED`, `AWAITING_ACK`, `ACKED`,
`IN_DISPUTE`, `DISPUTE_ACCEPTED`, `DISPUTE_PARTIAL` or `DISPUTE_REJECTED`, with
timestamps `ack_init_time`, `ack_time`, `dispute_raised_at`,
`dispute_resolved_at` and the email of who raised and resolved it. A score in
`IN_DISPUTE` is not final.

## Joining to your own systems

Every person carries a paired external id — `partner_agent_id` and
`partner_evaluator_id` hold the identifiers from your own CRM/CCaaS/WFM. Join on
those rather than building a name-matching table.

## Handling failures

Same as any Observe.AI reporting flow: 401 means the 2-hour token expired,
429 means a quota was hit with no `Retry-After` to read, 400 usually means the
date window exceeded 90 days, 404 means the `request_id` aged out. Evaluations
pages at a maximum of 1000 per page.

## Version note

Two historical changes still bite when reconciling against old extracts: the
default evaluation type changed to Manual on 12-Oct-2022, and the field
`present:true` was removed from all evaluation, coaching and interaction reports
on 19-Sep-2022.
