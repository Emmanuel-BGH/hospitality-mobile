# HANDYPROS HOSPITALITY

## API Documentation & Contract

**Document:** `HANDYPROS_API_DOCUMENTATION.md`\
**Version:** 1.0\
**Status:** Engineering API Baseline\
**Primary clients:** Web Application, Mobile Application, Admin
Dashboard\
**API base path:** `/api/v1`

------------------------------------------------------------------------

# 1. Purpose

This document defines the HTTP API contract for HandyPros Hospitality.

It is derived from:

-   HandyPros Hospitality Web Application PRD
-   HandyPros Hospitality Admin Dashboard PRD
-   `HANDYPROS_ENGINEERING_SPECIFICATION.md`

The API is the shared application contract between the backend and:

-   Client Web App
-   Partner Web App
-   Mobile App
-   Admin Dashboard

The backend remains the authoritative source of booking, payment,
availability, financial, dispute, withdrawal, notification, and
authorization state.

The exact route names below are the proposed v1 contract. Where the PRDs
define a business capability but do not prescribe a literal URL or
payload, this document establishes the implementation contract rather
than claiming that the PRD supplied the exact route.

------------------------------------------------------------------------

# 2. API Design Principles

## 2.1 Versioning

All application endpoints use:

`/api/v1`

Example:

`GET /api/v1/listings`

Breaking changes require a new API version.

## 2.2 JSON

Unless explicitly stated otherwise:

-   Request body: `application/json`
-   Response body: `application/json`
-   Encoding: UTF-8

## 2.3 Authentication

Authenticated requests use:

`Authorization: Bearer <access_token>`

Authentication tokens are issued by the Authentication Service.

The PRD requires role-based access control for Client, Partner, and
Admin users.

## 2.4 Authorization

The backend must enforce:

-   role
-   resource ownership
-   account status
-   action permission
-   booking state
-   financial eligibility

Frontend visibility is never an authorization mechanism.

## 2.5 IDs

Public API resources should expose stable opaque IDs or UUIDs.

Do not expose database auto-increment assumptions as part of the API
contract.

Examples:

`usr_01...`

`lst_01...`

`bkg_01...`

`pay_01...`

`wd_01...`

The exact ID implementation may be selected during database design.

## 2.6 Monetary values

All monetary API values must be represented as integer minor units.

For NGN:

`15000000` = ₦150,000.00 if the currency uses two minor digits.

However, because the current product operates primarily in NGN, the API
may expose a display amount separately where useful.

Canonical financial fields should use:

``` json
{
  "amount": 15000000,
  "currency": "NGN"
}
```

The frontend must never calculate authoritative financial values.

## 2.7 Dates and times

API timestamps use ISO 8601 UTC timestamps.

Example:

`2026-08-20T14:30:00Z`

Where the business rule is explicitly WAT-based, such as the daily
reconciliation job specified at 2am WAT, the backend handles timezone
conversion.

## 2.8 Pagination

Collection endpoints use:

``` text
?page=1&page_size=20
```

Default page size:

`20`

Maximum:

`100`

Response:

``` json
{
  "success": true,
  "data": [],
  "meta": {
    "page": 1,
    "page_size": 20,
    "total": 120,
    "total_pages": 6
  }
}
```

## 2.9 Errors

Standard error structure:

``` json
{
  "success": false,
  "error": {
    "code": "BOOKING_NOT_AVAILABLE",
    "message": "The selected dates are no longer available.",
    "details": {}
  }
}
```

Validation error:

``` json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "One or more fields are invalid.",
    "details": {
      "start_date": [
        "Start date must be before end date."
      ]
    }
  }
}
```

------------------------------------------------------------------------

# 3. HTTP Status Codes

  Status   Meaning
  -------- ------------------------------------------
  200      Successful request
  201      Resource created
  202      Accepted for asynchronous processing
  204      Successful request with no response body
  400      Malformed/invalid request
  401      Authentication required/invalid
  403      Authenticated but not permitted
  404      Resource not found
  409      State conflict / concurrency conflict
  422      Validation/business-rule failure
  429      Rate limit exceeded
  500      Internal server error
  502      External provider failure
  503      Service temporarily unavailable

------------------------------------------------------------------------

# 4. Idempotency

Financial and externally consequential POST requests must support
idempotency.

Client sends:

`Idempotency-Key: <unique-key>`

Required for, at minimum:

-   payment initialization
-   booking confirmation where applicable
-   Confirm Experience
-   dispute creation
-   withdrawal creation
-   admin dispute resolution
-   refunds
-   notification resend
-   Paystack-related financial commands

A repeated request with the same idempotency key must return the
original operation result rather than executing the operation again.

------------------------------------------------------------------------

# 5. Authentication API

## 5.1 Register Client

`POST /api/v1/auth/register`

**Role:** Public

### Request

``` json
{
  "first_name": "John",
  "last_name": "Doe",
  "phone": "+2348012345678",
  "email": "john@example.com",
  "password": "SecurePassword123"
}
```

The exact registration fields must follow the approved account model.

The PRD requires phone verification via WhatsApp OTP before a Client can
complete a booking.

### Response

`201 Created`

``` json
{
  "success": true,
  "data": {
    "user_id": "usr_123",
    "status": "otp_pending",
    "otp_channel": "whatsapp"
  }
}
```

------------------------------------------------------------------------

## 5.2 Request WhatsApp OTP

`POST /api/v1/auth/otp/request`

**Role:** Public

### Request

``` json
{
  "phone": "+2348012345678"
}
```

### Response

``` json
{
  "success": true,
  "data": {
    "otp_sent": true,
    "channel": "whatsapp",
    "expires_in": 300
  }
}
```

If WhatsApp OTP delivery fails, the notification system should use its
configured fallback behavior.

The PRD specifically requires WhatsApp OTP verification for Client
booking eligibility.

------------------------------------------------------------------------

## 5.3 Verify OTP

`POST /api/v1/auth/otp/verify`

**Role:** Public

### Request

``` json
{
  "phone": "+2348012345678",
  "otp": "123456"
}
```

### Response

``` json
{
  "success": true,
  "data": {
    "verified": true,
    "access_token": "<token>",
    "refresh_token": "<token>",
    "user": {
      "id": "usr_123",
      "role": "client"
    }
  }
}
```

------------------------------------------------------------------------

## 5.4 Login

`POST /api/v1/auth/login`

### Request

``` json
{
  "phone": "+2348012345678",
  "password": "SecurePassword123"
}
```

### Response

``` json
{
  "success": true,
  "data": {
    "access_token": "<token>",
    "refresh_token": "<token>",
    "user": {
      "id": "usr_123",
      "role": "client"
    }
  }
}
```

------------------------------------------------------------------------

## 5.5 Refresh Token

`POST /api/v1/auth/refresh`

### Request

``` json
{
  "refresh_token": "<token>"
}
```

### Response

``` json
{
  "success": true,
  "data": {
    "access_token": "<new-token>"
  }
}
```

------------------------------------------------------------------------

## 5.6 Logout

`POST /api/v1/auth/logout`

**Authentication:** Required

### Response

`204 No Content`

------------------------------------------------------------------------

## 5.7 Current User

`GET /api/v1/me`

**Authentication:** Required

### Response

``` json
{
  "success": true,
  "data": {
    "id": "usr_123",
    "role": "client",
    "first_name": "John",
    "last_name": "Doe",
    "phone": "+2348012345678",
    "email": "john@example.com",
    "phone_verified": true
  }
}
```

------------------------------------------------------------------------

# 6. Public Browse API

## 6.1 List Categories

`GET /api/v1/categories`

**Role:** Public

### Query parameters

``` text
vertical
active
```

### Response

``` json
{
  "success": true,
  "data": {
    {
      "id": "cat_hotel",
      "name": "Hotel",
      "vertical": "places_to_stay",
      "slug": "hotel"
    }
  ]
}
```

The PRD defines 19 hospitality categories across Places to Stay, Things
to Do, and Rentals and Experiences.

------------------------------------------------------------------------

## 6.2 List Cities

`GET /api/v1/cities`

**Role:** Public

### Response

``` json
{
  "success": true,
  "data": [
    {
      "id": "lagos",
      "name": "Lagos"
    },
    {
      "id": "lekki-ajah",
      "name": "Lekki/Ajah"
    },
    {
      "id": "abuja",
      "name": "Abuja"
    },
    {
      "id": "port-harcourt",
      "name": "Port Harcourt"
    }
  ]
}
```

------------------------------------------------------------------------

## 6.3 Search Listings

`GET /api/v1/listings`

**Role:** Public

### Query parameters

``` text
q
category_id
city_id
neighborhood
start_date
end_date
start_time
end_time
min_price
max_price
sort
page
page_size
```

### Response

``` json
{
  "success": true,
  "data": [
    {
      "id": "lst_123",
      "name": "Luxury Beach House",
      "slug": "luxury-beach-house",
      "category": {
        "id": "cat_beach_house",
        "name": "Beach-House Apartment"
      },
      "city": "Lagos",
      "neighborhood": "Lekki",
      "starting_price": {
        "amount": 25000000,
        "currency": "NGN"
      },
      "cover_image": "https://...",
      "rating": 4.8,
      "review_count": 21,
      "available": true
    }
  ],
  "meta": {}
}
```

Only approved/active listings are publicly discoverable.

------------------------------------------------------------------------

## 6.4 Listing Detail

`GET /api/v1/listings/{listing_id}`

**Role:** Public

### Response

``` json
{
  "success": true,
  "data": {
    "id": "lst_123",
    "name": "Luxury Beach House",
    "description": "...",
    "category": {},
    "partner": {
      "id": "prt_123",
      "display_name": "Example Hospitality"
    },
    "location": {},
    "images": [],
    "amenities": [],
    "pricing": {},
    "availability_summary": {},
    "payment_protection": {
      "auto_release_window_hours": 24
    }
  }
}
```

The listing detail must expose the category-specific information
required by the product.

------------------------------------------------------------------------

## 6.5 Partner Public Portfolio

`GET /api/v1/partners/{partner_id}/portfolio`

**Role:** Public

### Response

``` json
{
  "success": true,
  "data": {
    "partner_id": "prt_123",
    "display_name": "Example Hospitality",
    "listings": []
  }
}
```

------------------------------------------------------------------------

# 7. Availability API

## 7.1 Check Availability

`GET /api/v1/listings/{listing_id}/availability`

**Role:** Public

### Query parameters

For date-based services:

``` text
start_date
end_date
```

For slot-based services:

``` text
date
```

Optional:

``` text
start_time
end_time
quantity
```

### Response

``` json
{
  "success": true,
  "data": {
    "available": true,
    "slots": []
  }
}
```

Date-range listings are **available by default**. Partner date configuration
contains only exceptions (`unavailable` or `blocked`); removing an exception
reopens that date. Confirmed bookings and active inventory holds always make
their dates unavailable. The availability result reflects those current locks
as well as Partner exceptions. For a date-range query, provide both
`start_date` and `end_date`; the range is end-exclusive. For a slot query,
the returned slot objects include their identifiers and remaining capacity.

------------------------------------------------------------------------

## 7.2 Partner Availability Calendar

`GET /api/v1/partner/listings/{listing_id}/availability`

**Role:** Partner

### Query

``` text
start_date
end_date
```

### Response

``` json
{
  "success": true,
  "data": {
    "listing_id": "lst_123",
    "mode": "date",
    "default_status": "available",
    "days": []
  }
}
```

------------------------------------------------------------------------

## 7.3 Update Availability

`PUT /api/v1/partner/listings/{listing_id}/availability`

**Role:** Partner

### Request

``` json
{
  "entries": [
    {
      "date": "2026-08-20",
      "status": "unavailable"
    }
  ]
}
```

For date-range listings, submit only dates that must be closed with
`unavailable` or `blocked`; omitting an existing exception reopens that date.
The backend must prevent updates that create an invalid state against already
confirmed bookings.

------------------------------------------------------------------------

# 8. Client Booking API

## 8.1 Create Booking Quote

`POST /api/v1/listings/{listing_id}/quote`

**Role:** Public

This Phase 5 extension is the authoritative pricing boundary.  The request
contains only a schedule and guest/quantity selection; it must never contain
an authoritative amount, Partner ID, booking status, commission, payout, or
auto-release time.

### Date-range request

``` json
{
  "start_date": "2026-08-20",
  "end_date": "2026-08-23",
  "guest_count": 2,
  "quantity": 1
}
```

### Slot request

``` json
{
  "date": "2026-08-20",
  "slot_id": "slt_123",
  "guest_count": 1,
  "quantity": 1
}
```

### Response

``` json
{
  "success": true,
  "data": {
    "listing": {},
    "schedule": {},
    "quantity": 1,
    "guest_count": 2,
    "base_amount": 15000000,
    "pricing_adjustments": [],
    "final_gross_amount": 15000000,
    "amount": 15000000,
    "currency": "NGN",
    "minimum_stay": 2,
    "availability": { "available": true },
    "pricing": {
      "rule_type": "standard",
      "base_amount": 15000000,
      "final_amount": 15000000,
      "currency": "NGN"
    }
  }
}
```

All monetary values are integer minor units.  The server applies pricing
precedence of Custom Period, December/January, Public Holiday, Weekend, then
Standard.  A quote is informational only and is recalculated inside the
transaction that creates a booking.

------------------------------------------------------------------------

## 8.2 Create Booking

`POST /api/v1/bookings`

**Role:** Client

**Header:**

`Idempotency-Key: <unique-key>`

### Request

``` json
{
  "listing_id": "lst_123",
  "start_date": "2026-08-20",
  "end_date": "2026-08-22",
  "start_time": null,
  "end_time": null,
  "quantity": 1,
  "guest_count": 2
}
```

The exact fields depend on the category.

### Server actions

The server must:

1.  authenticate the Client
2.  verify phone eligibility
3.  verify listing is active
4.  validate category-specific booking fields
5.  check availability
6.  calculate authoritative price
7.  create booking in `PENDING_PAYMENT`
8.  create immutable pricing and service-detail snapshots
9.  create a temporary inventory lock in the same transaction
10. set `payment_expires_at` to 30 minutes after creation
11. record booking status history
12. return the pending-payment booking summary

Phase 5 deliberately does not initialize Paystack or create a payment
provider reference.  Payment initialization belongs to Phase 6.

### Response

`201 Created`

``` json
{
  "success": true,
  "data": {
    "booking": {
      "id": "bkg_123",
      "status": "pending_payment",
      "listing_id": "lst_123",
      "gross_amount": 15000000,
      "currency": "NGN"
    },
    "pricing": {},
    "inventory_lock": {},
    "payment_expires_at": "2026-08-20T12:30:00Z"
  }
}
```

------------------------------------------------------------------------

## 8.3 Get Booking

`GET /api/v1/bookings/{booking_id}`

**Role:** Client owner, Partner owner, authorized Admin

### Response

``` json
{
  "success": true,
  "data": {
    "id": "bkg_123",
    "status": "upcoming",
    "listing": {},
    "client": {},
    "partner": {},
    "schedule": {},
    "pricing": {},
    "payment": {},
    "release": {
      "eligible_at": "2026-08-21T14:00:00Z",
      "status": "pending"
    }
  }
}
```

Sensitive client/partner information must be filtered according to role.

------------------------------------------------------------------------

## 8.4 List Client Bookings

`GET /api/v1/me/bookings`

**Role:** Client

### Query

``` text
status
page
page_size
```

Allowed status filters:

``` text
pending_payment
upcoming
active
completed
cancelled
disputed
```

### Response

``` json
{
  "success": true,
  "data": [],
  "meta": {}
}
```

------------------------------------------------------------------------

## 8.5 Cancel Booking

`POST /api/v1/bookings/{booking_id}/cancel`

**Role:** Client where cancellation is permitted

**Header:**

`Idempotency-Key`

### Request

``` json
{
  "reason": "Change of plans"
}
```

### Response

``` json
{
  "success": true,
  "data": {
    "booking_id": "bkg_123",
    "status": "cancelled",
    "refund": {}
  }
}
```

Cancellation eligibility and refund outcome must follow the applicable
product rules.

------------------------------------------------------------------------

## 8.6 Confirm Experience

`POST /api/v1/bookings/{booking_id}/confirm-experience`

**Role:** Client owner

**Header:**

`Idempotency-Key`

### Response

``` json
{
  "success": true,
  "data": {
    "booking": {
      "id": "bkg_123",
      "status": "completed",
      "completed_at": "2026-08-21T14:00:00.000000Z"
    },
    "release": {
      "status": "released",
      "trigger_type": "client_confirmed",
      "already_released": false,
      "payout_events": [{
        "id": "pyo_123",
        "commission_record_id": "com_123",
        "amount": 9000000,
        "currency": "NGN",
        "trigger_type": "client_confirmed",
        "status": "released"
      }]
    }
  }
}
```

The endpoint must reject:

-   wrong Client
-   wrong booking state
-   any existing dispute (fail-closed)

The original `Idempotency-Key` replays the completed response without another
ledger or payout effect. A fresh key against an already settled booking also
returns the harmless `already_released: true` result; it never creates a
second payout. The command uses the same locking and release implementation
as the auto-release worker: it moves each immutable pending entitlement from
`pending` to `available`, creates one PayoutEvent per CommissionRecord, and
completes the booking atomically. It never calls a payout provider or creates
a withdrawal.

Typical errors: `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 NOT_FOUND`,
`409 SETTLEMENT_NOT_ELIGIBLE`, `409 CONFLICT` (the same key with a different
request), and `422 VALIDATION_ERROR`.

------------------------------------------------------------------------

## 8.7 Raise Dispute

`POST /api/v1/bookings/{booking_id}/disputes`

**Role:** Client owner

**Header:**

`Idempotency-Key`

### Request

``` json
{
  "reason": "service_not_as_described",
  "description": "The service materially differed from what was advertised and I could not use it as expected."
}
```

Minimum description length:

`100 characters`

### Server actions

1.  verify Client owns booking
2.  verify booking is `ACTIVE`
3.  verify auto-release datetime has not passed
4.  validate reason
5.  validate description length
6.  create Dispute
7.  set Booking status to `DISPUTED`
8.  block auto-release
9.  trigger Partner WhatsApp notification
10. create admin operational alert

### Response

`201 Created`

``` json
{
  "success": true,
  "data": {
    "dispute_id": "dsp_123",
    "booking_id": "bkg_123",
    "status": "open"
  }
}
```

------------------------------------------------------------------------

# 9. Payment API

All monetary values in this section are integer minor units. The Client
never supplies an authoritative amount, currency, Partner, commission,
payment status, or booking state.

Payment API types are:

```text
full_payment  normal booking obligation
deposit       Travel Package server-stored deposit obligation
balance       Travel Package server-stored balance obligation
```

Internal payment states are `initializing`, `initialized`, `pending`,
`successful`, `failed`, `abandoned`, `expired`, `reversed`, `refunded`,
and `partially_refunded` where applicable. `provider_status` is retained
separately and never replaces the internal state.

For a verified normal full payment or Travel Package deposit, the stored
booking state changes from `pending_payment` to `upcoming`. `upcoming` is
the authoritative confirmed-booking state; this API does not introduce a
separate `confirmed` booking status.

## 9.1 Initialize Payment

`POST /api/v1/payments/initialize`

**Role:** Authenticated Client

**Required header:**

`Idempotency-Key`

### Request

``` json
{
  "booking_id": "bkg_123",
  "payment_type": "full_payment"
}
```

`payment_type` is optional. For a normal booking it may only be
`full_payment`. For a Travel Package, omitted or `full_payment` is mapped
by the backend to the stored `deposit` obligation; `balance` is permitted
only after the stored deposit is paid and the stored balance due date has
arrived. The Client cannot choose a deposit percentage or balance amount.

### Validation and idempotency

The backend:

1. authenticates the Client and verifies ownership of `booking_id`;
2. locks the booking/payment obligation;
3. requires a pending, unexpired payment window for full/deposit payment;
4. derives the amount/currency only from the booking snapshot or stored
   Travel Package schedule;
5. requires an active effective category commission-rate configuration;
6. creates or safely reuses one durable Paystack attempt/reference;
7. initializes Paystack using a server-built callback URL;
8. stores the provider initialization result privately; and
9. returns the original result for an identical idempotency replay.

Reusing the same `Idempotency-Key` with a different booking/type returns
`409 CONFLICT`. Repeated clicks must not create another active payable
attempt. Fields such as `amount`, `currency`, `partner_id`, `commission`,
or `status` in a Client payload are not authoritative and are ignored.

### Response

``` json
{
  "success": true,
  "data": {
    "payment": {
      "id": "pay_123",
      "booking_id": "bkg_123",
      "booking_reference": "HP-20260811-ABC123",
      "payment_type": "full_payment",
      "provider": "paystack",
      "provider_reference": "HPP6_...",
      "amount": 15000000,
      "currency": "NGN",
      "status": "initialized",
      "authorization_url": "https://checkout.paystack.com/..."
    },
    "authorization": {
      "url": "https://checkout.paystack.com/...",
      "access_code": "provider-access-code",
      "reference": "HPP6_..."
    },
    "reused": false
  }
}
```

The authorization URL/access code is checkout authorization data, not a
success signal. Paystack secret values and raw provider payloads are never
returned.

Typical errors: `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 NOT_FOUND`,
`409 CONFLICT` (expired/paid/not-due obligation or idempotency mismatch),
`409 COMMISSION_CONFIGURATION_REQUIRED` (no active rate; no payment attempt
is created), `422 VALIDATION_ERROR`, and `502 PAYMENT_PROVIDER_ERROR`.

------------------------------------------------------------------------

## 9.2 Read Payment

`GET /api/v1/payments/{payment_id}`

**Role:** Client owner; owning Partner may read safe booking payment
context. Admin uses the dedicated Admin payment routes below.

The response excludes provider payloads, provider access codes, and all
credentials.

### Response

``` json
{
  "success": true,
  "data": {
    "payment": {
      "id": "pay_123",
      "booking_id": "bkg_123",
      "payment_type": "full_payment",
      "status": "successful",
      "provider": "paystack",
      "provider_reference": "HPP6_...",
      "amount": 15000000,
      "currency": "NGN",
      "paid_at": "2026-08-11T12:34:56.000000Z",
      "booking": {
        "id": "bkg_123",
        "reference": "HP-20260811-ABC123",
        "status": "upcoming"
      }
    }
  }
}
```

## 9.3 Independently Verify Payment

`GET /api/v1/payments/{payment_id}/verify`

**Role:** Client owner

This endpoint is intended for a return/callback screen. The backend calls
Paystack using the stored provider reference and requires the returned
reference, successful provider status, amount, and currency to match the
server-owned payment obligation. A browser callback by itself cannot
confirm a booking.

### Successful response

``` json
{
  "success": true,
  "data": {
    "payment": { "id": "pay_123", "status": "successful", "amount": 15000000, "currency": "NGN" },
    "booking": { "id": "bkg_123", "status": "upcoming", "confirmed": true },
    "verified": true,
    "already_processed": false
  }
}
```

Provider failure or a reference/amount/currency mismatch records a failed
payment attempt and leaves the booking pending while its deadline remains
open. A repeated successful verification returns `already_processed: true`
and does not create another receipt, booking transition, or commission
record. If expiry won the row-lock race first, the provider success is kept
as a durable exception for operations but the cancelled booking is never
resurrected.

On the first matching successful verification, Phase 7 uses the same locked
transaction to require an active effective category commission rate, freeze
its ID and basis points in an immutable `CommissionRecord`, and create the
matching Partner **pending** ledger credit. There is no fallback/default
commission rate. A missing active rate rejects payment initialization before
a provider attempt is created. For a Travel Package, each successful deposit
and balance payment receives its own frozen commission/ledger fact. This is
not a payout or withdrawal: the Partner balance is still pending until
service completion and a release trigger.

Typical errors: `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 NOT_FOUND`,
and `502 PAYMENT_PROVIDER_ERROR`.

## 9.4 List My Payments

`GET /api/v1/me/payments`

**Role:** Authenticated Client

Optional query parameters: `status`, `booking_id`, `limit` (1–100), and
`offset`. Only payment attempts owned by the authenticated Client are
returned.

``` json
{
  "success": true,
  "data": {
    "items": [{ "id": "pay_123", "status": "successful", "amount": 15000000, "currency": "NGN" }],
    "pagination": { "limit": 25, "offset": 0, "count": 1 }
  }
}
```

------------------------------------------------------------------------

## 9.5 Paystack Webhook

`POST /api/v1/webhooks/paystack`

**Role:** Paystack only

Authentication is provider signature based, not application bearer
authentication.

### Request and security

The handler receives the exact raw JSON body and requires the
`X-Paystack-Signature` header. It validates an HMAC-SHA512 calculated from
that raw body using the configured Paystack webhook secret (or the Paystack
secret key when a separate webhook secret is not configured). Invalid or
malformed requests are rejected before any payment/booking transition.

### Expected handling

At minimum:

-   `charge.success`
-   `charge.failed`
-   `transfer.success`
-   `transfer.failed`
-   `transfer.reversed`

For a signed payment event, the backend persists a
`payment_webhook_events` row keyed by stable provider event identity,
re-fetches the transaction from Paystack, and runs the same locked
verification/confirmation transaction as manual verification. Replayed
events return an accepted duplicate result and have no additional financial
or booking effect. Unknown references are retained as failed operational
events with no domain transition.

### Response

``` json
{
  "success": true,
  "data": {
    "accepted": true,
    "duplicate": false,
    "processing_status": "processed",
    "payment_id": "pay_123"
  }
}
```

For a signed `charge.*` event, the webhook performs the same idempotent
verified-payment transaction as the manual verification endpoint. It can
therefore create the immutable CommissionRecord and its pending Partner ledger
credit exactly once. It does not release funds, create a PayoutEvent, or issue
a refund.

For a signed `transfer.*` event, Phase 8 writes a distinct
`transfer_webhook_events` record keyed by the stable provider event identity.
It locates the server-created transfer attempt by reference, re-verifies the
transfer with Paystack, and accepts a terminal result only when the
reference, amount, currency, and (when explicitly supplied by the provider)
recipient-code facts match the recorded withdrawal. Paystack can serialize
the recipient as a numeric provider ID, which is not compared to the stored
server-created recipient code.
`success` debits reserved funds once; `failed` and `reversed` restore them
once. Unknown references remain `unmatched`, duplicate events are harmless,
and a verification failure remains retryable with funds still reserved. See
section 15.7 for the complete transfer reconciliation contract.

------------------------------------------------------------------------

# 10. Travel Package Payment API

The PRD defines a different payment model for Travel Packages:

-   deposit collected at booking
-   balance collected on a Partner-defined date
-   balance payment link sent through WhatsApp and email

## 10.1 Get Travel Payment Schedule

`GET /api/v1/bookings/{booking_id}/payment-schedule`

**Role:** Client owner

### Response

``` json
{
  "success": true,
  "data": {
    "booking_id": "bkg_123",
    "status": "deposit_paid",
    "deposit_amount": 5000000,
    "balance_amount": 10000000,
    "balance_due_at": "2026-09-01T12:00:00Z",
    "payments": [
      {
        "payment_type": "deposit",
        "status": "successful",
        "amount": 5000000
      }
    ]
  }
}
```

------------------------------------------------------------------------

## 10.2 Initialize a Travel Package Balance

Use `POST /api/v1/payments/initialize` with:

```json
{ "booking_id": "bkg_123", "payment_type": "balance" }
```

The required idempotency, authorization, provider verification, and response
contract are exactly those of section 9.1. The backend permits this only for
an owned `upcoming` Travel Package with a verified stored deposit, a positive
stored balance, and a configured due date that has arrived. No balance
percentage, balance amount, or due-date consequence is inferred from React.

------------------------------------------------------------------------

# 11. Partner API

## 11.1 Partner Registration

`POST /api/v1/partner/register`

**Role:** Public

### Request

``` json
{
  "business_name": "Example Hospitality",
  "legal_name": "Example Hospitality Ltd",
  "phone": "+2348012345678",
  "email": "partner@example.com",
  "bank": {
    "bank_code": "000",
    "account_number": "0123456789",
    "account_name": "Example Hospitality Ltd"
  }
}
```

The exact onboarding fields are category/business dependent.

The backend must verify required Partner information before approval.

------------------------------------------------------------------------

## 11.2 Partner Profile

`GET /api/v1/partner/profile`

**Role:** Partner

### Response

``` json
{
  "success": true,
  "data": {
    "id": "prt_123",
    "status": "active",
    "business_name": "Example Hospitality",
    "contact": {},
    "bank_account": {
      "masked_account_number": "******6789",
      "account_name": "Example Hospitality Ltd"
    }
  }
}
```

Sensitive bank information must be masked for normal application
responses.

------------------------------------------------------------------------

## 11.3 Update Partner Profile

`PATCH /api/v1/partner/profile`

**Role:** Partner

### Request

``` json
{
  "business_name": "Updated Hospitality"
}
```

------------------------------------------------------------------------

# 12. Partner Listing API

## 12.1 List Partner Listings

`GET /api/v1/partner/listings`

**Role:** Partner

### Query

``` text
status
category_id
page
page_size
```

------------------------------------------------------------------------

## 12.2 Create Listing

`POST /api/v1/partner/listings`

**Role:** Partner

### Request

``` json
{
  "category_id": "cat_hotel",
  "name": "Luxury Hotel",
  "description": "Full listing description...",
  "city_id": "abuja",
  "neighborhood": "Maitama",
  "address": "...",
  "amenities": [],
  "pricing": {},
  "images": []
}
```

The category-specific validation rules must be enforced by the backend.

------------------------------------------------------------------------

## 12.3 Get Partner Listing

`GET /api/v1/partner/listings/{listing_id}`

**Role:** Partner owner

------------------------------------------------------------------------

## 12.4 Update Listing

`PATCH /api/v1/partner/listings/{listing_id}`

**Role:** Partner owner

### Request

Partial update of permitted listing fields.

Changes that affect approval-sensitive content may return the listing to
review according to product rules.

------------------------------------------------------------------------

## 12.5 Submit Listing for Review

`POST /api/v1/partner/listings/{listing_id}/submit-review`

**Role:** Partner owner

### Response

``` json
{
  "success": true,
  "data": {
    "listing_id": "lst_123",
    "status": "pending_review"
  }
}
```

The backend must reject submission when mandatory category-specific
content is incomplete.

------------------------------------------------------------------------

## 12.6 Listing Commission Preview

`GET /api/v1/partner/categories/{category_id}/commission`

**Role:** Partner

### Response

``` json
{
  "success": true,
  "data": {
    "category_id": "cat_boat_cruise",
    "commission_rate": 10
  }
}
```

This endpoint is informational.

The rate used for a confirmed booking is frozen in the booking's
CommissionRecord.

------------------------------------------------------------------------

# 13. Partner Booking API

## 13.1 Partner Bookings

`GET /api/v1/partner/bookings`

**Role:** Partner

### Query

``` text
status
start_date
end_date
listing_id
page
page_size
```

------------------------------------------------------------------------

## 13.2 Partner Booking Detail

`GET /api/v1/partner/bookings/{booking_id}`

**Role:** Partner owner

The response must not expose unnecessary Client-sensitive information.

------------------------------------------------------------------------

# 14. Partner Earnings API

## 14.1 Earnings Summary

`GET /api/v1/partner/earnings`

**Role:** Partner

### Response

``` json
{
  "success": true,
  "data": {
    "account_id": "ledacct_123",
    "pending": {
      "amount": 9000000,
      "currency": "NGN"
    },
    "available": {
      "amount": 25000000,
      "currency": "NGN"
    },
    "reserved_for_withdrawal": {
      "amount": 5000000,
      "currency": "NGN"
    },
    "updated_at": "2026-08-21T14:00:00.000000Z"
  }
}
```

The balances are cached, locked projections of append-only ledger entries;
they are not a mutable Partner profile balance. The endpoint exposes only
the authenticated active Partner's account.

------------------------------------------------------------------------

## 14.2 Earnings Transactions

`GET /api/v1/partner/earnings/transactions`

**Role:** Partner

### Query

``` text
type
from
to
page
page_size
```

### Response

``` json
{
  "success": true,
  "data": {
    "items": [{
      "id": "led_123",
      "type": "SETTLEMENT_AVAILABLE_CREDIT",
      "direction": "credit",
      "balance_bucket": "available",
      "booking_id": "bkg_123",
      "commission_record_id": "com_123",
      "payout_event_id": "pyo_123",
      "amount": 9000000,
      "currency": "NGN",
      "created_at": "2026-08-21T14:00:00.000000Z"
    }],
    "meta": { "page": 1, "page_size": 20, "total": 1, "total_pages": 1 }
  }
}
```

Supported filters are applied only to that Partner's ledger: `type`, `from`,
`to`, `page`, and `page_size`. Dates use `YYYY-MM-DD` and the type is the
stored append-only entry type (for example `BOOKING_PENDING_CREDIT`,
`SETTLEMENT_PENDING_DEBIT`, or `SETTLEMENT_AVAILABLE_CREDIT`).

------------------------------------------------------------------------

## 14.3 Booking Commission Breakdown

`GET /api/v1/partner/bookings/{booking_id}/commission`

**Role:** Partner owner

### Response

``` json
{
  "success": true,
  "data": {
    "booking_id": "bkg_123",
    "booking_status": "upcoming",
    "auto_release_at": "2026-08-21T14:00:00.000000Z",
    "commission_records": [{
      "id": "com_123",
      "payment_id": "pay_123",
      "commission_rate_id": "rate_123",
      "gross_booking_amount": 10000000,
      "commission_rate_basis_points": 1000,
      "commission_amount": 1000000,
      "partner_net_payout": 9000000,
      "currency": "NGN",
      "settlement": { "status": "pending_release", "expected_release_at": "2026-08-21T14:00:00.000000Z" }
    }],
    "totals": {
      "gross_booking_amount": 10000000,
      "commission_amount": 1000000,
      "partner_net_payout": 9000000,
      "currency": "NGN"
    }
  }
}
```

The commission rate ID and basis points are frozen for each successful
payment. Travel Package deposit and balance payments therefore appear as
separate immutable records rather than one guessed aggregate rate.

------------------------------------------------------------------------

## 14.4 Settlement Lifecycle Worker

The target-schema lifecycle worker is invoked by:

```text
php backend/cli/process-v1-booking-lifecycle.php --limit=100
```

Run it at least every 15 minutes. It locks each eligible booking, activates
an `upcoming` booking only after a successful payment has a reconciled
pending entitlement, and releases an `active` booking only when its stored
`auto_release_at` is due and no dispute exists. Phase 7 is fail-closed: a
later resolution must explicitly own its financial outcome rather than a
guessed dispute status unlocking funds. Release creates the
PayoutEvent and the paired `pending` debit / `available` credit in one
transaction before completion is recorded. It does not recalculate a stored
release timestamp from current category data.

For a narrowly scoped operational recovery/verification run, the CLI accepts
`--booking={booking_id}`; scheduled invocations omit it. The worker uses the
same financial release method as `Confirm Experience`, so concurrent worker
and Client commands cannot create a duplicate payout.

------------------------------------------------------------------------

# 15. Partner Payout Account and Withdrawal API

Phase 8 adds the canonical payout boundary. Only an authenticated **active
Partner** may use these endpoints; the API derives the Partner, ledger
account, currency, available balance, and payout recipient from server-owned
records. A browser cannot submit a ledger account, recipient code, provider
reference, amount in major units, or withdrawal status.

All amounts are positive integer minor units. The only currently configured
currency path is the server-owned payout currency. A payout account number is
encrypted at rest and is never returned by a Partner or Admin withdrawal
response.

## 15.1 List Partner Bank Accounts

`GET /api/v1/partner/bank-accounts`

**Role:** Active Partner

### Response

```json
{
  "success": true,
  "data": {
    "items": [{
      "id": "bnk_123",
      "provider": "paystack",
      "bank_code": "058",
      "bank_name": "Example Bank",
      "account_name": "Example Hospitality Ltd",
      "masked_account_number": "******6789",
      "verification_status": "verified",
      "verified_at": "2026-08-20T10:00:00.000000Z"
    }]
  }
}
```

`account_number_encrypted`, the plaintext account number, account-number
fingerprint, provider credentials, and raw Paystack responses are never
returned. The only usable withdrawal target is a `verified` account with a
server-created Paystack recipient code.

------------------------------------------------------------------------

## 15.2 Create and Verify a Partner Bank Account

`POST /api/v1/partner/bank-accounts`

**Role:** Active Partner

**Required header:**

`Idempotency-Key: <unique-key>`

### Request

```json
{
  "bank_code": "058",
  "account_number": "0123456789",
  "bank_name": "Example Bank"
}
```

`bank_name` is optional display input. The backend resolves the account name
and bank facts with Paystack and creates the recipient code itself. A Partner
cannot provide `account_name`, `provider_recipient_code`, verification state,
or a provider response as an authoritative value.

### Durable verification workflow

1. Validate and encrypt the account number; retain only a keyed fingerprint
   and last four digits for duplicate detection/display.
2. In one transaction, reserve the idempotency key and persist a
   `pending_provider` bank-account marker.
3. Outside that transaction, resolve the account with Paystack and create the
   Paystack transfer recipient.
4. In a second transaction, persist the returned recipient code and mark the
   account `verified`, or mark a known provider failure safely.

The durable `pending_provider` marker is intentional fail-closed handling of
an unknown external recipient outcome. A repeated key while that outcome is
pending returns `409 CONFLICT`; it must not create a second Paystack
recipient. Once a command completes, an identical key replays its stored
response. A known failed target may be retried only through a new, safe
command after the stored account state has been reviewed.

### Response

`201 Created`

```json
{
  "success": true,
  "data": {
    "bank_account": {
      "id": "bnk_123",
      "provider": "paystack",
      "bank_code": "058",
      "bank_name": "Example Bank",
      "account_name": "Example Hospitality Ltd",
      "masked_account_number": "******6789",
      "verification_status": "verified",
      "verified_at": "2026-08-20T10:00:00.000000Z"
    },
    "reused": false
  }
}
```

If payout-account encryption is not configured, the endpoint fails closed
with `503 BANK_ACCOUNT_CONFIGURATION_REQUIRED`; no account number is stored
or transmitted to Paystack.

------------------------------------------------------------------------

## 15.3 Withdrawal Eligibility

`GET /api/v1/partner/withdrawals/eligibility`

**Role:** Active Partner

### Response

```json
{
  "success": true,
  "data": {
    "eligible": true,
    "available_amount": 25000000,
    "currency": "NGN",
    "bank_accounts": [{
      "id": "bnk_123",
      "masked_account_number": "******6789",
      "verification_status": "verified"
    }]
  }
}
```

`eligible` means a positive reconciled available balance and at least one
verified payout target. `minimum_withdrawal` appears only when an active
server-side policy record contains a positive value. The PRD does not define
a minimum, so no default or permanent minimum is invented and the field is
omitted otherwise.

------------------------------------------------------------------------

## 15.4 Create Withdrawal

`POST /api/v1/partner/withdrawals`

**Role:** Active Partner

**Required header:**

`Idempotency-Key: <unique-key>`

### Request

```json
{
  "amount": 5000000,
  "bank_account_id": "bnk_123"
}
```

### Atomic server actions

1. Authenticate the active Partner and lock that Partner/currency ledger
   account.
2. Reconcile the locked cached projection to append-only ledger entries.
3. Verify the selected bank account belongs to the Partner and is verified
   with a server-created Paystack recipient.
4. Apply a configured minimum only when one exists, then require
   `available_balance >= amount`.
5. Create a `pending_review` withdrawal and exactly two reservation entries:
   `WITHDRAWAL_AVAILABLE_DEBIT` and `WITHDRAWAL_RESERVED_CREDIT`.
6. Update the locked cached projection from `available` to `reserved`, write
   audit/business-notification facts, store the idempotent response, and
   commit.

No Paystack transfer is called by this endpoint. A duplicate key with the
same request replays the original reservation result; a key reused with a
different request returns `409 CONFLICT`. Concurrent distinct withdrawal
requests cannot overdraw the account because they serialize on the locked
ledger account.

### Response

`201 Created`

```json
{
  "success": true,
  "data": {
    "withdrawal": {
      "id": "wd_123",
      "reference": "WD-20260820-ABC123",
      "amount": 5000000,
      "currency": "NGN",
      "status": "pending_review",
      "provider": "paystack",
      "provider_status": null,
      "transfer_reference": null,
      "bank_account": {
        "id": "bnk_123",
        "masked_account_number": "******6789",
        "verification_status": "verified"
      }
    },
    "balances": {
      "available": { "amount": 20000000, "currency": "NGN" },
      "pending": { "amount": 0, "currency": "NGN" },
      "reserved": { "amount": 5000000, "currency": "NGN" }
    }
  }
}
```

------------------------------------------------------------------------

## 15.5 List Partner Withdrawals

`GET /api/v1/partner/withdrawals`

**Role:** Active Partner

### Query

```text
status
limit                 1-100
offset
page / page_size      accepted aliases for limit/offset pagination
```

Only the authenticated Partner's own withdrawals are returned, newest first.

```json
{
  "success": true,
  "data": {
    "items": [{ "id": "wd_123", "status": "processing", "amount": 5000000, "currency": "NGN" }],
    "pagination": { "limit": 20, "offset": 0, "count": 1, "total": 1 }
  }
}
```

------------------------------------------------------------------------

## 15.6 Partner Withdrawal Detail

`GET /api/v1/partner/withdrawals/{withdrawal_id}`

**Role:** Owning active Partner

The response contains the safe withdrawal view, its own append-only ledger
entries, and safe transfer state. It includes no plaintext bank number,
encrypted account value, account fingerprint, raw webhook payload, provider
secret, or another Partner's record. `audit_history` is intentionally empty
for a Partner response.

### Withdrawal state machine and ledger effects

```text
pending_review --hold--> held --approve--> approved --> processing --> completed
       |                       |                              |
       +------reject-----------+                              +--failed|reversed
                                                                  |
                                                        reserved --> available
```

- `pending_review`: requested funds are already reserved.
- `held`: funds remain reserved; no provider call has occurred.
- `approved`: one durable Paystack transfer attempt has been queued.
- `processing`: Paystack instruction is accepted or its outcome is still
  being reconciled; funds remain reserved.
- `completed`: verified provider success creates one
  `WITHDRAWAL_RESERVED_DEBIT_PAYOUT` entry and debits reserved once.
- `failed` or `reversed`: verified provider result creates one
  `WITHDRAWAL_RESERVED_DEBIT_RETURN` and one
  `WITHDRAWAL_AVAILABLE_CREDIT_RETURN`, restoring the reservation exactly
  once.
- `rejected`: the same two return entries restore reserved funds before any
  transfer is initiated.

`provider_status`, `transfer_reference`, `transfer_code`, timestamps, and
failure/reversal reasons are provider/workflow facts, not Partner-controlled
input. A terminal status never creates another ledger effect when read,
reconciled, or web-hooked again.

------------------------------------------------------------------------

## 15.7 Transfer Webhook and Reconciliation Lifecycle

Paystack transfer events use the existing canonical webhook:

`POST /api/v1/webhooks/paystack`

The exact raw request body and `X-Paystack-Signature` are verified with
HMAC-SHA512 before a `transfer.*` event is considered. A valid event is
persisted in the dedicated transfer-webhook event store using a stable
provider event identity. The handler then re-fetches Paystack transfer facts
by the server-owned reference and requires the reference, amount, currency,
and an explicit provider recipient code (when supplied) to match the
withdrawal before any financial transition. A numeric provider recipient ID
is not treated as the stored recipient code.

- Unknown transfer references are retained as `unmatched` operational facts;
  they do not move money.
- A duplicate provider event returns the recorded accepted result.
- Temporary provider verification failures remain retryable and leave the
  withdrawal reserved/processing.
- Only verified `success`, `failed`, or `reversed` facts can produce the
  terminal ledger effects described above.

```json
{
  "success": true,
  "data": {
    "accepted": true,
    "duplicate": false,
    "processing_status": "processed",
    "withdrawal_id": "wd_123",
    "status": "completed"
  }
}
```

The retry-safe reconciliation worker is:

```text
php backend/cli/process-v1-withdrawal-reconciliation.php --limit=100
```

`--limit` is bounded to 1-500. `--withdrawal={withdrawal_id}` is an
operational recovery/test scope, not a user-facing API. Deployment scheduling
must invoke the worker recurrently and monitor failures; the PRD does not
define a withdrawal polling interval, so this documentation does not invent
one. The worker starts queued attempts and re-verifies `initiating`,
`processing`, or `unknown` attempts. It is safe to run more than once.

------------------------------------------------------------------------

## 15.8 Withdrawal Errors, Audit, and Notification Facts

Typical errors are `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 NOT_FOUND`,
`409 WITHDRAWAL_NOT_ELIGIBLE` (insufficient available balance, invalid state,
unverified target, or configured-policy failure), `409 CONFLICT` (idempotency
mismatch or a pending external bank verification), `422 VALIDATION_ERROR`,
`429 RATE_LIMITED`, `502 PAYMENT_PROVIDER_ERROR`, and
`503 BANK_ACCOUNT_CONFIGURATION_REQUIRED`. Withdrawal and payout-account
creation use deployment-configured financial-command rate limits; the PRD does
not prescribe a numerical limit.

Every withdrawal request, hold, approval, rejection, transfer initiation,
provider terminal result, and Admin detail view is appended to the audit
trail with safe request context. Phase 8 also writes deduplicated, durable
business notification facts such as `withdrawal.requested`,
`withdrawal.approved`, `withdrawal.held`, `withdrawal.rejected`,
`withdrawal.processing`, `withdrawal.successful`, `withdrawal.failed`, and
`withdrawal.reversed`. These records are **not** channel delivery attempts:
Phase 9 owns WhatsApp/email delivery, retries, and fallback and must not
change the committed withdrawal outcome.

------------------------------------------------------------------------

# 16. Client Wishlist API

## 16.1 List Wishlist

`GET /api/v1/me/wishlist`

**Role:** Client

------------------------------------------------------------------------

## 16.2 Add Listing

`POST /api/v1/me/wishlist`

### Request

``` json
{
  "listing_id": "lst_123"
}
```

------------------------------------------------------------------------

## 16.3 Remove Listing

`DELETE /api/v1/me/wishlist/{listing_id}`

------------------------------------------------------------------------

# 17. Client Support Chat API

The PRD requires persistent in-app Client support chat with shared
history.

## 17.1 Get My Support Conversations

`GET /api/v1/me/support/conversations`

**Role:** Client

------------------------------------------------------------------------

## 17.2 Create Support Conversation

`POST /api/v1/me/support/conversations`

**Role:** Client

### Request

``` json
{
  "subject": "Question about my booking",
  "message": "I need help with my booking."
}
```

------------------------------------------------------------------------

## 17.3 Get Conversation

`GET /api/v1/me/support/conversations/{conversation_id}`

------------------------------------------------------------------------

## 17.4 Send Message

`POST /api/v1/me/support/conversations/{conversation_id}/messages`

### Request

``` json
{
  "message": "Please help me with this booking."
}
```

------------------------------------------------------------------------

## 17.5 Upload Chat Attachment

`POST /api/v1/me/support/conversations/{conversation_id}/attachments`

**Content-Type:** multipart/form-data

The PRD requires document capability for the support chat.

------------------------------------------------------------------------

# 18. Partner/Admin High-Ticket Chat API

## 18.1 Partner Conversations

`GET /api/v1/partner/conversations`

**Role:** Partner

------------------------------------------------------------------------

## 18.2 Conversation Detail

`GET /api/v1/partner/conversations/{conversation_id}`

**Role:** Partner participant

------------------------------------------------------------------------

## 18.3 Send Message

`POST /api/v1/partner/conversations/{conversation_id}/messages`

------------------------------------------------------------------------

## 18.4 Attachment

`POST /api/v1/partner/conversations/{conversation_id}/attachments`

**Content-Type:** multipart/form-data

The Admin PRD states that Full Admin can initiate vendor-admin
high-ticket conversations.

------------------------------------------------------------------------

# 19. Admin Authentication API

The Admin Dashboard uses a separate authentication infrastructure and
must not share authentication infrastructure with Client/Partner apps.

## 19.1 Admin Login

`POST /api/v1/admin/auth/login`

**Role:** Public admin login endpoint

### Request

``` json
{
  "email": "admin@example.com",
  "password": "SecurePassword123"
}
```

### Response

``` json
{
  "success": true,
  "data": {
    "access_token": "<token>",
    "refresh_token": "<token>",
    "admin": {
      "id": "adm_123",
      "role": "full_admin"
    }
  }
}
```

------------------------------------------------------------------------

## 19.2 Admin Current User

`GET /api/v1/admin/me`

------------------------------------------------------------------------

## 19.3 Admin Logout

`POST /api/v1/admin/auth/logout`

------------------------------------------------------------------------

# 20. Admin Dashboard API

## 20.1 Command Centre Summary

`GET /api/v1/admin/dashboard`

**Role:** Full Admin, Customer Support

Optional query: `from` and `to` as UTC `YYYY-MM-DD` dates. The date range
applies to booking and payment-derived metrics; account-balance projections
remain current authoritative ledger facts.

### Response

``` json
{
  "success": true,
  "data": {
    "metrics": {
      "clients": 42,
      "partners": 14,
      "active_partners": 11,
      "pending_partner_approvals": 2,
      "listings_pending_review": 4,
      "active_listings": 35,
      "bookings": 20,
      "upcoming_bookings": 8,
      "active_bookings": 2,
      "completed_bookings": 7,
      "cancelled_bookings": 3,
      "disputed_bookings": 1,
      "failed_notification_deliveries": 0,
      "system_alerts": 1,
      "gross_booking_value": 12500000,
      "successful_payments": 12500000,
      "platform_commission": 1250000,
      "partner_pending_earnings": 400000,
      "partner_available_earnings": 900000,
      "withdrawals_pending_review": 1,
      "withdrawals_processing": 0,
      "failed_withdrawals": 0
    },
    "alerts": [],
    "pending_actions": [],
    "recent_activity": [],
    "date_range": { "from": "2026-08-01", "to": "2026-08-31" },
    "capabilities": ["dashboard.read", "partners.read"]
  }
}
```

All money is integer minor units. Customer Support receives the operational
metrics and alerts it is permitted to inspect; finance-only metrics are not
returned to that role.

------------------------------------------------------------------------

# 21. Admin Partner API

## 21.1 List Partners

`GET /api/v1/admin/partners`

**Role:** Full Admin, Customer Support

### Query

``` text
status
city_id
search
page
page_size
```

------------------------------------------------------------------------

## 21.2 Partner Detail

`GET /api/v1/admin/partners/{partner_id}`

**Role:** Full Admin, Customer Support

Sensitive bank account numbers must remain masked for Customer Support.

------------------------------------------------------------------------

## 21.3 Approve Partner

`POST /api/v1/admin/partners/{partner_id}/approve`

**Role:** Full Admin

### Request

``` json
{
  "note": "Approved after verification."
}
```

The existing onboarding checklist remains configurable; this endpoint does not
invent a second onboarding-completion or bank-account gate. Approval is only
valid from `pending_approval` and records the approving Admin in immutable
history.

------------------------------------------------------------------------

## 21.4 Suspend Partner

`POST /api/v1/admin/partners/{partner_id}/suspend`

**Role:** Full Admin

### Request

``` json
{
  "reason": "Operational violation"
}
```

------------------------------------------------------------------------

## 21.5 Reactivate Partner

`POST /api/v1/admin/partners/{partner_id}/reactivate`

**Role:** Full Admin

### Request

``` json
{
  "reason": "Reinstated after the documented review."
}
```

Suspend and reactivate require `Idempotency-Key`. Suspending revokes active
Partner sessions, blocks protected Partner operations, and removes public
discovery through the active-Partner listing policy. It does not delete
bookings, immutable commission records, ledger history, payouts, or owed
funds.

Only an already approved Partner may be suspended and subsequently
reactivated. Pending approval is not an operational account state that can be
reactivated into approval.

------------------------------------------------------------------------

# 21A. Admin Client Operations API

## 21A.1 List and inspect Clients

`GET /api/v1/admin/clients`

`GET /api/v1/admin/clients/{client_id}`

**Role:** Full Admin, Customer Support

List query: `status`, `search`, `page`, `page_size`. Detail returns safe
identity, booking, dispute, and notification context. Full Admin receives the
payment-history section; Customer Support never receives payment amount,
provider, or provider-reference facts through this route. Neither role ever
receives password hashes, OTPs, tokens, or session secrets.

## 21A.2 Suspend or Reactivate a Client

`POST /api/v1/admin/clients/{client_id}/suspend`

`POST /api/v1/admin/clients/{client_id}/reactivate`

**Role:** Full Admin

**Required header:** `Idempotency-Key`

```json
{ "reason": "Documented account review outcome." }
```

Both commands require a reason. Suspension is valid only from `active`,
revokes Client sessions, and preserves historical booking/payment/dispute
facts. Reactivation is valid only from `suspended`; a separately blocked
state is not silently bypassed by this operational control.

------------------------------------------------------------------------

# 22. Admin Listing API

## 22.1 Pending Listings

`GET /api/v1/admin/listings`

**Role:** Full Admin, Customer Support

### Query

``` text
status
category_id
partner_id
city_id
search
page
page_size
```

------------------------------------------------------------------------

## 22.2 Listing Detail

`GET /api/v1/admin/listings/{listing_id}`

------------------------------------------------------------------------

## 22.3 Approve Listing

`POST /api/v1/admin/listings/{listing_id}/approve`

**Role:** Full Admin

### Request

``` json
{
  "note": "Listing meets requirements."
}
```

------------------------------------------------------------------------

## 22.4 Request Changes

`POST /api/v1/admin/listings/{listing_id}/request-changes`

**Role:** Full Admin

### Request

``` json
{
  "reason": "Please add check-in instructions and generator details."
}
```

------------------------------------------------------------------------

## 22.5 Suspend Listing

`POST /api/v1/admin/listings/{listing_id}/suspend`

**Role:** Full Admin

`reason` is required. A suspended listing is hidden from public discovery.

## 22.6 Reactivate Listing

`POST /api/v1/admin/listings/{listing_id}/reactivate`

**Role:** Full Admin

`reason` is required. Only a suspended listing can be reactivated. This
reuses the Phase 4 listing lifecycle and records both review history and audit
facts; it is not a second listing-review workflow.

------------------------------------------------------------------------

# 23. Admin Booking API

## 23.1 List Bookings

`GET /api/v1/admin/bookings`

**Role:** Full Admin, Customer Support

### Query

``` text
status
partner_id
client_id
listing_id
city_id
from
to
search
page
page_size
```

------------------------------------------------------------------------

## 23.2 Booking Detail

`GET /api/v1/admin/bookings/{booking_id}`

**Role:** Full Admin, Customer Support

The resolution workspace must include relevant:

-   Client context
-   booking
-   financial record
-   Partner communications

------------------------------------------------------------------------

## 23.3 List Payment Operations

`GET /api/v1/admin/payments`

**Role:** Full Admin

This is the payment-collection operations surface only. It does not expose
Partner settlement, ledger, payout, transfer, or withdrawal controls.

Optional query parameters:

```text
status
payment_type        full_payment | deposit | balance
reference
booking_reference
client
partner
search              reference, booking, Client, or Partner text
date_from / date_to YYYY-MM-DD
from / to           aliases for date_from / date_to
limit               1–100
offset
```

```json
{
  "success": true,
  "data": {
    "items": [{
      "id": "pay_123",
      "booking_id": "bkg_123",
      "booking_reference": "HP-20260811-ABC123",
      "payment_type": "full_payment",
      "provider": "paystack",
      "provider_reference": "HPP6_...",
      "amount": 15000000,
      "currency": "NGN",
      "status": "successful",
      "provider_status": "success",
      "initialized_at": "2026-08-11T12:00:00.000000Z",
      "paid_at": "2026-08-11T12:01:00.000000Z",
      "client": { "id": "usr_123", "email": "client@example.com" },
      "partner": { "id": "par_123", "display_name": "Example Hospitality" }
    }],
    "pagination": { "limit": 25, "offset": 0, "count": 1 }
  }
}
```

## 23.4 Payment Operations Detail

`GET /api/v1/admin/payments/{payment_id}`

**Role:** Full Admin

Returns the safe payment operation record plus associated webhook processing
state: event type, signature-verification result, processing status, time,
and sanitized processing error. Raw provider payloads and secrets are not
returned. Admin access is recorded in the audit trail.

Typical errors: `401 UNAUTHENTICATED`, `403 FORBIDDEN`, and `404 NOT_FOUND`.

------------------------------------------------------------------------

## 23.5 Booking Financial Trail

`GET /api/v1/admin/bookings/{booking_id}/financial`

**Role:** Full Admin only

Returns a read-only, audited financial trace for the booking: its frozen
CommissionRecords/rate provenance, PayoutEvents, append-only Partner ledger
entries, and the relevant cached Partner balance projection. It is designed
for operational reconciliation, not for payout or withdrawal control.

```json
{
  "success": true,
  "data": {
    "booking": { "id": "bkg_123", "status": "completed", "gross_amount": 10000000, "currency": "NGN" },
    "commission": { "commission_records": [{ "id": "com_123", "commission_rate_id": "rate_123", "partner_net_payout": 9000000 }] },
    "payout_events": [{ "id": "pyo_123", "amount": 9000000, "trigger_type": "auto_release", "status": "released" }],
    "ledger_entries": [{ "id": "led_123", "type": "SETTLEMENT_AVAILABLE_CREDIT", "bucket": "available", "amount": 9000000 }],
    "partner_balance": { "currency": "NGN", "available": 9000000, "pending": 0, "reserved": 0 }
  }
}
```

Provider authorization URLs, provider payloads, payment secrets, and bank
account details are never returned. Client, Partner, and Customer Support
sessions receive no financial-trail data.

------------------------------------------------------------------------

# 24. Admin Dispute API

## 24.1 List Disputes

`GET /api/v1/admin/disputes`

**Role:** Full Admin, Customer Support

### Query

``` text
status
priority
reason
partner_id
page
page_size
```

------------------------------------------------------------------------

## 24.2 Dispute Detail

`GET /api/v1/admin/disputes/{dispute_id}`

**Role:** Full Admin, Customer Support

------------------------------------------------------------------------

## 24.3 Add Dispute Note

`POST /api/v1/admin/disputes/{dispute_id}/notes`

**Role:** Full Admin, Customer Support

### Request

``` json
{
  "note": "Client contacted through support chat."
}
```

------------------------------------------------------------------------

## 24.4 Resolve Dispute

`POST /api/v1/admin/disputes/{dispute_id}/resolve`

**Role:** Full Admin only

**Header:**

`Idempotency-Key`

### Request

Option A:

``` json
{
  "resolution_type": "full_partner_release",
  "resolution_note": "Service was delivered as agreed and evidence supports release."
}
```

Option B:

``` json
{
  "resolution_type": "partial_partner_release",
  "partner_release_amount": 7000000,
  "client_refund_amount": 3000000,
  "resolution_note": "Partial service delivery was established."
}
```

Option C:

``` json
{
  "resolution_type": "full_client_refund",
  "client_refund_amount": 10000000,
  "resolution_note": "Service materially failed to meet the agreed description."
}
```

The Admin PRD requires a minimum 100-character internal resolution note.

### Server behavior

The Dispute Engine must:

1.  verify dispute is unresolved
2.  verify actor is Full Admin
3.  validate amounts
4.  execute applicable Paystack transfer/refund workflow
5.  update dispute and booking states
6.  create audit entry
7.  trigger notifications

The Admin PRD describes this as an atomic operational workflow. Because
external Paystack calls cannot participate in a database transaction,
the implementation must use a durable workflow/compensation mechanism to
achieve the required business guarantee rather than pretending an
external API call can roll back like a database query.

### Response

``` json
{
  "success": true,
  "data": {
    "dispute_id": "dsp_123",
    "status": "resolved",
    "booking_status": "completed",
    "resolution_type": "partial_partner_release"
  }
}
```

------------------------------------------------------------------------

# 25. Admin Withdrawal API

All endpoints in this section require an authenticated **Full Admin**. A
Customer Support session, Partner session, or Client session receives
`403 FORBIDDEN`. Withdrawal decisions remain server-authorized financial
commands. Full withdrawal detail access and every state-changing command are
appended to the audit trail. These APIs return only safe bank-account data
(masked number and verified account facts); they do not expose a plaintext
account number.

## 25.1 List Withdrawals

`GET /api/v1/admin/withdrawals`

**Role:** Full Admin

### Query

``` text
status
partner_id
from
to
limit                 1-100
offset
page / page_size      accepted aliases for limit/offset pagination
```

```json
{
  "success": true,
  "data": {
    "items": [{
      "id": "wd_123",
      "reference": "WD-20260820-ABC123",
      "partner": { "id": "prt_123", "display_name": "Example Hospitality" },
      "amount": 5000000,
      "currency": "NGN",
      "status": "pending_review",
      "bank_account": { "masked_account_number": "******6789", "verification_status": "verified" }
    }],
    "pagination": { "limit": 20, "offset": 0, "count": 1, "total": 1 }
  }
}
```

------------------------------------------------------------------------

## 25.2 Withdrawal Detail

`GET /api/v1/admin/withdrawals/{withdrawal_id}`

**Role:** Full Admin

Returns the safe withdrawal view, Partner display identity, safe bank target,
provider transfer facts, append-only withdrawal ledger entries, and audit
history. The provider recipient code may be shown to Full Admin for transfer
operations; the encrypted/plain account number and raw provider payload are
not returned. Reading this detail creates the `withdrawal.admin_viewed`
audit event.

------------------------------------------------------------------------

## 25.3 Approve Withdrawal

`POST /api/v1/admin/withdrawals/{withdrawal_id}/approve`

**Role:** Full Admin

**Header:**

`Idempotency-Key`

### Response

``` json
{
  "success": true,
  "data": {
    "withdrawal": {
      "id": "wd_123",
      "status": "processing",
      "transfer_reference": "hpwd_..."
    },
    "transfer": {
      "reference": "hpwd_...",
      "status": "processing"
    }
  }
}
```

Approval is allowed only from `pending_review` or `held`. It creates at most
one durable Paystack transfer attempt and invokes the transfer workflow after
that transaction commits. If the provider outcome is uncertain, the command
returns a reserved `processing` withdrawal with
`transfer_pending_reconciliation: true`; reconciliation, not another Admin
approval, determines the final outcome. Repeating a completed approval key
replays its original result. A second Admin approval after the state has
advanced returns the existing withdrawal with `already_approved: true` and
does not create another transfer attempt.

------------------------------------------------------------------------

## 25.4 Hold Withdrawal

`POST /api/v1/admin/withdrawals/{withdrawal_id}/hold`

**Role:** Full Admin

**Required header:**

`Idempotency-Key: <unique-key>`

### Request

``` json
{
  "reason": "Bank account discrepancy requires clarification."
}
```

`reason` is required. Holding a withdrawal keeps its reserved funds intact;
the Full Admin may later approve or reject it. It must not contact Paystack.

------------------------------------------------------------------------

## 25.5 Reject Withdrawal

`POST /api/v1/admin/withdrawals/{withdrawal_id}/reject`

**Role:** Full Admin

**Required header:**

`Idempotency-Key: <unique-key>`

### Request

``` json
{
  "reason": "Withdrawal failed compliance review."
}
```

`reason` is required. Rejection is allowed only before a transfer is
initiated (`pending_review` or `held`). In the same locked transaction it
creates the one allowed reserved-debit return and available-credit return
ledger entries, updates the cached projection, records the Admin audit event,
and marks the withdrawal `rejected`. A duplicate rejection is harmless;
rejection after transfer initiation is rejected rather than guessing a
provider reversal.

Rejected, failed, and reversed withdrawals restore reserved funds exactly
once. A completed withdrawal cannot be held, rejected, or re-approved.

------------------------------------------------------------------------

# 26. Admin Commission API

## 26.1 List Commission Rates

`GET /api/v1/admin/commission-rates`

**Role:** Full Admin

------------------------------------------------------------------------

## 26.2 Create a Versioned Commission Rate

`POST /api/v1/admin/commission-rates`

**Role:** Full Admin

**Required header:**

`Idempotency-Key`

### Request

``` json
{
  "category_id": "cat_123",
  "rate": 12.5,
  "effective_from": "2026-09-01T00:00:00Z",
  "effective_to": null,
  "reason": "Seasonal commission policy approved by Finance."
}
```

`rate` is a percentage with at most two decimal places (0–100), or callers
may send `rate_basis_points` (0–10000). `effective_from` cannot be in the
past and no active/effective/scheduled interval may overlap. A new future
rate may close a preceding interval at its exact effective time; this creates
an audited version rather than overwriting financial history.

The response is `{ "rate": { ... } }` and includes `rate_basis_points`, the
percentage `rate`, effective interval, configured status, and derived
`effective` / `scheduled` / `ended` state.

## 26.3 Retire / End a Future Commission Rate

`POST /api/v1/admin/commission-rates/{rate_id}/retire`

**Role:** Full Admin

**Required header:** `Idempotency-Key`

```json
{
  "effective_to": "2026-10-01T00:00:00Z",
  "reason": "The category is leaving the promotional rate."
}
```

The end time cannot be in the past, precede the rate start, or extend an
already bounded interval. Confirmed booking CommissionRecords retain their
frozen rate provenance and are never recalculated.

------------------------------------------------------------------------

# 27. Admin Support Chat API

## 27.1 List Support Conversations

`GET /api/v1/admin/support/conversations`

**Role:** Full Admin, Customer Support

### Query

``` text
status
assigned_to
search
page
page_size
```

------------------------------------------------------------------------

## 27.2 Conversation Detail

`GET /api/v1/admin/support/conversations/{conversation_id}`

------------------------------------------------------------------------

## 27.3 Assign Conversation

`POST /api/v1/admin/support/conversations/{conversation_id}/assign`

**Role:** Full Admin, Customer Support as permitted

### Request

``` json
{
  "admin_user_id": "adm_123"
}
```

------------------------------------------------------------------------

## 27.4 Send Support Message

`POST /api/v1/admin/support/conversations/{conversation_id}/messages`

### Request

``` json
{
  "message": "Hello, how can we help?"
}
```

------------------------------------------------------------------------

## 27.5 Close Conversation

`POST /api/v1/admin/support/conversations/{conversation_id}/close`

------------------------------------------------------------------------

# 28. Admin Partner Chat API

## 28.1 List Partner Conversations

`GET /api/v1/admin/partner-conversations`

**Role:** Full Admin

------------------------------------------------------------------------

## 28.2 Start High-Ticket Conversation

`POST /api/v1/admin/partner-conversations`

**Role:** Full Admin

### Request

``` json
{
  "partner_id": "prt_123",
  "subject": "Private Jet Booking Follow-up"
}
```

------------------------------------------------------------------------

## 28.3 Send Partner Message

`POST /api/v1/admin/partner-conversations/{conversation_id}/messages`

------------------------------------------------------------------------

# 29. Notification API

Notification delivery is system-generated and asynchronous. A domain command
commits its own business result and a durable notification fact; it never waits
for Meta or email provider I/O. WhatsApp is primary. Email is fallback only
after WhatsApp reaches final delivery failure, so a normal successful WhatsApp
delivery does not produce a duplicate email.

Each new Phase 9 notification fact stores the active application template
version selected at event time. Delivery and an eligible Admin resend resolve
that exact version, rather than silently switching a historic event to a newer
template version.

All notification list responses are safe projections. They never return a raw
phone/email recipient snapshot, OTP, bank data, provider credential, raw
provider payload, or internal exception trace. Client and Partner history
returns only records owned by the authenticated actor. Only an authorized
Admin notification detail response may include the sanitized delivery attempt
history.

Notification lifecycle values are:

``` text
queued
processing
sent
delivered
read
fallback_email_sent
failed
suppressed
```

Attempt lifecycle values are:

``` text
queued
sending
sent
delivered
read
failed
```

`fallback_email_sent` means an email fallback was accepted for delivery; it is
not a claim that the recipient has opened the email. `sent` means provider
acceptance/dispatch, while `delivered` and `read` require a matching provider
receipt where supported.

### OTP delivery safety

`auth.otp_requested` is a delivery-only event. The OTP record remains hash-only
with its existing expiry and verification-attempt rules. A plaintext OTP may be
made available transiently to the provider template renderer but is never
returned by notification/history APIs, persisted as plaintext in ordinary
notification payloads or attempt metadata, logged, included in audit context,
or exposed to React. It exists only as encrypted private server-side renderer
context for the short delivery window. If a WhatsApp OTP attempt reaches final
failure, the configured approved fallback may be used without weakening OTP
verification, expiry, rate limiting, or attempt limits. The encrypted private
renderer context is cleared after provider acceptance, successful fallback, or
terminal failure; the scheduled worker also purges any remaining `auth_otp`
context after the linked OTP verification expires.

The delivery worker sends an OTP only while its linked
`user_otp_verifications` record is unverified and unexpired. Verifying the OTP
suppresses any queued/processing delivery and clears its encrypted renderer
context atomically. `auth_otp` notifications cannot be resent by an Admin,
which prevents a manual delivery action from extending or recreating OTP
validity.

## 29.1 Client Notification History

`GET /api/v1/me/notifications`

**Role:** Client

### Query parameters

``` text
status              optional notification status
event_type          optional stable business event type
from                optional ISO-8601 UTC inclusive timestamp
to                  optional ISO-8601 UTC inclusive timestamp
page                optional positive integer
page_size           optional positive integer, capped by the server
```

### Response

``` json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "01JNOTIFICATION...",
        "event_type": "booking.confirmed",
        "template_key": "booking_confirmed",
        "template_version": 1,
        "status": "delivered",
        "primary_channel": "whatsapp",
        "entity_type": "booking",
        "entity_id": "01JBOOKING...",
        "created_at": "2026-08-13T09:00:00.000000Z",
        "sent_at": "2026-08-13T09:00:05.000000Z",
        "delivered_at": "2026-08-13T09:00:12.000000Z",
        "read_at": null
      }
    ],
    "pagination": {"page": 1, "page_size": 25, "total": 1}
  }
}
```

------------------------------------------------------------------------

## 29.2 Partner Notification History

`GET /api/v1/partner/notifications`

**Role:** Partner

The response and filter contract are the same safe projection as section
29.1. The server resolves ownership through the authenticated Partner; a
Client cannot obtain these records and a Partner cannot substitute another
Partner ID.

------------------------------------------------------------------------

## 29.3 Mark Notification Read

`POST /api/v1/notifications/{notification_id}/read`

**Role:** Authenticated owner

This accepts no body. The server sets `read_at` only for a Client/Partner who
owns the notification. It is idempotent: repeating the request retains the
original read timestamp and returns the current safe projection.

### Response

``` json
{
  "success": true,
  "data": {
    "notification": {
      "id": "01JNOTIFICATION...",
      "status": "read",
      "read_at": "2026-08-13T09:04:00.000000Z"
    }
  }
}
```

------------------------------------------------------------------------

## 29.4 Admin Notification Delivery Logs

`GET /api/v1/admin/notifications`

**Role:** Full Admin, Customer Support where permitted

### Query parameters

``` text
status              optional notification status
channel             optional primary/delivery channel
event_type          optional stable business event type
recipient_type      optional client|partner|admin
recipient           optional recipient name fragment or opaque ID
from                optional ISO-8601 UTC inclusive timestamp
to                  optional ISO-8601 UTC inclusive timestamp
page                optional positive integer
page_size           optional positive integer, capped by the server
```

The collection response contains no raw recipient contact snapshot or raw
provider data. It may expose the recipient type/opaque ID, event/template,
related entity, notification status, attempt count, fallback state, and safe
timestamps/last-failure summary required for operations.

------------------------------------------------------------------------

## 29.5 Admin Notification Detail

`GET /api/v1/admin/notifications/{notification_id}`

**Role:** Full Admin, Customer Support where permitted

The detail is the only API response that includes the sanitized attempt
history. Each attempt may contain its channel, provider, attempt number,
state, provider message ID when available, queued/sent/delivered/failed
timestamps, safe provider status/metadata, and a safe failure reason. It must
not include access tokens, webhook secrets, plaintext OTPs, full bank details,
or raw provider request/response bodies.

------------------------------------------------------------------------

## 29.6 Resend Notification

`POST /api/v1/admin/notifications/{notification_id}/resend`

**Role:** Full Admin

**Header:**

`Idempotency-Key`

The resend creates a new delivery attempt against the same business
notification rather than duplicating the underlying business event. Reusing
the same `Idempotency-Key` replays the prior resend result. The server rejects
unowned/nonexistent records, records the Admin audit context, and queues the
new attempt after its transaction commits.

OTP notifications are deliberately ineligible for this operation. A resend
never creates, extends, or reuses an OTP verification value.

The command returns `202 Accepted` with the safe notification projection, the
new queued attempt (`id`, `channel`, `attempt_number`, `status`), and a
`reused` flag. It does not synchronously call Meta or the email provider.

------------------------------------------------------------------------

# 30. WhatsApp Provider Webhook API

## 30.1 Meta Verification Handshake

`GET /api/v1/webhooks/whatsapp`

**Role:** Meta verification only

The endpoint validates Meta's subscription mode and the configured verify
token, then returns the supplied challenge as plain text. Invalid verification
returns a canonical `403` response and creates no notification/attempt.

## 30.2 Meta Delivery Status Webhook

`POST /api/v1/webhooks/whatsapp`

**Role:** Provider signature/authentication

Used for delivery/status events where supported.

The raw request must include `X-Hub-Signature-256` and pass the configured
Meta app-secret HMAC-SHA-256 authenticity check before processing. The handler
deduplicates a stable provider event identity, finds an existing attempt by
provider message ID, and maps recognized status receipts only.
Duplicate/out-of-order receipts are idempotent; an unknown provider message ID
is safely recorded for operations without creating a new notification or
changing any booking, payment, settlement, or withdrawal. Only a sanitized
fact (`message_id`, status, timestamp, and payload hash) is stored; the raw
request is not retained.

Possible states:

``` text
queued
sent
delivered
read
failed
```

Provider-specific statuses must be mapped into the internal Notification
Attempt model.

------------------------------------------------------------------------

# 31. Template Registry Boundary

Phase 9 stores versioned application templates in the server-side
`notification_templates` registry. Application template key/version and a
Meta provider template name/identifier are distinct values. A live Meta send
uses only an eligible configured provider template (apart from the separately
configured approved OTP template); an absent/invalid provider template is a
delivery failure that follows the bounded fallback path.

The ordered target migration chain is `001` through `015`.
`014_phase9_notification_delivery.sql` seeds the base registry and durable
delivery tables. `015_phase9_booking_cancellation_template.sql` adds version 1
email and WhatsApp rows for `booking_cancelled`; the payment-expiry booking
cancellation flow emits `booking.cancelled` separately to the affected Client
and Partner using deterministic recipient-scoped event keys.

Phase 9 intentionally exposes no template CRUD API. Template approval and
provider-name administration belong to the later Admin Operations surface.
React must never send a Meta template name, access token, phone-number ID, or
provider credential.

------------------------------------------------------------------------

# 32. Refund API

## 32.1 Client Refund Status

`GET /api/v1/bookings/{booking_id}/refund`

**Role:** Client owner

------------------------------------------------------------------------

## 32.2 Admin Initiate Refund

`POST /api/v1/admin/bookings/{booking_id}/refund`

**Role:** Full Admin

**Header:**

`Idempotency-Key`

### Request

``` json
{
  "amount": 10000000,
  "reason": "Dispute resolution"
}
```

The amount must be validated against the refundable amount.

The Admin PRD gives Full Admin authority to initiate Paystack refunds
for disputed bookings.

------------------------------------------------------------------------

# 33. High-Ticket Follow-Up API

The Admin PRD requires structured follow-up for high-value bookings such
as private jet charters, event halls, and group travel packages.

## 33.1 List Follow-Ups

`GET /api/v1/admin/follow-ups`

**Role:** Full Admin, Customer Support

### Query

``` text
status
due_before
assigned_to
page
page_size
```

------------------------------------------------------------------------

## 33.2 Complete Follow-Up

`POST /api/v1/admin/follow-ups/{follow_up_id}/complete`

### Request

``` json
{
  "note": "Client and Partner confirmed all arrangements."
}
```

------------------------------------------------------------------------

# 34. System Alerts / Operational Health API

## 34.1 List Actual Operational Alerts

`GET /api/v1/admin/system-alerts`

Alias: `GET /api/v1/admin/alerts`

**Role:** Full Admin, Customer Support

Query: `status`, `severity`, `type`, `page`, `page_size`.

This is a durable fact feed, not an invented uptime score. The server
materializes alerts from failed payment verification/webhook processing,
transfer uncertainty or failures, notification failures, reconciliation
mismatches, and payment-expiry work that has become stale. A resolved fact is
retained and is never silently reopened by a dashboard read.

## 34.2 Acknowledge or Resolve an Alert

`POST /api/v1/admin/system-alerts/{alert_id}/acknowledge`

`POST /api/v1/admin/system-alerts/{alert_id}/resolve`

Aliases under `/admin/alerts` are accepted.

**Role:** Full Admin

**Required header:** `Idempotency-Key`

```json
{ "reason": "Reconciliation run confirmed the provider result." }
```

Both commands require a reason and append an audit event. They change only
the administrative acknowledgement/resolution state; they do not alter the
underlying payment, withdrawal, notification, ledger, or reconciliation fact.

------------------------------------------------------------------------

# 35. Paystack Reconciliation API

## 35.1 Reconciliation Summary

`GET /api/v1/admin/reconciliation/paystack`

**Role:** Full Admin

### Query

``` text
date
from
to
status
page
page_size
```

------------------------------------------------------------------------

## 35.2 Reconciliation Record

`GET /api/v1/admin/reconciliation/paystack/{record_id}`

**Role:** Full Admin

------------------------------------------------------------------------

## 35.3 Resolve Unmatched Transaction

`POST /api/v1/admin/reconciliation/paystack/{record_id}/resolve`

**Role:** Full Admin

### Request

``` json
{
  "action": "link_to_booking",
  "booking_id": "bkg_123",
  "note": "Verified against customer payment reference."
}
```

This operation must be highly audited and must not create duplicate
financial records.

------------------------------------------------------------------------

# 36. Audit Trail API

## 36.1 List Audit Events

`GET /api/v1/admin/audit-logs`

Alias: `GET /api/v1/admin/audit`

**Role:** Full Admin

### Query

``` text
actor_id
actor
action
entity_type
entity_id
request_id
from
to
page
page_size
```

------------------------------------------------------------------------

## 36.2 Audit Event Detail

`GET /api/v1/admin/audit-logs/{audit_id}`

Alias: `GET /api/v1/admin/audit/{audit_id}`

**Role:** Full Admin

Audit records are append-only.

No API may provide deletion or modification of an existing audit record.

------------------------------------------------------------------------

# 36A. Phase 10 Admin Operations Contract

All Phase 10 Admin endpoints use the isolated Admin bearer session and the
canonical `/api/v1` prefix. Only the two approved roles are accepted:
`full_admin` and `customer_support`. `GET /admin/me` returns the server-owned
role and its fixed capability list; a supplied browser permission list is
never trusted.

Customer Support has read-only operational access to the dashboard, Partner,
Client, listing, booking, notification, and alert views. It cannot retrieve
payment/provider facts, financial trails, withdrawals, commission settings,
configuration, or audit history, and it cannot mutate any Admin resource.
Full Admin has the additional finance/configuration and state-transition
capabilities documented below. A `403 FORBIDDEN` is returned before data is
serialized when a capability is absent.

## 36A.1 Partner Financial Read Models

`GET /api/v1/admin/financial/partners`

`GET /api/v1/admin/financial/partners/{partner_id}`

**Role:** Full Admin

The list accepts `search`, `currency`, `page`, and `page_size`. Detail is a
read-only audited trail containing the current cached account projection,
append-only ledger entries, frozen CommissionRecords, PayoutEvents,
withdrawal history, and reconciliation facts. All money is integer minor
units. Neither endpoint can alter a balance, ledger row, settlement, or
withdrawal state.

## 36A.2 Global Search

`GET /api/v1/admin/search?q={term}`

**Role:** Full Admin, Customer Support

`q` is required and must contain 2--120 characters. Results are safe compact
records for booking reference, listing, Client, and Partner. A Full Admin
also receives payment-reference and withdrawal-reference results. Each item
contains opaque `id`, `type`, display `title`/`subtitle`, status, optional
reference, and a portal route; it never carries payment secrets, bank account
numbers, tokens, or raw provider payloads.

## 36A.3 Safe Platform Configuration

`GET /api/v1/admin/configuration`

**Role:** Full Admin

Returns read-only category/city activation reference data and configured
withdrawal policy facts. It intentionally has no endpoint for environment
secrets, encryption keys, provider tokens, SMTP credentials, or arbitrary
configuration mutation. Commission configuration is managed only through
the versioned endpoints in section 26.

## 36A.4 System-Alert Facts

The dashboard and `GET /admin/system-alerts` materialize only durable facts:
failed payment verification or webhook handling, failed or uncertain transfer
handling, failed notification delivery, reconciliation mismatch, and stale
pending-payment work. `acknowledge` and `resolve` update only the alert's
administrative state, require a reason and `Idempotency-Key`, and are audited;
they never alter the underlying financial or provider fact.

------------------------------------------------------------------------

# 37. Admin Reports API

## 37.1 Financial Summary

`GET /api/v1/admin/reports/financial-summary`

**Role:** Full Admin

### Query

``` text
from
to
city_id
category_id
```

### Response

``` json
{
  "success": true,
  "data": {
    "gross_booking_value": 0,
    "commission_revenue": 0,
    "partner_released": 0,
    "refunds": 0,
    "withdrawals": 0
  }
}
```

------------------------------------------------------------------------

## 37.2 Booking Report

`GET /api/v1/admin/reports/bookings`

**Role:** Full Admin

------------------------------------------------------------------------

## 37.3 Partner Report

`GET /api/v1/admin/reports/partners`

**Role:** Full Admin

------------------------------------------------------------------------

# 38. Account Settings API

## 38.1 Update Client Profile

`PATCH /api/v1/me`

**Role:** Client

------------------------------------------------------------------------

## 38.2 Change Password

`POST /api/v1/me/password`

**Role:** Authenticated user

### Request

``` json
{
  "current_password": "old",
  "new_password": "new"
}
```

------------------------------------------------------------------------

## 38.3 Partner Password

`POST /api/v1/partner/password`

**Role:** Partner

------------------------------------------------------------------------

# 39. Web/Mobile Client Contract

The Web and Mobile applications must consume the same endpoint contract.

Example:

``` text
Web
  ↓
POST /api/v1/bookings

Mobile
  ↓
POST /api/v1/bookings

Backend
  ↓
Same booking service
```

There must not be separate business implementations for Web and Mobile.

------------------------------------------------------------------------

# 40. State Transition API Rules

The backend must reject invalid state transitions.

Examples:

``` text
PENDING_PAYMENT → UPCOMING
```

Allowed only after authoritative successful payment.

``` text
ACTIVE → DISPUTED
```

Allowed only while the dispute window remains open and no prior release
has occurred.

``` text
DISPUTED → COMPLETED
```

Allowed after valid Admin resolution.

``` text
COMPLETED → COMPLETED
```

A repeated request must be idempotent or rejected as already completed;
it must never release funds twice.

------------------------------------------------------------------------

# 41. Financial API Rules

The following are mandatory across all endpoints:

1.  Client-submitted amount is never trusted as the authoritative
    booking total.
2.  Commission is calculated server-side.
3.  Commission rate is frozen at booking confirmation.
4.  Partner pending earnings are not withdrawable.
5.  Only available earnings can be withdrawn.
6.  Withdrawal funds must be reserved atomically.
7.  Failed transfers must reverse reservations exactly once.
8.  Disputed funds cannot auto-release.
9.  Release must be idempotent.
10. Refunds must be represented as financial events.
11. Historical CommissionRecords must not be rewritten because of future
    rate changes.
12. Provider webhooks must be idempotent.
13. A withdrawal may target only a verified, server-created payout recipient;
    plaintext bank numbers and recipient creation are never browser-owned.
14. An unknown Paystack transfer outcome leaves funds reserved until verified
    reconciliation establishes success, failure, or reversal.

------------------------------------------------------------------------

# 42. Notification Event Contract

Business services emit stable lower-case event types; the Notification Service
maps them to stable application template keys and recipient-specific durable
notification records. Event creation happens within the relevant business
transaction, while delivery is queued only after commit. The following Phase 9
events are supported where their source domain has emitted the required fact:

| Domain | Event type | Application template key | Intended operational recipient |
|---|---|---|---|
| Authentication | `auth.otp_requested` | `auth_otp` | Client or Partner owning the OTP |
| Onboarding | `partner.onboarding_submitted` | `partner_onboarding_submitted` | Submitting Partner |
| Listing | `listing.submitted` | `listing_submitted` | Owning Partner |
| Listing | `listing.approved` | `listing_approved` | Owning Partner |
| Listing | `listing.changes_requested` | `listing_changes_requested` | Owning Partner |
| Listing | `listing.suspended` | `listing_suspended` | Owning Partner |
| Booking | `booking.payment_required` | `booking_payment_required` | Owning Client |
| Booking | `booking.created` | `booking_payment_required` | Relevant Partner |
| Booking | `booking.cancelled` | `booking_cancelled` | Affected Client and Partner when payment expiry cancels the booking |
| Booking | `booking.confirmed` | `booking_confirmed` | Owning Client and relevant Partner |
| Booking | `booking.reminder` | `booking_reminder` | Owning Client |
| Booking | `booking.active` | `booking_active` | Owning Client and relevant Partner |
| Payment | `payment.successful` | `payment_successful` | Owning Client |
| Payment | `payment.failed` | `payment_failed` | Owning Client |
| Travel Package | `travel.balance_due` | `travel_balance_due` | Owning Client |
| Travel Package | `travel.balance_paid` | `travel_balance_paid` | Owning Client |
| Settlement | `settlement.entitlement_pending` | `partner_entitlement_pending` | Owning Partner |
| Settlement | `settlement.funds_released` | `funds_released` | Owning Partner |
| Booking | `booking.completed` | `booking_completed` | Owning Client and relevant Partner |
| Withdrawal | `withdrawal.requested` | `withdrawal_requested` | Owning Partner; a no-contact Admin fact is retained for operations |
| Withdrawal | `withdrawal.approved` | `withdrawal_approved` | Owning Partner |
| Withdrawal | `withdrawal.held` | `withdrawal_held` | Owning Partner |
| Withdrawal | `withdrawal.rejected` | `withdrawal_rejected` | Owning Partner |
| Withdrawal | `withdrawal.processing` | `withdrawal_processing` | Owning Partner |
| Withdrawal | `withdrawal.successful` | `withdrawal_successful` | Owning Partner |
| Withdrawal | `withdrawal.failed` | `withdrawal_failed` | Owning Partner |
| Withdrawal | `withdrawal.reversed` | `withdrawal_failed` | Owning Partner; no nonexistent distinct template is assumed |

Each resulting notification has a server-generated opaque ID, stable
recipient-scoped `event_key`, event type, optional related entity type/ID,
template key/version, primary channel, safe payload/context, lifecycle state,
and timestamps. The version is selected when the event is written and the
worker resolves that persisted version for delivery/retry/resend. A retry,
worker replay, or provider webhook replay never creates a second notification
from the same event key. A single underlying domain event may legitimately
produce separate notification records for different intended recipients; their
event keys must therefore differ.

Phase 8 withdrawal facts are the original durable source for the listed
withdrawal events. They are now eligible for asynchronous delivery but do not
gain any financial side effect from notification processing. The current
repository has no Phase 9 dispute command/service event source, so no dispute
notification is claimed here; one can be added later with a versioned template
and deterministic event key when that domain surface is implemented.

------------------------------------------------------------------------

# 43. Notification Delivery, Retry, and Fallback Contract

Operational notification flow:

``` text
Committed business event/outbox fact
      ↓
Delivery worker claims notification
      ↓
WhatsApp primary attempt(s), bounded and append-only
      ├── provider accepted / receipt advances status → complete
      └── final WhatsApp failure → one email fallback attempt
                                           ├── email accepted → fallback_email_sent
                                           └── email failed → failed
```

The delivery worker persists the attempt before provider I/O and uses an
exclusive claim/lease or equivalent locked transition. This makes a duplicate
worker execution, retry, Admin resend request, or provider receipt harmless.
The bounded retry policy is configuration, not an excuse for an infinite loop;
after its final WhatsApp failure, the notification does not repeatedly create
fallback emails.

The notification record stores safe scheduling fields such as next eligible
attempt time and aggregate attempt count. Each append-only attempt stores its
channel, provider, provider message ID where available, attempt number,
queued/sent/delivered/failed timestamps, safe provider status/metadata, and
safe failure reason. Provider receipts are separately deduplicated before they
advance an existing attempt. Provider submission/acceptance is `sent`; only a
matching delivery receipt may assert `delivered` or `read`.

A notification failure must not roll back, defer, or alter the business
transaction. For example, a verified payment and confirmed booking remain so
if WhatsApp retries and the email fallback both fail.

------------------------------------------------------------------------

# 44. Webhook Security

All provider webhooks must:

1.  authenticate/verify provider signature
2.  validate event structure
3.  identify provider event ID/reference
4.  check idempotency
5.  persist provider event
6.  process domain state transition
7.  log processing outcome

Malformed or unauthenticated webhooks must not modify financial state.

------------------------------------------------------------------------

# 45. Rate Limiting

Rate limiting should be applied particularly to:

-   login
-   OTP request
-   OTP verification
-   password reset
-   listing search
-   booking creation
-   payment initialization
-   dispute creation
-   withdrawal creation
-   chat message sending
-   admin authentication

Exact limits are deployment configuration and are not defined by the
current PRDs.

------------------------------------------------------------------------

# 46. File Upload API Rules

Where document/image uploads are supported:

`multipart/form-data`

The backend must validate:

-   file size
-   MIME type
-   extension
-   ownership
-   resource state

Uploaded files must not be trusted merely because the client supplies an
extension.

------------------------------------------------------------------------

# 47. API Security Requirements

The implementation must include:

-   TLS in production
-   secure token handling
-   password hashing
-   provider webhook verification
-   authorization middleware
-   input validation
-   output filtering
-   rate limiting
-   audit logging for sensitive Admin actions
-   protection against replay of financial commands
-   idempotency for financial operations

------------------------------------------------------------------------

# 48. API Acceptance Criteria

The API layer is considered complete only when:

### Authentication

-   Client can register
-   WhatsApp OTP works
-   Client can authenticate
-   Partner can authenticate
-   Admin authentication is isolated
-   Role authorization works

### Marketplace

-   Public listing discovery works
-   category filtering works
-   city filtering works
-   listing detail works
-   Partner listing workflow works
-   Admin approval works

### Booking

-   availability can be queried
-   booking creation works
-   duplicate booking is prevented
-   booking states are enforced
-   cancellation rules are enforced

### Payment

-   Paystack initialization works
-   webhook signature is verified
-   duplicate webhook is harmless
-   payment state is recorded
-   successful payment confirms booking

### Settlement

-   commission is frozen
-   Partner pending balance is correct
-   Confirm Experience works
-   auto-release works
-   dispute blocks release
-   release cannot execute twice

### Withdrawal

-   only available funds can be withdrawn
-   funds are reserved
-   Admin approval works
-   Paystack transfer status is tracked
-   failed transfer returns funds exactly once

### Disputes

-   Client can raise eligible disputes
-   auto-release stops
-   Admin can resolve
-   all three resolution paths are supported
-   financial outcomes are auditable

### Notifications

-   WhatsApp events are dispatched
-   delivery is tracked
-   failed WhatsApp delivery triggers email fallback
-   notification failures do not corrupt business state

### Admin

-   role restrictions work
-   financial operations are protected
-   audit records are created
-   system alerts are visible
-   Paystack reconciliation is available

------------------------------------------------------------------------

# 49. Endpoint Inventory

## Public

``` text
POST   /auth/register
POST   /auth/otp/request
POST   /auth/otp/verify
POST   /auth/login
POST   /auth/refresh
GET    /categories
GET    /cities
GET    /listings
GET    /listings/{listing_id}
GET    /listings/{listing_id}/availability
POST   /listings/{listing_id}/quote
GET    /partners/{partner_id}/portfolio
```

## Client

``` text
POST   /auth/logout
GET    /me
PATCH  /me
POST   /me/password
GET    /me/bookings
POST   /bookings
GET    /bookings/{booking_id}
POST   /bookings/{booking_id}/cancel
POST   /bookings/{booking_id}/confirm-experience
POST   /bookings/{booking_id}/disputes
POST   /payments/initialize
GET    /me/payments
GET    /payments/{payment_id}
GET    /payments/{payment_id}/verify
GET    /bookings/{booking_id}/payment-schedule
GET    /me/wishlist
POST   /me/wishlist
DELETE /me/wishlist/{listing_id}
GET    /me/support/conversations
POST   /me/support/conversations
GET    /me/support/conversations/{conversation_id}
POST   /me/support/conversations/{conversation_id}/messages
POST   /me/support/conversations/{conversation_id}/attachments
GET    /me/notifications
POST   /notifications/{notification_id}/read
GET    /bookings/{booking_id}/refund
```

## Partner

``` text
POST   /partner/register
GET    /partner/profile
PATCH  /partner/profile
POST   /partner/password
GET    /partner/listings
POST   /partner/listings
GET    /partner/listings/{listing_id}
PATCH  /partner/listings/{listing_id}
POST   /partner/listings/{listing_id}/submit-review
GET    /partner/categories/{category_id}/commission
GET    /partner/listings/{listing_id}/availability
PUT    /partner/listings/{listing_id}/availability
GET    /partner/bookings
GET    /partner/bookings/{booking_id}
GET    /partner/earnings
GET    /partner/earnings/transactions
GET    /partner/bookings/{booking_id}/commission
GET    /partner/bank-accounts
POST   /partner/bank-accounts
GET    /partner/withdrawals/eligibility
POST   /partner/withdrawals
GET    /partner/withdrawals
GET    /partner/withdrawals/{withdrawal_id}
GET    /partner/conversations
GET    /partner/conversations/{conversation_id}
POST   /partner/conversations/{conversation_id}/messages
POST   /partner/conversations/{conversation_id}/attachments
GET    /partner/notifications
```

## Admin

``` text
POST   /admin/auth/login
POST   /admin/auth/logout
GET    /admin/me

GET    /admin/dashboard

GET    /admin/partners
GET    /admin/partners/{partner_id}
POST   /admin/partners/{partner_id}/approve
POST   /admin/partners/{partner_id}/suspend
POST   /admin/partners/{partner_id}/reactivate

GET    /admin/clients
GET    /admin/clients/{client_id}
POST   /admin/clients/{client_id}/suspend
POST   /admin/clients/{client_id}/reactivate

GET    /admin/listings
GET    /admin/listings/{listing_id}
POST   /admin/listings/{listing_id}/approve
POST   /admin/listings/{listing_id}/request-changes
POST   /admin/listings/{listing_id}/suspend
POST   /admin/listings/{listing_id}/reactivate

GET    /admin/bookings
GET    /admin/bookings/{booking_id}
GET    /admin/bookings/{booking_id}/financial
GET    /admin/payments
GET    /admin/payments/{payment_id}

GET    /admin/disputes
GET    /admin/disputes/{dispute_id}
POST   /admin/disputes/{dispute_id}/notes
POST   /admin/disputes/{dispute_id}/resolve

GET    /admin/withdrawals
GET    /admin/withdrawals/{withdrawal_id}
POST   /admin/withdrawals/{withdrawal_id}/approve
POST   /admin/withdrawals/{withdrawal_id}/hold
POST   /admin/withdrawals/{withdrawal_id}/reject

GET    /admin/commission-rates
POST   /admin/commission-rates
POST   /admin/commission-rates/{rate_id}/retire

GET    /admin/financial/partners
GET    /admin/financial/partners/{partner_id}

GET    /admin/support/conversations
GET    /admin/support/conversations/{conversation_id}
POST   /admin/support/conversations/{conversation_id}/assign
POST   /admin/support/conversations/{conversation_id}/messages
POST   /admin/support/conversations/{conversation_id}/close

GET    /admin/partner-conversations
POST   /admin/partner-conversations
POST   /admin/partner-conversations/{conversation_id}/messages

GET    /admin/notifications
GET    /admin/notifications/{notification_id}
POST   /admin/notifications/{notification_id}/resend

POST   /admin/bookings/{booking_id}/refund

GET    /admin/follow-ups
POST   /admin/follow-ups/{follow_up_id}/complete

GET    /admin/system-alerts
POST   /admin/system-alerts/{alert_id}/acknowledge
POST   /admin/system-alerts/{alert_id}/resolve

GET    /admin/audit-logs
GET    /admin/audit-logs/{audit_id}
GET    /admin/search
GET    /admin/configuration
```

## Webhooks

``` text
POST /webhooks/paystack
GET  /webhooks/whatsapp
POST /webhooks/whatsapp
```

------------------------------------------------------------------------

# 50. Implementation Notes / Explicit Gaps

The current PRDs define the product behavior but do not fully specify
several low-level API choices.

The following must therefore be finalized during API/database
implementation rather than silently invented:

1.  Exact authentication token technology and expiry values.
2.  Exact public ID format.
3.  Exact minimum withdrawal amount, if any.
4.  Exact pagination implementation beyond the proposed page/page_size
    model.
5.  Exact API rate limits.
6.  Exact file storage provider and limits.
7.  Exact chat real-time transport.
8.  Exact Paystack provider request fields after the implementation
    stack is selected.
9.  Exact WhatsApp BSP implementation.
10. Exact email provider.
11. Exact refund eligibility matrix for every cancellation scenario.
12. Exact category-specific request schemas for all 19 listing
    categories.
13. Exact Travel Package deposit percentages and balance rules.
14. Exact subscription API contract for Partner premium subscriptions.
15. Exact review/rating API contract if reviews are retained in the
    final scope.

These gaps are intentionally visible.

They must be resolved before the affected implementation phase rather
than guessed by the coding agent.

------------------------------------------------------------------------

# 51. API Development Rule

Codex must treat this document as a contract.

When implementing an endpoint:

1.  Read the relevant product requirement.
2.  Read the Engineering Specification.
3.  Read this API contract.
4.  Inspect the existing codebase.
5.  Implement the endpoint.
6.  Add validation.
7.  Add authorization.
8.  Add idempotency where required.
9.  Add tests.
10. Update this document if the approved contract changes.
11. Do not silently change the API to make implementation easier.

The API is not considered complete merely because the route returns HTTP
200.

The complete contract includes:

-   authorization
-   validation
-   business rules
-   state transition
-   persistence
-   transaction handling
-   external provider behavior
-   idempotency
-   error handling
-   tests

------------------------------------------------------------------------

# 52. Relationship to Other Engineering Documents

``` text
PRD
 ↓
HANDYPROS_ENGINEERING_SPECIFICATION.md
 ↓
HANDYPROS_API_DOCUMENTATION.md
 ↓
HANDYPROS_DATABASE_SPECIFICATION.md
 ↓
IMPLEMENTATION
```

The PRD defines product intent.

The Engineering Specification defines business and system behavior.

This document defines API communication.

The Database Specification will define persistence.

The implementation must conform to all four.

------------------------------------------------------------------------

# 53. Next Artifact

After this API contract is reviewed and approved, the next document is:

`HANDYPROS_DATABASE_SPECIFICATION.md`

That document must derive its schema from:

-   the domain entities
-   booking state machine
-   payment state
-   CommissionRecord
-   Partner ledger
-   withdrawal lifecycle
-   dispute lifecycle
-   notification events
-   audit requirements
-   Admin operations
-   API resource requirements

Only after the API and database contracts are stable should the fresh
Codex implementation project begin.

------------------------------------------------------------------------

# 54. Phase 11 Protection, Cancellation, Refund and Dispute Contract

All endpoints below are canonical `/api/v1` endpoints. Money is integer NGN
minor units. Financial commands require `Idempotency-Key`; the server derives
all payment, currency, policy and entitlement facts.

## Cancellation and refunds

| Endpoint | Actor | Contract |
|---|---|---|
| `POST /bookings/{id}/cancellation-quote` | owning Client | Creates a ten-minute, policy-provenanced quote. |
| `POST /bookings/{id}/cancel` | owning Client | `{quote_id, reason}`; executes only the quoted server calculation. |
| `POST /partner/bookings/{id}/cancellation-quote` / `cancel` | owning active Partner | Same request shape, but policy selection is actor-specific and normally routes undefined Partner commercial outcomes to Admin review. |
| `POST /admin/bookings/{id}/cancellation-quote` / `cancel` | Full Admin | Uses the same policy engine; never accepts a browser-provided ledger amount. |
| `GET /bookings/{id}/refunds`, `GET /partner/bookings/{id}/refunds` | owner | Safe refund timeline only. |
| `GET /admin/refunds`, `GET /admin/refunds/{id}` | Full Admin / safe support projection | List/detail. Provider and recovery data are Full-Admin-only. |
| `POST /admin/bookings/{id}/refunds` | Full Admin | `{amount, reason, resolution_basis}`; validates remaining verified payment basis. |
| `POST /admin/refunds/{id}/retry` | Full Admin | Empty JSON body plus idempotency key; only definitively failed refund intents can retry. |

Cancellation quote responses contain `id`, `expires_at`, eligibility,
outcome, amount paid, refundable/non-refundable amount, currency, policy
identity/rule, and explanation. They do not expose provider payloads. Active
versioned policies are selected by effective dates, category, actor, booking
state, payment state and timing. There are no seeded production percentages.

Refund states are `requested`, `processing`, `successful`, `failed`,
`provider_uncertain`, and `rejected`. A request is durable before Paystack I/O.
Provider acceptance is not success: signed webhook facts or provider
verification determine the terminal state. A `provider_uncertain` intent is
reconciled, not blindly retried.

## Disputes

| Endpoint | Actor | Contract |
|---|---|---|
| `GET/POST /bookings/{id}/disputes` | owning Client | Create requires `reason_category`, a 100–5000-character `description`, and idempotency. |
| `POST /bookings/{id}/disputes/{disputeId}/evidence` | owning Client | Multipart `file`; PDF, JPEG, PNG, WEBP or plain text; maximum 10 MiB. |
| `GET /partner/bookings/{id}/disputes` | owning Partner | Safe case/evidence projection. |
| `POST /partner/bookings/{id}/disputes/{disputeId}/response` | owning Partner | `{response}` plus idempotency; response is 20–5000 characters. |
| `POST /partner/bookings/{id}/disputes/{disputeId}/evidence` | owning Partner | Same private-evidence safeguards. |
| `GET /admin/disputes`, `GET /admin/disputes/{id}` | Full Admin / Customer Support | Support receives a non-financial read-only case projection. |
| `POST /admin/disputes/{id}/resolve` | Full Admin | `{resolution_type, resolution_note, client_refund_amount?, partner_release_amount?}` plus idempotency. |

Supported resolutions are `full_partner_release`, `partial_partner_release`,
and `full_client_refund`. The service validates a server-derived retained
Partner entitlement from frozen CommissionRecords before accepting a partial
release amount. Refund outcomes go through the single Refund Service; a
Partner-favour outcome uses the existing Settlement Service exactly once.

Evidence is stored privately and response bodies return file metadata only,
never a public storage URL. An unresolved dispute holds `active_booking_id`,
freezing Client confirm and scheduler release. A resolved case clears that
hold only through Full-Admin resolution.

## Provider, audit and errors

`POST /webhooks/paystack` also processes Paystack refund events. It verifies
the HMAC signature, persists a deduplicated `refund_provider_events` fact, and
matches provider refund ID, original transaction reference, amount and
currency before changing a refund state. The scheduler entry point is
`backend/cli/process-v1-refund-reconciliation.php` and has no public mutation
route.

All cancellation, refund and dispute commands emit audit facts and durable
Phase-9 events. Standard responses are `{success:true,data:...}`. Relevant
safe failures are `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `409
CANCELLATION_NOT_ELIGIBLE`, `409 REFUND_NOT_ELIGIBLE`, `409
DISPUTE_NOT_ELIGIBLE`, `409 CONFLICT`, and `429 RATE_LIMITED`.
