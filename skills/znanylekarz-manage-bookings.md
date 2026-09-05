---
name: znanylekarz-manage-bookings
description: >-
  Read, confirm, move, cancel and mark attendance on ZnanyLekarz patient appointments, and request
  a patient review after a visit. Use when syncing appointment state between a practice-management
  system and the marketplace. Handles real patient data and real appointments.
api: Docplanner Integrations API v1.14.0
base_url: https://www.znanylekarz.pl/api/v3/integration
generated: '2026-09-05'
method: generated
source: openapi/znanylekarz-integrations-api.yml
operations:
  - getBookings
  - getBooking
  - bookSlot
  - confirmBooking
  - moveBooking
  - cancelBooking
  - markPatientPresence
  - markPatientAbsence
  - requestOpinion
---

# Manage ZnanyLekarz bookings

Every operationId below is verified against the provider's OpenAPI 1.14.0.

## Handle with care

These operations act on **real patients' medical appointments**. `getBookings` with
`?with=booking.patient` returns name, surname, email, phone, birth date, national identification
number (`nin`), gender and insurance number — special-category personal data under GDPR Article 9,
in an EU jurisdiction. Request that extension only when you need it, do not log it, and do not
pass it to any surface that was not part of the agreed processing.

There is no idempotency key on this API. A retried `bookSlot` can create a second real
appointment; a retried `cancelBooking` can cancel one you did not mean to.

## Steps

1. **List.** `getBookings` on the address. Useful extensions:
   `booking.patient` (patient identity), `booking.address_service` (what was booked),
   `booking.presence` (attendance), `address_service.public_insurance_flow` (NFZ public-healthcare
   bookings in Poland). `getBooking` fetches one, and supports `booking.moving`.

2. **Create.** `bookSlot` — `POST .../slots/{start}/book`. The optional `is_recurring` field exists
   (changelog 1.3.2). Returns 422 when the slot cannot carry the booking.

3. **Confirm.** `confirmBooking` (PUT) since 1.9.0. Returns 409 if already confirmed.

4. **Move.** `moveBooking`. Since 1.13.0 a booking may be moved to an `address_id` belonging to a
   **different doctor in the same facility**, which reassigns it to that doctor; an address outside
   the facility named in the path is still rejected with 403. It returns **422 when the booking
   still holds money and the target address is billed through a different payment account** — the
   documented remedy is to cancel and create a new booking instead. A payment that was fully
   refunded or charged back holds no money and does not restrict the move.

5. **Cancel.** `cancelBooking` (DELETE). No cancellation deadline is published, so do not tell a
   user there is one — and do not assume there is not; treat it as unstated.

6. **Attendance.** `markPatientPresence` (POST) and `markPatientAbsence` (DELETE) on the same path.
   A clean toggle: either call reverses the other.

7. **Ask for a review.** `requestOpinion` (PUT, on the doctor rather than the address) triggers a
   patient-facing review request. This sends a real message to a real patient — send it once, after
   the visit, and never as part of a retry loop.

## Reversibility

| Action | Reverse with | Window |
|---|---|---|
| `bookSlot` | `cancelBooking` | none stated |
| `moveBooking` | `moveBooking` back | none stated; blocked by 422 when money is held across payment accounts |
| `confirmBooking` | none published | — |
| `cancelBooking` | none — create a new booking | — |
| `markPatientPresence` | `markPatientAbsence` | none stated |
| `requestOpinion` | none — the message is sent | — |

`cancelBooking` and `requestOpinion` are the two one-way doors. Confirm intent before either.

## Errors

`400` malformed · `401` expired token · `403` outside your estate · `404` gone (possibly deleted —
the docs warn identifiers change through both the API and the Docplanner UI) · `409` state conflict
· `422` business rule, read the `message` · `429` rate limited, back off. Envelope is
`{"errors": [], "message": "..."}` under `application/vnd.error+docplanner+json`.
