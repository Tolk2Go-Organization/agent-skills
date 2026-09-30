---
name: booking-tolk2go-interpreters
description: "Books and manages professional interpreters through Tolk2Go's public API. Use when a user or agent needs to discover interpreter services, search availability, create or track a booking, complete a deposit, request a booking change, or cancel a booking."
---

# Booking Tolk2Go Interpreters

Use Tolk2Go's public, self-service booking API. Public search requires no credential. Recurring booking access uses a revocable account-scoped agent key and requires no partner onboarding or webhook receiver.

- Base URL: `https://api.tolk2go.com/api/v1`
- OpenAPI source of truth: `https://api.tolk2go.com/api/v1/spec`
- MCP endpoint: `https://api.tolk2go.com/api/v1/mcp`
- All progress is obtained by authenticated polling. Do not submit or wait for a webhook.

## Safety and credentials

- Do not expose passwords, Bearer tokens, personal details or payment URLs in logs or summaries.
- Ask the user before creating a request if the intended language, time, mode and location are not already clear.
- Before selecting an interpreter, show the relevant price and obtain confirmation when selection creates a financial commitment.
- Never enter or request card details. Return the secure `payment.payment_url` for the customer to open.
- Confirm with the user before submitting a change or cancellation unless they already explicitly requested that exact action.
- Store agent keys only in the runtime credential vault. Never put them in prompts, memory, logs or user-visible summaries.
- Use a unique idempotency key for every write and reuse it only when retrying that exact operation.

## Journey

### 1. Discover the contract

Fetch `/spec` when field or response details are needed. Catalogue and availability routes are public:

- `GET /languages`
- `GET /language-pairs?language_id={id}&lang={code}`
- `GET /situations?lang={code}`
- `GET /situations/{situation_id}/details?lang={code}`
- `GET /delivery-modes?lang={code}`
- `GET /service-cards?lang={code}`

Use IDs or slugs returned by these endpoints; do not invent identifiers. Select a situation before searching availability.

### 2. Search availability

Call `GET /availability` with:

- `language_pair_id`, `situation_id`, `start`, `end` and `type` (`phone` or `address`)
- `lat` and `lng` for an address booking
- optional `sort` (`price_asc` or `rating_desc`)

Use ISO-8601 datetimes. Preserve the customer's timezone separately for booking. Present suitable results and retain the chosen interpreter IDs and quoted financial details.

### 3. Register or log in

For a new customer, call `POST /register` with `email` and a strong unique password. Optional fields are `first_name`, `last_name`, `company_name` and `vat_number`. A `409` means the email already has an account; log in instead rather than registering another address.

Call `POST /login` with the email and password. Keep the returned token private and send it as `Authorization: Bearer {token}`. The token is valid for 300 seconds; log in again as needed throughout polling.

For recurring access, use that login token once with `POST /agent-access-keys`, then securely store the returned plaintext agent key. It defaults to 90 days and can be rotated or revoked only after normal account login. Use the agent key for later REST calls or as the Bearer credential for the booking MCP endpoint. If it is expired or revoked, ask the customer to reconnect through account login; never create a replacement identity.

### 4. Create the booking request

Call `POST /bookings` with the owner credential and a unique `Idempotency-Key` header and:

```json
{
  "language_pair_id": "id-or-slug-from-catalogue",
  "situation_id": "id-or-slug-from-catalogue",
  "start": "2026-10-15T09:00:00.000Z",
  "end": "2026-10-15T10:00:00.000Z",
  "type": "phone",
  "phone_type": "video",
  "bookingTimezone": "Europe/Amsterdam",
  "description": "Context needed by the interpreter",
  "client_reference": "optional-customer-reference",
  "preferred_interpreter_ids": ["id-from-availability"]
}
```

For `address`, include `location` with `lat`, `lng` and `address`. For remote work, send the customer's IANA `bookingTimezone`. Save the returned `booking_id`. A new API account can have only one searching request before completing its first booking.

Each request covers one `start` and `end`. For recurring appointments, such as every Monday for eight weeks, create one request per occurrence, each with its own `Idempotency-Key`, and track every returned `booking_id` separately.

Only for the Conference / Simultaneous situation (slug `simultaan`): when the session lasts more than 1 hour, recommend a second interpreter. If the customer wants one, submit a separate request for the same time.

### 5. Poll responses and select

Poll `GET /bookings/{booking_id}/responses` using the owner's Bearer token. An empty `responses` array means wait and poll again. Use `GET /bookings/{booking_id}` to read `polling.recommended_after_seconds` (currently 30); add backoff for transport or 5xx failures.

When the customer chooses an available response, call:

```http
POST /bookings/{booking_id}/book
Authorization: Bearer {token}
Content-Type: application/json

{"interpreter_id":"id-from-responses"}
```

Do not select an ID that was not returned as available for this booking.

### 6. Complete payment and confirmation

Inspect the returned `payment` object:

- `next_action: open_payment_url`: give the customer the exact HTTPS `payment_url` and ask them to complete payment.
- `next_action: poll_booking`: poll the authoritative booking until a payment URL or final state appears.
- `next_action: none`: no payment action is needed.

After customer action, poll `GET /bookings/{booking_id}` until `payment.status` is `paid` and booking `status` is `confirmed`. Do not claim confirmation while the booking is `booked_pending_payment`.

### 7. Track, change or cancel

`GET /bookings/{booking_id}` is authoritative and owner-scoped. Possible states include `searching`, `booked_pending_payment`, `confirmed`, `reported`, `completed`, `disputed`, `cancelled` and `processing`.

For recurring monitoring, call `GET /bookings?updated_since={timestamp}` once and retain the returned `change_cursor`; later call `GET /bookings?cursor={cursor}` before polling changed bookings individually. The same journey is available through namespaced `tolk2go_interpreting_*` MCP tools.

To request a change to a booked or confirmed booking, `PATCH /bookings/{booking_id}` with at least one supported field: `start`, `end`, `bookingTimezone`, `type`, `phone_type`, `location`, `description`, `situation_id` or `language_pair_id`. This uses Tolk2Go's customer change-request flow and may require interpreter approval. A `202` means submitted, not approved; poll until `change_request.status` returns from `pending` to `none`, then verify the resulting booking fields.

To cancel, call `DELETE /bookings/{booking_id}` with an optional JSON `reason`. Poll once more and verify `status: cancelled` before reporting success.

## Error handling

- `400`: correct missing or unsupported input from the API response.
- `401`: log in again; retry reads, but inspect booking state before retrying writes.
- `403`: a new account must complete its first request before creating another.
- `404`: do not disclose whether another account owns the ID; re-check catalogue IDs or the current account.
- `409`: registration already exists, or a booking change conflicts with policy/pending work.
- `422`: correct the supplied address/location.
- `429`: respect the limit and do not rotate identities or IP addresses.

Report the booking ID, current state, next action and any user action required. Never report a booking, change, payment or cancellation as complete until an authoritative API read confirms it.
