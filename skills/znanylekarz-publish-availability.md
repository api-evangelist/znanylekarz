---
name: znanylekarz-publish-availability
description: >-
  Publish and maintain a doctor's bookable availability on the ZnanyLekarz marketplace from a
  practice-management system — enable the calendar, replace slots for a date range, and hide time
  with calendar breaks. Use when a clinic's schedule changes and the marketplace must match it.
api: Docplanner Integrations API v1.14.0
base_url: https://www.znanylekarz.pl/api/v3/integration
generated: '2026-09-05'
method: generated
source: openapi/znanylekarz-integrations-api.yml
operations:
  - getFacilities
  - getDoctors
  - getAddresses
  - getCalendar
  - enableCalendar
  - getSlots
  - replaceSlots
  - deleteSlots
  - getCalendarBreaks
  - addCalendarBreak
  - moveCalendarBreak
  - deleteCalendarBreak
---

# Publish availability to ZnanyLekarz

Every operationId below is verified against the provider's OpenAPI 1.14.0. Do not invent others.

## Before you start

- Get a token: `POST https://www.znanylekarz.pl/oauth/v2/token` with HTTP Basic
  `{client_id}:{client_secret}` and body `grant_type=client_credentials&scope=integration`.
  The token is a bearer token valid for 3600 seconds. Send it as `Authorization: Bearer {token}`.
- HTTPS is mandatory. There is no sandbox hostname — the environment is chosen by WHICH key you
  use, against the same URL. Confirm you are holding sandbox credentials before any write.
- **There is no idempotency key and no dry-run on this API.** Every write below changes a live
  clinic calendar. Never blind-retry a write; re-read state first (step 5).

## Steps

1. **Resolve the estate.** `getFacilities` → pick `facility_id`. Then `getDoctors` for that
   facility → `doctor_id`. Then `getAddresses` → `address_id`. An address is a doctor's presence at
   one location and is the anchor for everything that follows; a 403 means the id is outside the
   estate your client was provisioned for, not that it does not exist.
   Save a round trip with `?with=facility.doctors` on `getFacility`, and `?with=doctor.addresses`
   on `getDoctor`.

2. **Check the calendar.** `getCalendar` returns a `status`. If online booking is off, call
   `enableCalendar`. It returns 409 if the calendar is already enabled — treat 409 here as
   success-equivalent, not as an error to retry.

3. **Read what is already published.** `getSlots` for the date range. **The range may not exceed
   180 days** (changelog 1.9.3) — a wider window returns 400. Add `?with=slot.services` to see
   which address services each slot can be booked for.

4. **Publish availability.** `replaceSlots` (PUT) takes WORK PERIODS with durations, not individual
   slots — Docplanner computes the segmentation itself. This operation REPLACES the availability in
   the range you send. It can return 429; back off exponentially and read `X-RateLimit-Reset`.
   Use `deleteSlots` only to clear a specific date.

5. **On any ambiguous failure, re-read before retrying.** A timeout on `replaceSlots` may have
   succeeded. Call `getSlots` for the same range and compare before sending it again. This is the
   only protection available — the API has no replay guard.

6. **Hide time with breaks, not by deleting slots.** `addCalendarBreak` with `since`/`till` hides
   slots for lunch, holidays or leave, and is reversible with `deleteCalendarBreak`. Breaks may
   overlap, but an exact duplicate timeframe returns 409. Since 1.14.0 the optional
   `apply_on_coupled_addresses` field creates the same break on the doctor's other addresses in the
   facility, and `moveCalendarBreak` / `deleteCalendarBreak` then move or remove those coupled
   breaks together with the one they follow.

## Reversibility

| Action | Reverse with | Window |
|---|---|---|
| `enableCalendar` | `disableCalendar` | none stated |
| `addCalendarBreak` | `deleteCalendarBreak` | none stated |
| `moveCalendarBreak` | `moveCalendarBreak` back | none stated |
| `replaceSlots` | `replaceSlots` with the prior periods (capture them in step 3 first) | none stated |
| `deleteSlots` | `replaceSlots` republishes availability, but does not restore booking state | none stated |

Capture the output of step 3 before step 4. It is the only rollback material you will have.

## Errors

`400` malformed or out-of-range · `401` expired token, re-issue · `403` outside your estate or
client not enabled · `404` gone · `409` already in that state · `422` business rule ·
`429` rate limited. Bodies are `application/vnd.error+docplanner+json` shaped
`{"errors": [], "message": "..."}` — there is no machine-readable error code, so branch on the
status, not on the message text.
