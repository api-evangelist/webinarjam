---
name: webinarjam-register-attendee
description: >-
  Register a person for a WebinarJam or EverWebinar webinar and return their unique live-room,
  replay-room and thank-you URLs. Handles the mandatory read-before-write chain (list webinars →
  get webinar detail for the schedule id and custom-field labels → register).
generated: '2026-08-13'
method: generated
source: https://support.webinarjam.com/en/collections/19655423-developer-api
api: WebinarJam / EverWebinar API (no OpenAPI published — operations are transcribed from the provider's help centre)
operations:
  - POST https://api.webinarjam.com/webinarjam/webinars
  - POST https://api.webinarjam.com/webinarjam/webinar
  - POST https://api.webinarjam.com/webinarjam/register
  - POST https://api.webinarjam.com/api/webinarjam/countries
---

# Register an attendee for a WebinarJam or EverWebinar webinar

WebinarJam publishes no OpenAPI. Every URL, parameter and response field below is transcribed
verbatim from the provider's own developer articles; nothing here is inferred.

## Before you start

- You need an **approved** API key. Access requires a paid subscription plus an application
  reviewed by WebinarJam (typically two business days).
  See `authentication/webinarjam-authentication.yml`.
- One key covers **both** products. Swap `webinarjam` for `everwebinar` in the path to work
  against automated webinars.
- **Every call is a POST**, including the reads. The body is
  `application/x-www-form-urlencoded`. A GET returns 405.
- Throttle to **20 requests/second**. Exceeding it returns 429 with no `Retry-After`, so pick
  your own back-off. See `rate-limits/webinarjam-rate-limits.yml`.

## Step 1 — find the webinar

```
POST https://api.webinarjam.com/webinarjam/webinars
api_key=<key>
```

Returns `{"status":"success","webinars":[{webinar_id, webinar_hash, name, title, description,
type, schedules, timezone}, ...]}`. Match on `name` (private) or `title` (public) and keep
`webinar_id`.

## Step 2 — read the webinar detail to get a schedule id

```
POST https://api.webinarjam.com/webinarjam/webinar
api_key=<key>&webinar_id=<id>
```

You need this call. `schedule` is **only** available here, and the value it returns is *not*
the schedule id shown in the Schedules tab of the webinar settings UI — use the API value.

If the webinar is a **series**, all sessions share one schedule id. Pick the individual session
by matching the `schedules[].date` value (`YYYY-MM-DD HH:MM`).

This response also carries the custom registration fields you will need in step 3.

## Step 3 — register

```
POST https://api.webinarjam.com/webinarjam/register
api_key=<key>
webinar_id=<id>
schedule=<schedule id from step 2>
first_name=<string>
email=<string>
```

Optional / conditionally required fields: `last_name`, `country`, `state`, `ip_address`,
`phone_country_code` (with `+`), `phone` (digits only), `twilio_consent` (`1`/`0`),
`timezone_id`.

Conditions the provider states explicitly:

- `phone_country_code` and `phone` become **mandatory** if the phone field is enabled on that
  webinar.
- `timezone_id` is **mandatory for registrants in Texas, USA** — pass `2` for Mountain Time or
  `3` for Central Time.
- For **EverWebinar** only, you may also pass `timezone` (`GMT-5`, `GMT+4:30`) and `date`
  (`2025-01-01 09:00`). If you pass a `date` it must exactly match a date returned by step 2,
  or the registration silently fails to attach to any session. If the read used a custom
  timezone, pass the same timezone on the write.

### Custom fields

Use the field **label** from step 2 as the parameter name.

- Text field → pass the value directly: `company=XYZ`
- Dropdown → pass the answer option **ID**; for multi-select pass an array:
  `whereDidYouHearAboutUs=["id_1","id_2"]`

### Series

Register the person **once, against the first schedule**. The API auto-registers them to every
following session in the series. Do not loop over schedules — you will create duplicates.

## Step 4 — use the response

```json
{"status":"success","user":{"webinar_id":5,"webinar_hash":"...","user_id":1234567,
 "first_name":"...","email":"...","schedule":34,"date":"2024-01-05 13:00",
 "timezone":"America/Los_Angeles","live_room_url":"...","replay_room_url":"...",
 "thank_you_url":"..."}}
```

`live_room_url`, `replay_room_url` and `thank_you_url` are **unique to that attendee** — treat
them as secrets and deliver them only to that person. Some fields (`password`, `phone`,
`last_name`) are returned only when enabled on the webinar, so code defensively.

## Failure handling

- `429` — you exceeded 20 calls/second. Queue and retry.
- Everything else is undocumented. WebinarJam publishes no error catalogue, no error body shape
  and no error codes. See `errors/webinarjam-problem-types.yml`.
- **There is no idempotency key.** A retried registration is a second registration. Deduplicate
  on your own side before re-sending.

## Consent

Registrant data is PII. WebinarJam states the integrator is responsible for collecting consent
to handle personal data, to subscribe the person to webinars, and to contact them later. If you
set `twilio_consent=1` you are asserting the person consented to SMS.
