---
name: Fetch Observe.AI GenAI interaction summaries
description: Retrieve Summarization AI summaries for interactions, either across a date window or for a known list of interaction ids.
api: openapi/observeai-reporting-apis-openapi.yml
operations:
  - Create Auth Token
  - Create Summaries Request
  - Get Summaries
  - Get Summaries By Ids
  - Get Interactions By Ids
generated: '2026-08-14'
method: generated
source: openapi/observeai-reporting-apis-openapi.yml + https://api-docs.observe.ai/
---

# Fetch Observe.AI GenAI interaction summaries

Summarization AI returns GenAI-generated summaries alongside Moments,
transcripts and metadata for voice calls and webchat. It lives on `/v2` of the
interactions path.

## The /v2 trap

`/v2/data/reports/interactions` is **not a newer version of**
`/v1/data/reports/interactions`. The two are concurrent surfaces of different
products: v1 returns interaction records, v2 returns those records plus GenAI
summaries. Do not "upgrade" a v1 integration to v2 expecting the same payload,
and do not assume v1 is deprecated — it is not, and nothing in the spec is
marked `deprecated: true`.

## Steps

1. **Mint a token.** `Create Auth Token` (`POST /v1/oauth/token`). Note the
   token endpoint stays on `/v1` even for v2 calls. Cache the JWT for its
   7200-second life.

2. **Choose the access pattern.**

   *For a date window:* `Create Summaries Request`
   (`POST /v2/data/reports/interactions`), 90-day maximum window, returns a
   `request_id`. Then poll `Get Summaries`
   (`GET /v2/data/reports/interactions/{request_id}`) at no more than 20
   requests/minute until `COMPLETED`, and download the pre-signed S3 URL within
   24 hours.

   *For known interactions:* `Get Summaries By Ids`
   (`POST /v2/data/reports/interactions/ids`) takes an explicit list of
   interaction ids and avoids the register/poll cycle entirely. Prefer this when
   you already have the ids — it is one call instead of many and it does not
   consume the 2000/day report-registration budget the same way a full window
   scan does.

3. **Pair with raw interactions if needed.** `Get Interactions By Ids`
   (`POST /v1/data/reports/interactions/ids`) returns the underlying interaction
   records for the same ids, so you can hold the summary and the transcript side
   by side.

## Identifier note

The summary dataset keys on `interaction_id` and `account_id`. The interactions
dataset keys the same conversation as `id`. Evaluation datasets key it as
`meeting_id`, and the retired Calls Report API called it `observeCallId`. All
four name one interaction — normalize before joining.

## Handling failures

401 means the token expired; 429 means a per-app-credential quota was exceeded
with no `Retry-After` header to read, so back off exponentially; 400 usually
means the date window exceeded 90 days; 404 means the `request_id` no longer
resolves. Error bodies are `{"message": "..."}` with no machine-readable code.

## Availability

Summarization AI was introduced 22-Jul-2024 and is a licensed module. If your
account is not entitled to it these operations return 403 rather than an empty
result — that is an entitlement signal, not a data signal, and retrying will not
change it. Escalate to the CSM.
