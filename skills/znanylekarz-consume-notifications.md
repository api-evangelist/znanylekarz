---
name: znanylekarz-consume-notifications
description: >-
  Consume ZnanyLekarz appointment events — by pulling the FIFO notification queue or by receiving
  pushed callbacks — and correctly answer the two BLOCKING events that can approve or deny a
  patient's booking in real time. Use when keeping a practice-management system in sync with
  marketplace activity.
api: Docplanner Integrations API v1.14.0
base_url: https://www.znanylekarz.pl/api/v3/integration
generated: '2026-09-05'
method: generated
source: openapi/znanylekarz-integrations-api.yml
operations:
  - Pull Notification
  - Pull Multiple Notification
  - Release Notifications
---

# Consume ZnanyLekarz notifications

17 event types are defined as OpenAPI callbacks on `POST /{client-endpoint-url}` in the provider's
own contract. Each has a typed notification schema. Nothing below is invented.

## Choose a delivery mode

**Pull** — you poll. `GET /notifications` returns the earliest unpulled notification, one per
request, FIFO, until the queue is empty; `GET /notifications/multiple?limit=N` (1–100, default 1)
returns a batch plus the count of notifications remaining. **Notifications not pulled within 72
hours are marked expired and deleted.** There is no replay for an expired notification.

**Push** — Docplanner POSTs to an endpoint you supply. Ordinary events are pushed **once**
regardless of the status your endpoint returns; a delivery that fails outright is retried twice
more, after 5 and 10 minutes. Missed pushes can be re-dispatched with
`POST /notifications/release`, but **only once every 60 minutes** — the endpoint returns 429 with a
`Retry-After` header carrying the seconds remaining — and **failed notifications are permanently
deleted 14 days after creation**. It returns 403 if your client is not on the push model.

Both modes may run at once. Push traffic arrives from a single Docplanner IP; the current source
addresses are published as machine-readable JSON at
`https://www.znanylekarz.pl/public/docs/public-ips.json`.

## The two blocking events — get these right

`slot-booking` and `booking-moving` are **synchronous approval gates**, not notifications.

- `slot-booking`: sent when a patient is about to book. Return **2xx to approve**. Anything other
  than 2xx **denies the booking** and the patient does not get the appointment.
- `booking-moving`: the same contract for a move.

To deny with a message shown to the patient, respond `application/json` with `{"error_code": 1}`.

Treat a timeout in your handler as a denial with no explanation — because that is what the patient
will experience. Keep these handlers fast and make them fail closed only deliberately.

## The other 15 events

`slot-booked` · `booking-canceled` · `booking-moved` · `booking-confirmed` ·
`booking-payment-status-changed` · `break-created` · `break-moved` · `break-removed` ·
`presence-marked` · `address-service-created` · `address-service-changed` ·
`address-service-deleted` · `address-commercial-type-changed` · `address-assigned` ·
`address-unassigned`.

Several are optional and enabled on request; `address-assigned` and `address-unassigned` are
Docplanner PMS clients only. Acknowledge with 200.

## Verify what you received

**There is no webhook signature.** No HMAC header, no signing secret, no message-signature scheme
is published. Authenticity rests entirely on the published source-IP allowlist, so:

- allowlist the published IPs at your edge, and refresh that list — it is served, not static;
- treat the payload as a claim, not as truth. For anything that moves money or cancels care,
  re-read the authoritative record with `getBooking` before acting on it.

## Envelope

A pulled notification is `{name, data, created_at}` served as
`application/vnd.docplanner+json; charset=UTF-8`. `name` is the event key from the list above and
`data` carries that event's typed payload. `GET /notifications/multiple` additionally reports how
many notifications remain — the only endpoint in this API that tells you the size of what is left.
