---
name: webinarjam-export-and-unsubscribe
description: >-
  Pull the registrant/attendee list for a WebinarJam or EverWebinar session — filtered by live
  or replay attendance, purchase behaviour and date range — and unsubscribe a lead from that
  webinar's notifications.
generated: '2026-08-13'
method: generated
source: https://support.webinarjam.com/en/collections/19655423-developer-api
api: WebinarJam / EverWebinar API (no OpenAPI published — operations are transcribed from the provider's help centre)
operations:
  - POST https://api.webinarjam.com/webinarjam/webinar
  - POST https://api.webinarjam.com/webinarjam/registrants
  - POST https://api.webinarjam.com/webinarjam/unsubscribe
---

# Export registrants and unsubscribe leads

Swap `webinarjam` for `everwebinar` in every path to work against automated webinars. All calls
are POST with an `application/x-www-form-urlencoded` body. Throttle to 20 requests/second.

## Step 1 — get the schedule id

```
POST https://api.webinarjam.com/webinarjam/webinar
api_key=<key>&webinar_id=<id>
```

Keep `schedules[].schedule`. A series shares one schedule id across sessions, so narrow to a
specific session in step 2 with `date_range` rather than expecting a per-session id.

## Step 2 — list registrants and attendees

```
POST https://api.webinarjam.com/webinarjam/registrants
api_key=<key>&webinar_id=<id>&schedule_id=<schedule id>
```

Filters (all optional):

| Parameter | Values |
|---|---|
| `attended_live` | `0` all · `1` attended live · `2` did not attend · `3` left before `attended_live_timestamp` · `4` left after it |
| `attended_replay` | same 0–4 scheme, against `attended_replay_timestamp` |
| `purchased` | `0` all · `1` purchased · `2` did not purchase |
| `attended_live_timestamp` / `attended_replay_timestamp` | seconds, min 0 |
| `date_range` | `0` all time · `1` today · `2` yesterday · `3` this week · `4` last week · `5` last 7 days · `6` this month · `7` last month · `8` last 30 days |
| `search` | free-text string |
| `page` | integer, min 1 |

**Paging is thin.** `page` is the only pagination control. There is no page-size parameter, no
total count, no cursor and no link header — you cannot tell you are on the last page except by
receiving no rows. Increment `page` until the result set is empty, and respect the rate limit
while you do it.

## Step 3 — read the fields you actually need

The row is wide and mostly PII: `first_name`, `last_name`, `email`, `phone_country_code`,
`phone`, `ip`, `signup_date`, `attended_live`, `date_live`, `entered_live`, `time_live`,
`purchased_live`, `revenue_live`, `attended_replay`, `date_replay`, `time_replay`,
`purchased_replay`, `revenue_replay`, `subscribed`, `gdpr_status`, `gdpr_communications`,
`gdpr_status_date`, `gdpr_status_ip`, `twilio_consented_at`, `utm_source`, `utm_medium`,
`utm_campaign`, `utm_term`, `utm_content`, `live_room`, `replay_room`, `unsubscribe`.

- `last_name`, `phone_country_code` and `phone` are returned **only** when enabled on that
  webinar. Do not assume they exist.
- `live_room` and `replay_room` are per-attendee secret URLs. Never log or forward them.
- Take the consent fields seriously: `gdpr_status`, `gdpr_communications` and
  `twilio_consented_at` are the record of what each person agreed to. Honour `subscribed=0`.

## Step 4 — unsubscribe a lead

```
POST https://api.webinarjam.com/webinarjam/unsubscribe
api_key=<key>&webinar_id=<id>&lead_id=<lead id>
```

`lead_id` **must** come from the step 2 listing. Note the identifier mismatch: the register
operation returns `user_id`, unsubscribe consumes `lead_id`, and the provider never states that
they are the same value. Do not pass a `user_id` here on the assumption that it works.

Success is **`204 No Content`** — not the usual `{"status":"success"}` envelope. A client that
unconditionally parses JSON will break on this call.

The effect is scoped to that webinar: the lead stops receiving its pending notifications. You
can verify in the dashboard under Registrants → the **Subscribed** column reads "No".

## Failure handling

- `429` on exceeding 20 calls/second, with no `Retry-After`.
- No other error codes, bodies or shapes are documented. See
  `errors/webinarjam-problem-types.yml`.
- Unsubscribe has no idempotency contract; re-sending is presumed harmless but is not
  documented as such.
