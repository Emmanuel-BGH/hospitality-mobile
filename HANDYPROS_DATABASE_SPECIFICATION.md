# HANDYPROS HOSPITALITY

## Database Specification

**Document:** `HANDYPROS_DATABASE_SPECIFICATION.md`\
**Version:** 1.0\
**Status:** Engineering Database Baseline\
**Database role:** Authoritative persistence layer for Client, Partner,
Admin, booking, payment, settlement, communication, and operational
state.

------------------------------------------------------------------------

# 1. Purpose

This document defines the relational database structure required to
implement HandyPros Hospitality.

It is derived from the HandyPros Hospitality Product Requirement
Document and the approved engineering/API direction.

The database must support:

-   Client accounts
-   Partner accounts
-   Admin operations
-   19 hospitality categories
-   Four launch city areas
-   Listings and category-specific content
-   Listing approval and visibility
-   Seasonal pricing
-   Availability
-   Booking lifecycle
-   Paystack payments and webhooks
-   Immutable commission records
-   Auto-release
-   Partner earnings
-   Payout events
-   Withdrawals
-   Disputes
-   Refunds
-   At Risk flags
-   Futuristic bookings and follow-ups
-   WhatsApp/email notifications
-   In-app support chat
-   Partner-admin chat
-   Document acknowledgement
-   Wishlists
-   Premium subscriptions
-   Audit trail
-   Reconciliation
-   System alerts
-   High-value inquiries

The database is the source of truth. Frontend state, cached search
results, webhook payloads, and provider dashboards are not authoritative
application state.

------------------------------------------------------------------------

# 2. Database Principles

## 2.1 Relational database

Use a transactional relational database.

Recommended baseline:

-   MySQL 8.x or MariaDB version compatible with the production
    environment.
-   InnoDB for transactional tables.
-   UTF-8 capable collation.
-   Foreign keys enabled where operationally appropriate.

Do not design the system around an eventually consistent document
database.

## 2.2 Financial correctness over convenience

Financial records must be append-oriented and auditable.

Do not implement Partner earnings as a single mutable number with no
transaction history.

The system must maintain:

1.  immutable financial source records;
2.  ledger entries;
3.  payout events;
4.  withdrawal records;
5.  reconciliation records.

A cached balance may exist for performance, but it must be derivable and
reconcilable from the ledger.

## 2.3 Monetary storage

Never use floating point for money.

Use integer minor units.

Example:

``` text
amount_minor = 15000000
currency = NGN
```

If the application chooses to store decimal amounts, use fixed-precision
DECIMAL, never FLOAT/DOUBLE.

The implementation must choose one canonical monetary representation and
use it consistently.

## 2.4 Timestamps

Store timestamps in UTC.

Business display may convert to WAT.

Important scheduling dates include:

-   service start datetime
-   service end datetime
-   auto-release datetime
-   payment due datetime
-   withdrawal processing timestamps
-   notification timestamps
-   dispute timestamps
-   follow-up due timestamps

## 2.5 Soft deletion

Do not physically delete important operational or financial records.

For records that can be hidden from normal application views, use:

``` text
deleted_at
```

or a domain-specific inactive status.

Financial records must never be deleted.

Audit records must never be deleted.

------------------------------------------------------------------------

# 3. Entity Overview

Core domain entities:

``` text
Users
  ├── Clients
  ├── Partners
  └── Admin Users

Partners
  ├── Partner Bank Accounts
  ├── Listings
  ├── Partner Conversations
  ├── Earnings Ledger
  ├── Withdrawals
  └── Premium Subscriptions

Listings
  ├── Categories
  ├── Cities
  ├── Images
  ├── Amenities
  ├── Category Data
  ├── Pricing Rules
  └── Availability

Bookings
  ├── Booking Items / Service Details
  ├── Payments
  ├── Commission Records
  ├── Payout Events
  ├── Disputes
  ├── At Risk Flags
  ├── Follow-Ups
  ├── Refunds
  └── Receipts

Communication
  ├── Conversations
  ├── Messages
  ├── Attachments
  ├── Notifications
  └── Notification Attempts

Administration
  ├── Commission Rates
  ├── Audit Events
  ├── System Alerts
  └── Reconciliation Records
```

------------------------------------------------------------------------

# 4. Identity and Access Tables

## 4.1 `users`

Central identity table for Client and Partner accounts.

### Fields

  Field               Type             Rules
  ------------------- ---------------- -------------------------------------
  id                  UUID/opaque ID   PK
  role                ENUM/string      client, partner
  first_name          VARCHAR          required
  last_name           VARCHAR          required
  phone               VARCHAR          unique
  email               VARCHAR          nullable/unique where present
  password_hash       VARCHAR          required
  phone_verified_at   DATETIME         nullable
  status              VARCHAR          pending, active, suspended, blocked
  last_login_at       DATETIME         nullable
  created_at          DATETIME         required
  updated_at          DATETIME         required
  deleted_at          DATETIME         nullable

### Constraints

-   phone must be unique.
-   password must never be stored plaintext.
-   role cannot be changed casually by client-facing APIs.
-   blocked/suspended users cannot perform protected operations.

------------------------------------------------------------------------

## 4.2 `user_otp_verifications`

Stores OTP lifecycle.

  Field         Type
  ------------- -------------------
  id            PK
  user_id       FK nullable
  phone         VARCHAR
  channel       VARCHAR
  otp_hash      VARCHAR
  purpose       VARCHAR
  expires_at    DATETIME
  verified_at   DATETIME nullable
  attempts      INT
  created_at    DATETIME

Never store plaintext OTP values.

Purpose examples:

``` text
registration
login
phone_verification
```

------------------------------------------------------------------------

# 5. Partner Tables

## 5.1 `partners`

Partner business profile.

  Field                  Type
  ---------------------- -------------------
  id                     PK
  user_id                FK users.id
  business_name          VARCHAR
  legal_name             VARCHAR nullable
  display_name           VARCHAR
  description            TEXT nullable
  status                 VARCHAR
  approved_at            DATETIME nullable
  approved_by_admin_id   FK nullable
  suspended_at           DATETIME nullable
  suspended_reason       TEXT nullable
  created_at             DATETIME
  updated_at             DATETIME

Partner status should support at minimum:

``` text
pending_approval
active
suspended
blacklisted
```

The PRD requires Partners to begin in Pending Approval and remain unable
to access their dashboard until approval.

------------------------------------------------------------------------

## 5.2 `partner_bank_accounts`

Stores Partner payout account information.

  Field                              Type
  ---------------------------------- -------------------
  id                                 PK
  partner_id                         FK
  provider                           VARCHAR
  provider_recipient_code            VARCHAR nullable
  bank_code                          VARCHAR
  bank_name                          VARCHAR
  account_number_encrypted           TEXT
  account_number_last4               CHAR(4) nullable
  account_number_fingerprint         CHAR(64) nullable
  account_name                       VARCHAR
  verification_status                VARCHAR
  verification_failure_reason        VARCHAR nullable
  verified_at                        DATETIME nullable
  created_at                         DATETIME
  updated_at                         DATETIME

### Verification and uniqueness

Phase 8 uses the following lifecycle values:

``` text
pending_provider
verified
failed
```

`pending_provider` is a durable marker written before Paystack account
resolution/recipient creation leaves the transaction boundary. It prevents a
replayed command from creating another provider recipient when the first
external outcome is not yet known. A withdrawal may use only a `verified`
record with a non-null `provider_recipient_code`.

The target schema uniquely indexes:

``` text
(partner_id, provider, account_number_fingerprint)
(provider, provider_recipient_code)
```

The fingerprint is a keyed, non-plaintext duplicate-detection value. The
recipient uniqueness prevents the same provider recipient from being attached
to multiple bank-account records.

### Security

Bank account numbers must be encrypted at rest using a dedicated application
encryption key independent of Paystack credentials. No plaintext account
number, encryption key, raw recipient payload, or account fingerprint may be
included in normal API output, audit metadata, or logs. The current Phase 8
withdrawal surfaces return only a masked value and safe verification facts.

Any future Admin feature that permits a deliberately unmasked read must be a
separate Full Admin operation and create an append-only audit event. The PRD
explicitly requires encryption at rest and auditability for that access.

------------------------------------------------------------------------

## 5.3 `partner_onboarding_progress`

  Field                 Type
  --------------------- -------------------
  id                    PK
  partner_id            FK
  step_1_completed_at   DATETIME nullable
  step_2_completed_at   DATETIME nullable
  step_3_completed_at   DATETIME nullable
  completed_at          DATETIME nullable
  created_at            DATETIME
  updated_at            DATETIME

The exact three checklist steps should remain configurable at
application level unless finalized elsewhere.

------------------------------------------------------------------------

# 6. Admin Tables

## 6.1 `admin_users`

Separate Admin authentication infrastructure.

  Field           Type
  --------------- -------------------
  id              PK
  email           VARCHAR unique
  password_hash   VARCHAR
  role            VARCHAR
  status          VARCHAR
  last_login_at   DATETIME nullable
  created_at      DATETIME
  updated_at      DATETIME

Roles must include the roles defined by the Admin product, at minimum:

``` text
full_admin
customer_support
```

The Admin authentication system must not reuse Client/Partner
authentication infrastructure.

------------------------------------------------------------------------

## 6.2 `admin_sessions`

  Field                Type
  -------------------- -------------------
  id                   PK
  admin_user_id        FK
  refresh_token_hash   TEXT
  expires_at           DATETIME
  revoked_at           DATETIME nullable
  created_at           DATETIME

------------------------------------------------------------------------

# 7. Geography and Taxonomy

## 7.1 `cities`

Launch cities:

``` text
Lagos
Lekki/Ajah
Abuja
Port Harcourt
```

  Field        Type
  ------------ ----------------
  id           PK
  name         VARCHAR
  slug         VARCHAR unique
  status       VARCHAR
  created_at   DATETIME
  updated_at   DATETIME

------------------------------------------------------------------------

## 7.2 `verticals`

  Field        Type
  ------------ ----------------
  id           PK
  name         VARCHAR
  slug         VARCHAR unique
  sort_order   INT
  active       BOOLEAN

Required launch verticals:

``` text
Places to Stay
Things to Do
Rentals and Experiences
```

------------------------------------------------------------------------

## 7.3 `service_categories`

The 19 launch categories must be represented as data, not hard-coded
throughout the application.

  Field                  Type
  ---------------------- ----------------
  id                     PK
  vertical_id            FK
  name                   VARCHAR
  slug                   VARCHAR unique
  category_type          VARCHAR
  booking_mode           VARCHAR
  auto_release_hours     INT
  auto_release_trigger   VARCHAR
  active                 BOOLEAN
  sort_order             INT
  created_at             DATETIME
  updated_at             DATETIME

Required categories:

### Places to Stay

``` text
Hotel
Short-Stay Apartment
Beach-House Apartment
```

### Things to Do

``` text
Travel Package
Spa
Gym
Tennis Court Session
Netball
Paddle Court Session
One-on-One Swimming Session
House Party
Wine Tasting Event
```

### Rentals and Experiences

``` text
Event Hall
Car Rental with Driver
Drive-Yourself Car Rental
Boat Cruise
Private Jet Rental
```

The PRD defines category-specific auto-release windows. Store the
effective window on the category and snapshot it onto the booking at
confirmation.

------------------------------------------------------------------------

# 8. Commission Tables

## 8.1 `commission_rates`

Current configurable category rates.

  Field                 Type
  --------------------- -------------------
  id                    PK
  category_id           FK
  rate_percent          DECIMAL
  effective_from        DATETIME
  effective_to          DATETIME nullable
  status                VARCHAR
  created_by_admin_id   FK
  created_at            DATETIME

Do not overwrite historical rates used by confirmed bookings.

A new rate should create a new effective record/version.

------------------------------------------------------------------------

## 8.2 `commission_records`

Immutable booking-level commission snapshot.

  Field                Type
  -------------------- ----------------
  id                   PK
  booking_id           FK
  payment_id           FK nullable
  category_id          FK
  gross_amount         BIGINT/DECIMAL
  commission_rate      DECIMAL
  commission_amount    BIGINT/DECIMAL
  net_partner_payout   BIGINT/DECIMAL
  currency             CHAR(3)
  source               VARCHAR
  created_at           DATETIME

### Immutability

After creation:

-   no UPDATE
-   no DELETE

If a correction is required, create a compensating financial record
rather than editing history.

For Travel Packages, the PRD specifies a CommissionRecord on the deposit
and another on the balance payment.

------------------------------------------------------------------------

# 9. Listing Tables

## 9.1 `listings`

Core listing record.

  Field                  Type
  ---------------------- -------------------
  id                     PK
  partner_id             FK
  category_id            FK
  city_id                FK
  name                   VARCHAR
  slug                   VARCHAR unique
  description            TEXT
  neighborhood           VARCHAR
  address                TEXT
  status                 VARCHAR
  visibility_status      VARCHAR
  base_price             BIGINT/DECIMAL
  currency               CHAR(3)
  minimum_stay           INT nullable
  max_capacity           INT nullable
  created_at             DATETIME
  updated_at             DATETIME
  submitted_at           DATETIME nullable
  approved_at            DATETIME nullable
  approved_by_admin_id   FK nullable
  deleted_at             DATETIME nullable

Listing statuses should support:

``` text
draft
pending_review
changes_requested
active
hidden
inactive
suspended
```

Only Active listings are publicly visible.

------------------------------------------------------------------------

## 9.2 `listing_category_data`

Stores structured category-specific fields.

  Field            Type
  ---------------- ----------
  id               PK
  listing_id       FK
  schema_version   VARCHAR
  data_json        JSON
  created_at       DATETIME
  updated_at       DATETIME

### Important

The PRD requires category-specific forms and structured content.

Do not put every possible field from all 19 categories into the main
`listings` table.

Use structured category data with a versioned schema.

Common/high-value fields may still have dedicated columns where they are
frequently queried.

The application must validate `data_json` against the category schema
before submission.

------------------------------------------------------------------------

## 9.3 `listing_images`

  Field         Type
  ------------- -------------------
  id            PK
  listing_id    FK
  file_url      TEXT
  storage_key   TEXT
  sort_order    INT
  is_cover      BOOLEAN
  created_at    DATETIME
  deleted_at    DATETIME nullable

------------------------------------------------------------------------

## 9.4 `amenities`

  Field    Type
  -------- ----------------
  id       PK
  name     VARCHAR
  slug     VARCHAR unique
  active   BOOLEAN

------------------------------------------------------------------------

## 9.5 `listing_amenities`

  Field        Type
  ------------ ------
  listing_id   FK
  amenity_id   FK

Primary key:

``` text
(listing_id, amenity_id)
```

------------------------------------------------------------------------

## 9.6 `listing_review_events`

Tracks Admin review decisions.

  Field           Type
  --------------- ---------------
  id              PK
  listing_id      FK
  action          VARCHAR
  note            TEXT nullable
  admin_user_id   FK
  created_at      DATETIME

Actions:

``` text
submitted
approved
changes_requested
suspended
hidden
```

This preserves listing approval history without relying only on the
current status.

------------------------------------------------------------------------

# 10. Pricing Tables

## 10.1 `pricing_rules`

Used for seasonal accommodation pricing.

  Field          Type
  -------------- ----------------
  id             PK
  listing_id     FK
  rule_type      VARCHAR
  start_date     DATE nullable
  end_date       DATE nullable
  days_of_week   JSON nullable
  rate           BIGINT/DECIMAL
  priority       INT
  active         BOOLEAN
  created_at     DATETIME
  updated_at     DATETIME

Rule types:

``` text
standard
weekend
public_holiday
december_january
custom_period
```

Priority follows the PRD:

``` text
Custom Period
    >
Holiday Season
    >
Public Holiday
    >
Weekend
    >
Standard
```

The actual final rate for a booking must be stored in the booking
pricing snapshot so future Partner price changes do not alter historical
bookings.

------------------------------------------------------------------------

## 10.2 `public_holidays`

  Field     Type
  --------- ---------
  id        PK
  name      VARCHAR
  date      DATE
  country   VARCHAR
  active    BOOLEAN

------------------------------------------------------------------------

# 11. Availability Tables

## 11.1 `availability_days`

For date-based inventory.

  Field            Type
  ---------------- -------------------------
  id               PK
  listing_id       FK
  date             DATE
  status           VARCHAR
  price_override   BIGINT/DECIMAL nullable
  created_at       DATETIME
  updated_at       DATETIME

Statuses:

``` text
available
unavailable
blocked
```

Unique:

``` text
(listing_id, date)
```

------------------------------------------------------------------------

## 11.2 `availability_slots`

For experience/rental time slots.

  Field        Type
  ------------ ----------
  id           PK
  listing_id   FK
  slot_date    DATE
  start_time   TIME
  end_time     TIME
  capacity     INT
  status       VARCHAR
  created_at   DATETIME
  updated_at   DATETIME

Unique index should prevent duplicate slot definitions.

------------------------------------------------------------------------

## 11.3 `booking_inventory_locks`

Temporary booking locks.

  Field            Type
  ---------------- -------------------
  id               PK
  listing_id       FK
  booking_id       FK nullable
  inventory_type   VARCHAR
  inventory_date   DATE nullable
  slot_id          FK nullable
  quantity         INT
  status           VARCHAR
  expires_at       DATETIME
  created_at       DATETIME
  released_at      DATETIME nullable

This table supports the Availability Service requirement to enforce
booking-level locks and prevent double booking.

Locks must be acquired transactionally.

------------------------------------------------------------------------

# 12. Booking Tables

## 12.1 `bookings`

Primary booking record.

  Field                 Type
  --------------------- --------------------------
  id                    PK
  booking_reference     VARCHAR unique
  client_id             FK users.id
  partner_id            FK partners.id
  listing_id            FK listings.id
  category_id           FK service_categories.id
  city_id               FK cities.id
  status                VARCHAR
  currency              CHAR(3)
  gross_amount          BIGINT/DECIMAL
  service_start_at      DATETIME
  service_end_at        DATETIME
  auto_release_at       DATETIME
  completed_at          DATETIME nullable
  cancelled_at          DATETIME nullable
  cancellation_reason   TEXT nullable
  confirmed_at          DATETIME nullable
  created_at            DATETIME
  updated_at            DATETIME

### Booking statuses

``` text
pending_payment
upcoming
active
completed
cancelled
disputed
refunded
```

Additional operational states such as `at_risk` should not replace the
core booking lifecycle status. Use a separate At Risk record/flag.

------------------------------------------------------------------------

## 12.2 `booking_pricing_snapshots`

Immutable snapshot of what the Client paid and how it was calculated.

  Field              Type
  ------------------ ----------------
  id                 PK
  booking_id         FK
  pricing_version    VARCHAR
  base_amount        BIGINT/DECIMAL
  discount_amount    BIGINT/DECIMAL
  surcharge_amount   BIGINT/DECIMAL
  final_amount       BIGINT/DECIMAL
  currency           CHAR(3)
  calculation_json   JSON
  created_at         DATETIME

For standard HandyPros client checkout:

``` text
surcharge_amount = 0
```

because the PRD requires no platform surcharge.

------------------------------------------------------------------------

## 12.3 `booking_service_details`

Category-specific booking snapshot.

  Field          Type
  -------------- ----------
  id             PK
  booking_id     FK
  details_json   JSON
  created_at     DATETIME

Examples:

-   guest count
-   selected room type
-   session slot
-   rental quantity
-   travel package details

The exact fields depend on the category.

------------------------------------------------------------------------

## 12.4 `booking_status_history`

  Field             Type
  ----------------- ------------------
  id                PK
  booking_id        FK
  from_status       VARCHAR nullable
  to_status         VARCHAR
  changed_by_type   VARCHAR
  changed_by_id     VARCHAR nullable
  reason            TEXT nullable
  created_at        DATETIME

Every important state transition must be traceable.

------------------------------------------------------------------------

# 13. Travel Package Tables

## 13.1 `travel_package_payment_schedules`

  Field            Type
  ---------------- ----------------
  id               PK
  booking_id       FK
  deposit_amount   BIGINT/DECIMAL
  balance_amount   BIGINT/DECIMAL
  balance_due_at   DATETIME
  status           VARCHAR
  created_at       DATETIME
  updated_at       DATETIME

The PRD establishes a deposit at booking and later balance payment on a
Partner-defined date.

The exact deposit percentage is not defined in the source PRD and must
not be invented here.

------------------------------------------------------------------------

# 14. Payment Tables

## 14.1 `payments`

Represents application payment attempts.

  Field                Type
  -------------------- -------------------
  id                   PK
  booking_id           FK
  client_id            FK
  payment_type         VARCHAR
  provider             VARCHAR
  provider_reference   VARCHAR
  amount               BIGINT/DECIMAL
  currency             CHAR(3)
  status               VARCHAR
  provider_status      VARCHAR nullable
  paid_at              DATETIME nullable
  created_at           DATETIME
  updated_at           DATETIME

Payment types:

``` text
booking
travel_deposit
travel_balance
premium_subscription
```

Statuses:

``` text
pending
successful
failed
reversed
refunded
partially_refunded
```

Provider reference must be unique.

------------------------------------------------------------------------

## 14.2 `payment_webhook_events`

Stores inbound Paystack webhook events.

  Field                Type
  -------------------- -------------------
  id                   PK
  provider             VARCHAR
  provider_event_id    VARCHAR nullable
  event_type           VARCHAR
  reference            VARCHAR nullable
  signature_verified   BOOLEAN
  payload_json         JSON
  processing_status    VARCHAR
  processed_at         DATETIME nullable
  error_message        TEXT nullable
  created_at           DATETIME

Unique constraint should be applied to a stable provider event
identifier where Paystack provides one.

Webhook processing must be idempotent.

------------------------------------------------------------------------

## 14.3 `refunds`

  Field                   Type
  ----------------------- -------------------
  id                      PK
  booking_id              FK
  payment_id              FK
  amount                  BIGINT/DECIMAL
  currency                CHAR(3)
  provider_reference      VARCHAR nullable
  reason                  TEXT
  status                  VARCHAR
  initiated_by_admin_id   FK
  created_at              DATETIME
  completed_at            DATETIME nullable

Statuses:

``` text
pending
processing
successful
failed
```

Never delete refunds.

------------------------------------------------------------------------

# 15. Partner Earnings and Ledger

## 15.1 `partner_ledger_accounts`

One logical earnings account per Partner.

  Field               Type
  ------------------- ----------------
  id                  PK
  partner_id          FK unique
  currency            CHAR(3)
  available_balance   BIGINT/DECIMAL
  pending_balance     BIGINT/DECIMAL
  reserved_balance    BIGINT/DECIMAL
  created_at          DATETIME
  updated_at          DATETIME

These balances are cached/derived values.

The ledger remains authoritative.

------------------------------------------------------------------------

## 15.2 `partner_ledger_entries`

Append-only financial ledger.

  Field               Type
  ------------------- ----------------
  id                  PK
  ledger_account_id   FK
  partner_id          FK
  booking_id          FK nullable
  withdrawal_id       FK nullable
  payout_event_id     FK nullable
  entry_type          VARCHAR
  direction           VARCHAR
  amount              BIGINT/DECIMAL
  currency            CHAR(3)
  balance_bucket      VARCHAR
  reference           VARCHAR
  description         TEXT
  created_at          DATETIME

Entry types:

``` text
pending_credit
release_credit
withdrawal_reservation
withdrawal_release
withdrawal_debit
refund_adjustment
manual_adjustment
```

Directions:

``` text
credit
debit
```

Buckets:

``` text
pending
available
reserved
```

### Critical rule

No financial operation should simply do:

``` sql
UPDATE partners SET balance = balance + X;
```

without a corresponding ledger entry.

------------------------------------------------------------------------

# 16. Payout Tables

## 16.1 `payout_events`

Represents money becoming available to the Partner after release.

  Field          Type
  -------------- ----------------
  id             PK
  booking_id     FK
  partner_id     FK
  amount         BIGINT/DECIMAL
  currency       CHAR(3)
  trigger_type   VARCHAR
  status         VARCHAR
  created_at     DATETIME

Trigger types:

``` text
auto_release
client_confirmed
admin_resolution
```

The event must be unique per release action.

A booking cannot create two full payout events for the same settlement
amount.

------------------------------------------------------------------------

# 17. Withdrawal Tables

## 17.1 `withdrawals`

  Field                        Type
  ---------------------------- -------------------
  id                           PK
  withdrawal_reference         VARCHAR unique
  partner_id                   FK
  ledger_account_id            FK
  bank_account_id              FK
  amount                       BIGINT positive minor units
  currency                     CHAR(3)
  status                       VARCHAR
  provider                     VARCHAR
  provider_reference           VARCHAR nullable
  provider_transfer_code       VARCHAR nullable
  provider_status              VARCHAR nullable
  requested_at                 DATETIME
  approved_at                  DATETIME nullable
  approved_by_admin_id         FK nullable
  processing_at                DATETIME nullable
  transfer_initiated_at        DATETIME nullable
  completed_at                 DATETIME nullable
  failed_at                    DATETIME nullable
  rejected_at                  DATETIME nullable
  reversed_at                  DATETIME nullable
  rejection_reason             TEXT nullable
  failure_reason               TEXT nullable
  reversal_reason              TEXT nullable
  reconciliation_checked_at    DATETIME nullable
  created_at                   DATETIME
  updated_at                   DATETIME

The persisted Phase 8 state machine is:

``` text
pending_review -> held | approved | rejected
held           -> approved | rejected
approved       -> processing
processing     -> completed | failed | reversed
```

`approved` and `processing` retain the reservation. `rejected`, `failed`,
and `reversed` have different operational causes but all return the reserved
amount exactly once. `completed` consumes the reserved amount exactly once.
The service must reject a terminal-state mutation instead of treating it as a
new payout instruction.

Target constraints include a positive-amount check, unique withdrawal
reference, unique `(provider, provider_reference)`, unique non-null
`(provider, provider_transfer_code)`, lifecycle/query indexes, and foreign
keys to the Partner, ledger account, bank account, and approving Admin.

------------------------------------------------------------------------

## 17.2 `withdrawal_policy_settings`

Configuration for a minimum withdrawal, when Product supplies one.

  Field                       Type
  --------------------------- -------------------
  id                          PK
  currency                    CHAR(3) unique
  minimum_withdrawal_amount   BIGINT nullable
  active                      BOOLEAN
  created_at                  DATETIME
  updated_at                  DATETIME

The amount is either null or a positive integer. The absence of an active
positive value means no minimum is applied or exposed. No default minimum is
seeded because the PRD does not define one.

------------------------------------------------------------------------

## 17.3 `withdrawal_transfer_attempts`

One durable Paystack transfer instruction per withdrawal.

  Field                      Type
  -------------------------- -------------------
  id                         PK
  withdrawal_id              FK unique
  provider                   VARCHAR
  transfer_reference         VARCHAR
  provider_transfer_code     VARCHAR nullable
  provider_status            VARCHAR nullable
  status                     VARCHAR
  amount                     BIGINT positive minor units
  currency                   CHAR(3)
  initiation_attempted_at    DATETIME nullable
  initiated_at               DATETIME nullable
  last_verified_at           DATETIME nullable
  finalized_at               DATETIME nullable
  failure_reason             TEXT nullable
  created_at                 DATETIME
  updated_at                 DATETIME

`transfer_reference` is generated by the backend before the provider call and
is unique with `(provider, transfer_reference)`. The provider transfer code is
also unique when present. Attempt states distinguish at least `queued`,
`initiating`, `processing`, `unknown`, `successful`, `failed`, and
`reversed`. An unknown outcome is deliberately not converted to failure until
verified reconciliation establishes a terminal provider fact.

------------------------------------------------------------------------

## 17.4 `transfer_webhook_events`

Append-only, signed Paystack transfer-webhook evidence.

  Field                  Type
  ---------------------- -------------------
  id                     PK
  provider               VARCHAR
  provider_event_id      VARCHAR
  event_type             VARCHAR
  reference_value        VARCHAR nullable
  withdrawal_id          FK nullable
  signature_verified     BOOLEAN
  payload_json           JSON
  processing_status      VARCHAR
  processed_at           DATETIME nullable
  error_message          VARCHAR nullable
  created_at             DATETIME

`(provider, provider_event_id)` is unique. The stored payload is a sanitized
transfer fact, never a raw credential-bearing provider request. Processing
statuses distinguish `processing`, `processed`, `unmatched`, and `retryable`.
Duplicate webhook deliveries return the existing result and cannot make a
second ledger movement.

------------------------------------------------------------------------

## 17.5 Withdrawal reservation and terminal ledger effects

Do not create a separate balance mutation without a ledger entry.

When a withdrawal is requested in one transaction:

``` text
available
   ↓
reserved
```

This occurs atomically with the withdrawal creation using two append-only
entries with the same withdrawal ID:

``` text
WITHDRAWAL_AVAILABLE_DEBIT    debit  available
WITHDRAWAL_RESERVED_CREDIT    credit reserved
```

The Partner ledger account row is locked and reconciled against the ledger
before the balance projection changes. The target unique key
`(withdrawal_id, entry_type)` prevents a repeated command/event from adding a
second instance of any Phase 8 withdrawal entry type.

If the withdrawal is rejected, permanently fails, or is reversed:

``` text
reserved
   ↓
available
```

If successfully paid:

``` text
reserved
   ↓
WITHDRAWAL_RESERVED_DEBIT_PAYOUT
```

The exact compensating ledger records for rejected, failed, and reversed
withdrawals are:

``` text
WITHDRAWAL_RESERVED_DEBIT_RETURN     debit  reserved
WITHDRAWAL_AVAILABLE_CREDIT_RETURN   credit available
```

Provider communication is outside the database transaction but cannot change
the final accounting result until it has been durably associated with the
server-owned transfer attempt and verified against the withdrawal.

------------------------------------------------------------------------

# 18. Dispute Tables

## 18.1 `disputes`

  Field                    Type
  ------------------------ -------------------------
  id                       PK
  booking_id               FK
  client_id                FK
  partner_id               FK
  reason_code              VARCHAR
  description              TEXT
  status                   VARCHAR
  raised_at                DATETIME
  resolved_at              DATETIME nullable
  resolved_by_admin_id     FK nullable
  resolution_type          VARCHAR nullable
  resolution_note          TEXT nullable
  partner_release_amount   BIGINT/DECIMAL nullable
  client_refund_amount     BIGINT/DECIMAL nullable
  created_at               DATETIME
  updated_at               DATETIME

Description must satisfy the 100-character minimum.

Statuses:

``` text
open
under_review
resolved
closed
```

The booking itself becomes `disputed` while the dispute is active.

------------------------------------------------------------------------

## 18.2 `dispute_notes`

  Field           Type
  --------------- ----------
  id              PK
  dispute_id      FK
  admin_user_id   FK
  note            TEXT
  created_at      DATETIME

------------------------------------------------------------------------

## 18.3 `dispute_evidence`

Although structured evidence submission is listed as a Phase 2
improvement, the schema may support it without exposing the feature in
Phase 1.

  Field               Type
  ------------------- ---------------
  id                  PK
  dispute_id          FK
  submitted_by_type   VARCHAR
  submitted_by_id     VARCHAR
  file_id             FK nullable
  description         TEXT nullable
  created_at          DATETIME

Do not expose Phase 2 behavior before it is approved.

------------------------------------------------------------------------

# 19. At Risk Tables

## 19.1 `at_risk_flags`

  Field                  Type
  ---------------------- -------------------
  id                     PK
  booking_id             FK
  partner_id             FK
  reason_code            VARCHAR
  description            TEXT
  status                 VARCHAR
  created_at             DATETIME
  resolved_at            DATETIME nullable
  resolved_by_admin_id   FK nullable

Description minimum:

`100 characters`

Statuses:

``` text
open
under_review
resolved
cancelled
```

At Risk is an operational condition and must not replace the core
Booking status.

------------------------------------------------------------------------

# 20. Futuristic Booking and Follow-Up Tables

## 20.1 `futuristic_bookings`

A booking is futuristic when the gap between creation and service start
is 30 or more calendar days.

  Field              Type
  ------------------ -----------
  id                 PK
  booking_id         FK unique
  service_start_at   DATETIME
  created_at         DATETIME

------------------------------------------------------------------------

## 20.2 `booking_follow_ups`

  Field               Type
  ------------------- -------------------
  id                  PK
  booking_id          FK
  follow_up_type      VARCHAR
  due_at              DATETIME
  status              VARCHAR
  assigned_admin_id   FK nullable
  completed_at        DATETIME nullable
  completion_note     TEXT nullable
  created_at          DATETIME

Follow-up types:

``` text
30_day
14_day
3_day
24_hour
```

The PRD requires 30/14/3-day futuristic booking follow-ups and a 24-hour
reminder for every booking.

------------------------------------------------------------------------

# 21. Wishlist Tables

## 21.1 `wishlists`

  Field        Type
  ------------ -----------
  id           PK
  client_id    FK unique
  created_at   DATETIME
  updated_at   DATETIME

------------------------------------------------------------------------

## 21.2 `wishlist_items`

  Field         Type
  ------------- ----------
  wishlist_id   FK
  listing_id    FK
  created_at    DATETIME

Primary key:

``` text
(wishlist_id, listing_id)
```

------------------------------------------------------------------------

# 22. Inquiry Tables

High-value categories require pre-booking inquiry capability.

## 22.1 `inquiries`

  Field         Type
  ------------- -------------
  id            PK
  client_id     FK nullable
  listing_id    FK
  category_id   FK
  subject       VARCHAR
  message       TEXT
  status        VARCHAR
  created_at    DATETIME
  updated_at    DATETIME

Statuses:

``` text
open
in_progress
converted
closed
```

The exact threshold for high-value Event Hall bookings is not defined in
the supplied PRD and must remain configurable.

------------------------------------------------------------------------

# 23. Communication Tables

## 23.1 `conversations`

Shared conversation entity.

  Field               Type
  ------------------- -------------------
  id                  PK
  conversation_type   VARCHAR
  booking_id          FK nullable
  client_id           FK nullable
  partner_id          FK nullable
  status              VARCHAR
  assigned_admin_id   FK nullable
  created_at          DATETIME
  updated_at          DATETIME
  closed_at           DATETIME nullable

Conversation types:

``` text
client_support
partner_admin
booking_high_ticket
```

------------------------------------------------------------------------

## 23.2 `conversation_participants`

  Field              Type
  ------------------ -------------------
  conversation_id    FK
  participant_type   VARCHAR
  participant_id     VARCHAR
  joined_at          DATETIME
  left_at            DATETIME nullable

------------------------------------------------------------------------

## 23.3 `messages`

  Field             Type
  ----------------- -------------------
  id                PK
  conversation_id   FK
  sender_type       VARCHAR
  sender_id         VARCHAR
  message_type      VARCHAR
  body              TEXT
  created_at        DATETIME
  edited_at         DATETIME nullable
  deleted_at        DATETIME nullable

Client support replies must be presented as HandyPros brand messages
rather than exposing individual staff identities where the product
requires brand attribution.

------------------------------------------------------------------------

## 23.4 `message_attachments`

  Field         Type
  ------------- ----------
  id            PK
  message_id    FK
  file_name     VARCHAR
  mime_type     VARCHAR
  storage_key   TEXT
  file_url      TEXT
  file_size     BIGINT
  created_at    DATETIME

------------------------------------------------------------------------

## 23.5 `message_acknowledgements`

For Partner-admin documents requiring acknowledgement.

  Field             Type
  ----------------- -------------
  id                PK
  message_id        FK
  booking_id        FK nullable
  partner_id        FK
  acknowledged_at   DATETIME
  created_at        DATETIME

The PRD explicitly requires MOU/document acknowledgement to be
timestamped against the booking.

------------------------------------------------------------------------

# 24. Notification Tables

The current ordered target migration chain is `001` through `016`.
`014_phase9_notification_delivery.sql` creates the Phase 9 notification
delivery additions and versioned application template registry.
`015_phase9_booking_cancellation_template.sql` is an additive reference-data
migration that adds the explicit `booking_cancelled` email and WhatsApp
version-1 templates. `016_phase10_admin_operations.sql` adds the durable
operational-alert fact table used by the Admin control centre.

## 24.1 `notifications`

Represents a durable business-notification/outbox fact. It is created with the
business transaction but provider I/O happens only after commit. The unique
event key includes the semantic recipient, so replaying a business event does
not create uncontrolled duplicate messages while one event may still inform
multiple different recipients with distinct keys. Phase 8 writes withdrawal
facts here; Phase 9 owns asynchronous channel selection, attempts, retry,
fallback, and delivery receipts.

  Field                       Target type / rule
  --------------------------- ------------------------------------------
  id                          CHAR(26) primary key
  event_key                   VARCHAR(180) unique
  event_type                  VARCHAR(100)
  template_key                VARCHAR(120) nullable
  template_version            INT UNSIGNED nullable for pre-Phase 9 facts;
                               persisted active version for new Phase 9 facts
  recipient_type              VARCHAR(30)
  recipient_id                CHAR(26) nullable opaque identity
  recipient_name              VARCHAR(190) nullable, server-only
  recipient_phone             VARCHAR(32) nullable, server-only
  recipient_email             VARCHAR(190) nullable, server-only
  withdrawal_id               CHAR(26) nullable FK
  entity_type                 VARCHAR(40) nullable
  entity_id                   CHAR(26) nullable
  booking_id                  CHAR(26) nullable
  payment_id                  CHAR(26) nullable
  listing_id                  CHAR(26) nullable
  status                      VARCHAR(30)
  primary_channel             VARCHAR(30), defaults to `whatsapp`
  current_channel             VARCHAR(30) nullable
  fallback_channel            VARCHAR(30), defaults to `email`
  fallback_status             VARCHAR(30), defaults to `none`
  payload_json                JSON safe context only
  private_payload_encrypted   MEDIUMTEXT nullable
  next_attempt_at             DATETIME(6) nullable
  last_attempt_at             DATETIME(6) nullable
  sent_at                     DATETIME(6) nullable
  delivered_at                DATETIME(6) nullable
  read_at                     DATETIME(6) nullable
  failed_at                   DATETIME(6) nullable
  delivery_attempt_count      INT UNSIGNED, defaults to 0
  last_failure_reason         VARCHAR(255) nullable, safe code only
  created_at                  DATETIME(6)
  updated_at                  DATETIME(6) nullable

Notification statuses:

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

Current emitted event types:

``` text
auth.otp_requested
partner.onboarding_submitted
listing.submitted
listing.approved
listing.changes_requested
listing.suspended
booking.payment_required
booking.created
booking.cancelled
booking.confirmed
booking.reminder
booking.active
payment.successful
payment.failed
travel.balance_due
travel.balance_paid
settlement.entitlement_pending
settlement.funds_released
booking.completed
withdrawal.requested
withdrawal.approved
withdrawal.held
withdrawal.rejected
withdrawal.processing
withdrawal.successful
withdrawal.failed
withdrawal.reversed
```

Phase 8 uses deterministic withdrawal keys such as
`withdrawal:{withdrawal_id}:requested:partner`. This makes a transaction
retry/replayed provider fact harmless without claiming delivery success. The
payload contains safe IDs/references only; it must not contain plaintext OTPs,
bank details, encrypted account values, provider credentials, or raw provider
request/response bodies. Recipient name/phone/email are access-controlled
delivery-only fields and are not returned by Client/Partner history APIs;
private template context is held in `private_payload_encrypted` rather than
the ordinary safe payload.

For OTP delivery, the hash-only OTP verification record remains the source of
truth. A plaintext OTP, if required transiently by the provider renderer, is
not notification payload/attempt/webhook/audit/log data and is never exposed
to a web/mobile API. The encrypted private context is cleared after provider
acceptance, successful email fallback, or terminal delivery failure. The
scheduled notification worker also clears residual `auth_otp` context once the
linked `user_otp_verifications.expires_at` has passed; this cleanup never
changes the hash-only verification record or its attempt limits. An OTP attempt
is eligible only while the linked verification remains unverified and
unexpired. Verifying it suppresses any queued/processing notification and
clears the encrypted context in the same transaction. `auth_otp` facts are not
eligible for Admin resend.

------------------------------------------------------------------------

## 24.2 `notification_attempts`

`notification_attempts` is append-only delivery history. It is never
overwritten to hide a failed attempt, retry, or fallback.

  Field                    Target type / rule
  ------------------------ ------------------------------------------
  id                       CHAR(26) primary key
  notification_id          CHAR(26) FK
  channel                  VARCHAR(30)
  provider                 VARCHAR(60)
  status                   VARCHAR(30)
  provider_message_id      VARCHAR(180) nullable
  error_message            TEXT nullable, safe failure code only
  attempt_number           INT UNSIGNED
  queued_at                DATETIME(6) nullable
  sent_at                  DATETIME(6) nullable
  delivered_at             DATETIME(6) nullable
  read_at                  DATETIME(6) nullable
  failed_at                DATETIME(6) nullable
  provider_status          VARCHAR(80) nullable
  provider_metadata_json   JSON nullable, sanitized
  retryable                BOOLEAN, defaults to false
  created_at               DATETIME(6)
  updated_at               DATETIME(6) nullable

Attempt statuses:

``` text
queued
sending
sent
delivered
read
failed
```

Channels:

``` text
whatsapp
email
```

There is one attempt identity per notification/channel/attempt number. A
non-null provider message identity is indexed for delivery-receipt lookup.
WhatsApp is primary. Email is a fallback attempt linked to the same
notification only after final WhatsApp failure; successful WhatsApp does not
cause a default email duplicate. Provider status receipts can advance only a
matching existing attempt and must be idempotent/out-of-order safe.

------------------------------------------------------------------------

## 24.3 `notification_templates`

  Field                  Target type / rule
  ---------------------- ------------------------------------------
  id                     CHAR(26) primary key
  template_key           VARCHAR(120)
  channel                VARCHAR(30)
  version                INT UNSIGNED
  provider_template_id   VARCHAR(180) nullable
  language_code          VARCHAR(20) nullable
  subject_template       VARCHAR(255) nullable
  body_template          TEXT
  status                 VARCHAR(30)
  created_at             DATETIME(6)
  updated_at             DATETIME(6)

`(template_key, channel, version)` is unique. WhatsApp provider template
approval must not be assumed to equal application template existence. The
reference seed contains application email and WhatsApp rows; a real Meta send
requires an eligible configured `provider_template_id` (except the separately
configured approved OTP template). The `notifications.template_version` field
captures the active version selected when the business event is persisted; the
worker resolves that exact version on delivery, retry, and eligible resend
instead of silently selecting a newer registry version.

------------------------------------------------------------------------

## 24.4 `whatsapp_webhook_events`

Stores sanitized, deduplicated Meta webhook facts separately from delivery
attempt history.

  Field                    Target type / rule
  ------------------------ ------------------------------------------
  id                       CHAR(26) primary key
  provider                 VARCHAR(40)
  provider_event_id        VARCHAR(180)
  provider_message_id      VARCHAR(180) nullable
  notification_attempt_id  CHAR(26) nullable FK
  event_type               VARCHAR(80)
  delivery_status          VARCHAR(40) nullable
  signature_verified       BOOLEAN
  payload_hash             CHAR(64)
  payload_json             JSON sanitized
  processing_status        VARCHAR(30)
  processed_at             DATETIME(6) nullable
  error_message            VARCHAR(255) nullable, safe code only
  created_at               DATETIME(6)

`(provider, provider_event_id)` is unique. The implementation derives a
stable receipt identity from each inbound Meta status fact so identical
deliveries are harmless. Invalid-signature evidence records only a hash and a
safe error code; malformed payloads are rejected without a business-state
transition. Unknown provider message IDs can be retained as sanitized
operational evidence but never create a notification or attempt. Raw headers,
access tokens, app secrets, full recipient contacts, and raw provider payloads
are never stored.

------------------------------------------------------------------------

# 25. Premium Subscription Tables

## 25.1 `subscription_plans`

  Field              Type
  ------------------ ----------------
  id                 PK
  name               VARCHAR
  price              BIGINT/DECIMAL
  currency           CHAR(3)
  billing_interval   VARCHAR
  active             BOOLEAN
  created_at         DATETIME
  updated_at         DATETIME

------------------------------------------------------------------------

## 25.2 `partner_subscriptions`

  Field                  Type
  ---------------------- -------------------
  id                     PK
  partner_id             FK
  plan_id                FK
  provider               VARCHAR
  provider_reference     VARCHAR nullable
  status                 VARCHAR
  current_period_start   DATETIME
  current_period_end     DATETIME
  cancelled_at           DATETIME nullable
  created_at             DATETIME
  updated_at             DATETIME

The PRD requires premium subscription to remain optional and not affect
basic listing visibility.

------------------------------------------------------------------------

# 26. Audit Tables

## 26.1 `audit_events`

Append-only.

  Field           Type
  --------------- ------------------
  id              PK
  actor_type      VARCHAR
  actor_id        VARCHAR nullable
  action          VARCHAR
  entity_type     VARCHAR
  entity_id       VARCHAR
  before_json     JSON nullable
  after_json      JSON nullable
  metadata_json   JSON nullable
  ip_address      VARCHAR nullable
  user_agent      TEXT nullable
  created_at      DATETIME

Audit at minimum:

-   Admin full bank-account access
-   Partner approval
-   Listing approval/rejection/changes requested
-   Commission rate changes
-   Dispute resolution
-   Refund initiation
-   Withdrawal approval/hold/rejection
-   Reconciliation adjustments
-   Manual financial adjustments
-   Admin account/security actions

Audit records cannot be edited or deleted.

------------------------------------------------------------------------

# 27. System Alert Tables

## 27.1 `system_alerts`

`016_phase10_admin_operations.sql` owns this durable operational-fact table.
It is intentionally separate from the underlying financial/provider records:
acknowledging or resolving an alert never changes a payment, withdrawal,
notification, ledger, or reconciliation record.

  Field                      Type / rule
  -------------------------- -----------------------------------------------
  id                         PK
  alert_type                 VARCHAR
  source_key                 VARCHAR, unique deterministic source identity
  severity                   VARCHAR
  entity_type                VARCHAR nullable
  entity_id                  VARCHAR nullable
  title                      VARCHAR
  message                    TEXT
  status                     `open` / `acknowledged` / `resolved`
  created_at / updated_at    DATETIME(6)
  acknowledged_at            DATETIME(6) nullable
  acknowledged_by_admin_id   FK nullable
  resolved_at                DATETIME(6) nullable
  resolved_by_admin_id       FK nullable
  resolution_reason          TEXT nullable

The acknowledgement FK pair and resolution FK pair are constrained so a
timestamp cannot exist without its responsible Admin. `source_key` prevents
dashboard/read retries from manufacturing duplicate alerts. Resolved facts
are retained rather than silently reopened by a read operation.

Alert types include:

``` text
missing_commission_rate
payment_webhook_failure
transfer_webhook_failure
withdrawal_failed
withdrawal_provider_uncertain
notification_delivery_failed
payment_verification_failed
stale_pending_payment
reconciliation_mismatch
```

------------------------------------------------------------------------

# 28. Reconciliation Tables

## 28.1 `reconciliation_runs`

  Field                    Type
  ------------------------ -------------------
  id                       PK
  provider                 VARCHAR
  run_date                 DATE
  started_at               DATETIME
  completed_at             DATETIME nullable
  status                   VARCHAR
  total_provider_records   INT
  matched_records          INT
  unmatched_records        INT
  mismatch_count           INT
  created_at               DATETIME

------------------------------------------------------------------------

## 28.2 `reconciliation_records`

  Field                  Type
  ---------------------- -------------------------
  id                     PK
  run_id                 FK
  provider               VARCHAR
  provider_reference     VARCHAR
  internal_entity_type   VARCHAR nullable
  internal_entity_id     VARCHAR nullable
  provider_amount        BIGINT/DECIMAL
  internal_amount        BIGINT/DECIMAL nullable
  status                 VARCHAR
  resolution_note        TEXT nullable
  resolved_by_admin_id   FK nullable
  resolved_at            DATETIME nullable
  created_at             DATETIME

Statuses:

``` text
matched
unmatched
amount_mismatch
missing_internal_record
resolved
```

The PRD requires daily financial reconciliation against Paystack
transaction records.

------------------------------------------------------------------------

# 29. File Tables

## 29.1 `files`

Generic metadata for uploaded documents.

  Field              Type
  ------------------ -------------------
  id                 PK
  uploaded_by_type   VARCHAR
  uploaded_by_id     VARCHAR
  storage_provider   VARCHAR
  storage_key        TEXT
  original_name      VARCHAR
  mime_type          VARCHAR
  size_bytes         BIGINT
  checksum           VARCHAR nullable
  created_at         DATETIME
  deleted_at         DATETIME nullable

Do not store large binary files directly in the relational database
unless there is a specific approved requirement.

------------------------------------------------------------------------

# 30. Receipt Tables

## 30.1 `receipts`

  Field            Type
  ---------------- ----------------
  id               PK
  booking_id       FK unique
  receipt_number   VARCHAR unique
  file_id          FK nullable
  generated_at     DATETIME
  created_at       DATETIME

Receipt contents should be generated from authoritative
booking/payment/commission data.

------------------------------------------------------------------------

# 31. Idempotency Table

## 31.1 `idempotency_keys`

  Field             Type
  ----------------- ----------
  id                PK
  key               VARCHAR
  actor_type        VARCHAR
  actor_id          VARCHAR
  endpoint          VARCHAR
  request_hash      VARCHAR
  response_status   INT
  response_body     JSON
  created_at        DATETIME
  expires_at        DATETIME

Unique constraint:

``` text
(actor_type, actor_id, key, endpoint)
```

Use this for financial and other non-repeatable commands.

For Phase 8, the scope includes Partner payout-account creation, Partner
withdrawal creation, and each Full Admin withdrawal command. An identical
completed command replays its stored response; a different request hash for
the same actor/scope/key is a conflict. A payout-account row may intentionally
remain `provider_pending` while an external recipient result is unknown; that
state fails closed and must not cause a second recipient-creation call.

------------------------------------------------------------------------

# 32. Critical Relationships

## User → Partner

``` text
users 1 ─── 0..1 partners
```

## Partner → Listings

``` text
partners 1 ─── N listings
```

## Category → Listings

``` text
service_categories 1 ─── N listings
```

## Listing → Pricing

``` text
listings 1 ─── N pricing_rules
```

## Listing → Availability

``` text
listings 1 ─── N availability_days
listings 1 ─── N availability_slots
```

## Client → Bookings

``` text
users 1 ─── N bookings
```

## Partner → Bookings

``` text
partners 1 ─── N bookings
```

## Booking → Payments

``` text
bookings 1 ─── N payments
```

This is required for Travel Package deposit + balance payments.

## Booking → Commission

``` text
bookings 1 ─── N commission_records
```

This is required because Travel Packages can generate separate
commission records for deposit and balance.

## Booking → Dispute

``` text
bookings 1 ─── 0..N disputes
```

Business rules should normally allow only one active dispute.

## Partner → Ledger

``` text
partners 1 ─── 1 partner_ledger_accounts
```

## Ledger Account → Ledger Entries

``` text
partner_ledger_accounts 1 ─── N partner_ledger_entries
```

------------------------------------------------------------------------

# 33. Booking State Machine Persistence

The database must support:

``` text
PENDING_PAYMENT
      │
      │ Paystack confirmed
      ▼
UPCOMING
      │
      │ service starts
      ▼
ACTIVE
   ┌──┴───────────────┐
   │                  │
   │ dispute          │ release
   ▼                  ▼
DISPUTED          COMPLETED
   │
   │ admin decision
   ├───────────────► COMPLETED
   │
   └───────────────► REFUNDED

PENDING_PAYMENT / UPCOMING
      │
      │ cancellation
      ▼
CANCELLED
```

The exact transition authorization belongs in the Booking Service.

The database must not permit arbitrary status mutation from the
frontend.

------------------------------------------------------------------------

# 34. Auto-Release Persistence

At booking confirmation:

``` text
category.auto_release_hours
          +
service trigger datetime
          ↓
booking.auto_release_at
```

The resulting `auto_release_at` must be stored on the booking.

Do not recalculate it later from the current category configuration.

This protects historical bookings when an Admin changes future category
rules.

------------------------------------------------------------------------

# 35. Auto-Release Transaction

The release operation must be atomic from the database perspective.

Within one database transaction:

``` text
1. Lock booking row.
2. Verify booking is eligible.
3. Verify no active dispute.
4. Verify release has not already happened.
5. Set booking status = completed.
6. Set completed_at.
7. Create payout_event.
8. Create Partner ledger entry.
9. Move appropriate Partner balance from pending to available.
10. Commit.
```

If the transaction fails:

-   booking remains unreleased;
-   Partner balance remains unchanged;
-   an admin system alert is created;
-   scheduler retries safely.

Do not send the external WhatsApp notification before the financial
transaction is committed.

------------------------------------------------------------------------

# 36. Confirm Experience Transaction

When Client selects Confirm Experience:

``` text
1. Lock booking.
2. Verify Client owns booking.
3. Verify booking is Active.
4. Verify no active dispute.
5. Verify not already released.
6. Calculate/use frozen net payout.
7. Mark booking Completed.
8. Create payout event with trigger = client_confirmed.
9. Create ledger entries.
10. Commit.
11. Dispatch notifications.
```

Repeated Confirm Experience requests must not produce another payout.

------------------------------------------------------------------------

# 37. Dispute Transaction

When a Client raises a dispute:

``` text
1. Lock booking.
2. Verify Client ownership.
3. Verify Active state.
4. Verify auto-release window has not expired.
5. Verify no active dispute.
6. Create dispute.
7. Change booking to Disputed.
8. Prevent auto-release.
9. Commit.
10. Dispatch Partner notification.
11. Create admin alert.
```

The notification is not part of the database transaction.

------------------------------------------------------------------------

# 38. Withdrawal Transaction

When Partner requests withdrawal:

``` text
1. Lock Partner ledger account.
2. Reconcile cached available/pending/reserved balances to append-only ledger entries.
3. Verify a Partner-owned, verified payout account with a server-created recipient.
4. Apply a minimum only if a positive active policy exists.
5. Verify available balance >= requested amount.
6. Create withdrawal.
7. Create available debit and reserved credit ledger entries.
8. Decrease available cached balance.
9. Increase reserved cached balance.
10. Write safe audit/notification facts and idempotent response.
11. Commit.
```

No Paystack Transfer API call should occur until the application has
successfully reserved the funds.

When Admin approves:

``` text
1. Lock withdrawal.
2. Verify pending-review or held state and Full Admin authority.
3. Persist one server-generated transfer attempt/reference.
4. Commit approved durable state.
5. Initiate Paystack transfer workflow outside the transaction.
6. Persist processing/unknown provider state without releasing the reservation.
```

Signed provider webhook facts and the reconciliation worker re-fetch/validate
the transfer before they determine a terminal state. Verified success moves
reserved to paid-out once. Verified failure/reversal creates compensating
reserved-to-available entries once. An unknown provider result remains
reserved until reconciliation resolves it.

------------------------------------------------------------------------

# 39. Commission Immutability

At payment confirmation:

``` text
booking gross amount
        ↓
category
        ↓
current commission rate
        ↓
CommissionRecord
        ↓
gross
commission
net payout
```

After creation, the CommissionRecord cannot be edited.

If Admin changes:

``` text
10% → 12%
```

then:

``` text
Existing confirmed booking
10% remains

New booking
12%
```

The PRD explicitly requires this behavior.

------------------------------------------------------------------------

# 40. Financial Invariants

The following must always hold.

## 40.1 Commission

``` text
commission_amount
=
gross_amount × frozen_commission_rate
```

Subject to the chosen monetary rounding rule.

## 40.2 Partner net

``` text
net_partner_payout
=
gross_amount - commission_amount
```

## 40.3 Partner balance

At minimum:

``` text
available + pending + reserved
```

must reconcile with the ledger's outstanding balance categories.

## 40.4 No double payout

For a booking:

``` text
total_release_events
```

must never cause more than the authorized net payout to become
available.

## 40.5 No withdrawal overdraft

``` text
available_balance
>=
withdrawal_amount
```

must be true while the withdrawal reservation transaction executes.

## 40.6 Withdrawal terminality

For each withdrawal, the append-only ledger must contain exactly one of the
following terminal outcomes:

``` text
success  -> one reserved debit payout
rejected -> one reserved debit return + one available credit return
failed   -> one reserved debit return + one available credit return
reversed -> one reserved debit return + one available credit return
```

An unknown or pending provider result is not terminal and keeps the funds in
the reserved bucket. Unique withdrawal/entry-type and provider-event keys,
plus the locked withdrawal/account transition, enforce this invariant against
duplicate Admin commands, webhooks, and reconciliation runs.

------------------------------------------------------------------------

# 41. Indexing Requirements

At minimum index:

## Users

``` text
users.phone
users.email
users.role
users.status
```

## Partners

``` text
partners.status
partners.user_id
```

## Listings

``` text
listings.partner_id
listings.category_id
listings.city_id
listings.status
listings.visibility_status
listings.slug
```

## Availability

``` text
availability_days(listing_id, date)
availability_slots(listing_id, slot_date)
```

## Bookings

``` text
bookings.booking_reference
bookings.client_id
bookings.partner_id
bookings.listing_id
bookings.status
bookings.service_start_at
bookings.auto_release_at
```

## Payments

``` text
payments.provider_reference
payments.booking_id
payments.status
```

## Ledger

``` text
partner_ledger_entries.partner_id
partner_ledger_entries.booking_id
partner_ledger_entries.created_at
```

## Withdrawals

``` text
withdrawals.partner_id
withdrawals.status
withdrawals.created_at
withdrawals.provider_reference
```

## Disputes

``` text
disputes.booking_id
disputes.status
disputes.created_at
```

## Notifications

``` text
notifications.recipient_id
notifications.event_key
notifications.status
notifications.next_attempt_at
notifications.entity_type + notifications.entity_id
notifications.booking_id
notifications.payment_id
notifications.listing_id
notifications.withdrawal_id
notification_attempts.notification_id
notification_attempts.notification_id + notification_attempts.channel + notification_attempts.attempt_number
notification_attempts.provider_message_id
notification_attempts.status
whatsapp_webhook_events.provider + provider event identity
whatsapp_webhook_events.provider_message_id
```

------------------------------------------------------------------------

# 42. Concurrency and Locking

Double booking is a critical failure.

For date-based accommodation:

-   lock the relevant inventory rows or use an equivalent
    transaction-safe mechanism;
-   check existing confirmed/pending locks;
-   create booking inventory lock;
-   commit together with booking state.

For slot-based experiences:

-   lock the selected slot row;
-   verify remaining capacity;
-   reserve capacity transactionally.

For financial operations:

-   lock the Partner ledger account before balance movement;
-   lock the booking before release;
-   lock the withdrawal before changing withdrawal state.

Do not rely on frontend availability alone.

------------------------------------------------------------------------

# 43. Booking Expiration

The PRD states that a pending payment booking expires after 30 minutes
if payment is abandoned.

The database must support:

``` text
bookings.status = pending_payment
```

and an expiration field such as:

``` text
payment_expires_at
```

When the expiry job runs:

1.  lock booking;
2.  verify still pending payment;
3.  release inventory lock;
4.  mark booking expired/cancelled according to the final API state
    model;
5.  record status history.

The exact public status label for expired unpaid bookings should be
finalized in the implementation/API contract.

------------------------------------------------------------------------

# 44. Data Retention

Financial and audit records should be retained according to applicable
legal, tax, accounting, and privacy requirements.

The database must not implement automatic deletion of:

-   payments
-   refunds
-   commission records
-   payout events
-   ledger entries
-   withdrawals
-   reconciliation records
-   audit events

User privacy deletion requests must be handled through a controlled
anonymization/retention process rather than deleting records that are
legally required for financial auditability.

------------------------------------------------------------------------

# 45. Existing Production Data

The PRD states that an existing HandyPros application already has:

-   live Partner listings
-   active Client accounts
-   historical booking records

and that new work must preserve data integrity and existing
functionality.

Therefore the implementation must NOT:

-   drop production tables blindly;
-   reset IDs;
-   delete existing bookings;
-   replace financial history;
-   migrate data without a backup;
-   assume an empty database.

Before production migration:

``` text
Backup
  ↓
Schema migration
  ↓
Data mapping
  ↓
Validation
  ↓
Reconciliation
  ↓
Application cutover
```

The existing production schema must be inspected before writing
migration scripts.

------------------------------------------------------------------------

# 46. Phase 1 vs Phase 2 Schema

Phase 1 database support is required for:

-   core marketplace
-   booking
-   payments
-   commission
-   auto-release
-   disputes
-   withdrawals
-   notifications
-   support chat
-   Partner-admin chat
-   premium subscriptions
-   follow-ups
-   audit
-   reconciliation
-   PDF receipts
-   wishlist
-   inquiry
-   At Risk

Phase 2 source requirements include:

-   structured dispute evidence
-   post-service ratings
-   SMS fallback
-   iCal synchronization
-   Admin TOTP 2FA
-   enhanced seasonal campaign behavior
-   Partner earnings analytics

Phase 2 tables may be created in advance only when doing so does not
create premature product behavior.

------------------------------------------------------------------------

# 47. Database Migration Rules

Every schema change must be versioned.

Example:

``` text
001_create_users
002_create_partners
003_create_categories
004_create_listings
005_create_availability
006_create_bookings
007_create_payments
008_create_commission
009_create_ledger
010_create_withdrawals
...
```

Never modify production schema manually without a migration.

Every migration must be:

-   reversible where practical;
-   tested on staging;
-   safe against existing data;
-   documented.

------------------------------------------------------------------------

# 48. Seed Data Requirements

Initial seed data must include:

## Cities

``` text
Lagos
Lekki/Ajah
Abuja
Port Harcourt
```

## Verticals

``` text
Places to Stay
Things to Do
Rentals and Experiences
```

## Categories

The current approved launch set contains the 17 explicitly defined categories.
The PRD's two undefined category slots remain deferred until Product supplies
their definitions; no placeholder category or invented release rule is seeded.

## Auto-release windows

``` text
Spa / Gym / Tennis / Netball / Paddle / Swimming / Wine Tasting
→ 24 hours after session end

Hotel / Short-Stay Apartment / Beach-House Apartment
→ 24 hours after check-in datetime

Event Hall / Boat Cruise / House Party
→ 48 hours after event start datetime

Travel Package
→ 72 hours after departure date

Car Rental with Driver / Drive-Yourself / Private Jet
→ 24 hours after rental/charter start
```

These are source-derived product rules and should be seeded as
configurable category settings.

------------------------------------------------------------------------

# 49. Database Acceptance Criteria

The database implementation is complete only when:

### Identity

-   Client accounts work.
-   Partner accounts work.
-   Admin accounts are isolated.
-   OTP records are secure.

### Marketplace

-   All four cities exist.
-   All 17 explicitly defined launch categories exist; the PRD's two
    undefined category slots remain deferred without placeholders.
-   Partner listings can be represented.
-   Category-specific content is supported.
-   Listing approval history is preserved.
-   Only active listings can appear publicly.

### Availability

-   Date availability works.
-   Time-slot availability works.
-   Inventory locking prevents double booking.

### Booking

-   All required booking states are supported.
-   Status history is recorded.
-   Payment expiration is supported.
-   Service dates are immutable after confirmation unless a controlled
    rescheduling feature is explicitly introduced.

### Payments

-   Multiple payments per booking are supported.
-   Travel deposit + balance are supported.
-   Paystack references are unique.
-   Webhook events are idempotent.
-   Refunds are auditable.

### Commission

-   Commission rate is snapshot/frozen.
-   CommissionRecord is immutable.
-   Historical bookings survive future rate changes.

### Settlement

-   Partner ledger exists.
-   Release events are recorded.
-   Auto-release is idempotent.
-   Confirm Experience is idempotent.
-   Disputes block release.

### Withdrawals

-   Available funds can be reserved.
-   Withdrawal lifecycle is persisted.
-   Provider references are stored.
-   Failed withdrawals restore reserved funds exactly once.
-   Partner bank numbers are encrypted and normal API output is masked.
-   A transfer attempt and signed provider-event identity are unique.
-   Unknown provider outcomes retain the reservation pending reconciliation.

### Communication

-   Conversation history is permanent.
-   Attachments are represented.
-   Notification events are unique per semantic event/recipient and durable
    before dispatch.
-   Notification attempts and Meta webhook receipts are append-only and
    replay-safe.
-   WhatsApp failures can be followed by one bounded-retry email fallback.
-   Notification failure cannot roll back a booking, payment, settlement,
    dispute, refund, or withdrawal result.
-   Plaintext OTPs, full bank data, provider credentials, and raw provider
    request/response bodies are excluded from notification history/API/log
    payloads.

### Operations

-   At Risk flags exist.
-   Futuristic booking follow-ups exist.
-   Admin alerts exist.
-   Reconciliation exists.
-   Audit events are append-only.

------------------------------------------------------------------------

# 50. Required Database Tests

Before Phase 1 sign-off, automated tests must cover:

1.  duplicate phone registration;
2.  duplicate email where applicable;
3.  unauthorized Partner access;
4.  unauthorized Admin access;
5.  listing visibility;
6.  listing approval;
7.  seasonal pricing snapshot;
8.  minimum stay;
9.  double booking attempt;
10. abandoned payment expiration;
11. successful Paystack payment;
12. duplicate Paystack webhook;
13. incorrect Paystack signature;
14. commission calculation;
15. commission immutability;
16. auto-release;
17. duplicate auto-release;
18. Confirm Experience;
19. duplicate Confirm Experience;
20. dispute pause;
21. duplicate dispute;
22. withdrawal reservation;
23. withdrawal overdraft prevention;
24. duplicate withdrawal request;
25. failed withdrawal balance restoration;
25a. duplicate transfer-webhook/reconciliation finalization;
25b. concurrent Admin approval creates one transfer attempt;
25c. unverified/foreign payout bank cannot be used;
25d. unknown provider transfer outcome remains reserved;
26. dispute resolution;
27. refund record creation;
28. notification retry;
29. audit record creation;
30. daily reconciliation mismatch detection.

------------------------------------------------------------------------

# 51. Explicit Gaps Not Defined by the PRD

The source PRD does not fully specify the following database-level
choices.

They must be finalized before implementation rather than silently
invented:

1.  Exact database engine/version.
2.  Exact public ID strategy.
3.  Exact monetary representation.
4.  Exact retention periods.
5.  Exact cancellation/refund matrix.

------------------------------------------------------------------------

# Phase 11 implemented protection schema

The ordered target chain now ends at `018_phase11_dispute_response.sql`.
`017_phase11_protection_financials.sql` is additive and introduces no
production cancellation-policy seed. It adds:

- `cancellation_policies`: versioned, effective-dated, category/actor/
  booking-state/payment-state/timing rules, with `full_refund`,
  `partial_refund`, `no_refund`, or `admin_review` outcomes;
- `booking_cancellation_quotes` and `booking_cancellations`: one immutable
  calculation/provenance and one cancellation per booking;
- refund provenance and provider lifecycle fields, recovery obligations, and
  deduplicated `refund_provider_events`;
- append-only refund-linked Partner ledger entries;
- active-dispute booking hold, resolution fields, private evidence and notes.

`018` adds `disputes.partner_response` without rewriting an already-applied
migration. `partner_recovery_obligations` prevents a post-settlement Client
refund from silently producing a negative Partner wallet. `refund_provider_events`
stores signed provider facts separately from payment and transfer events.
Refunds, CommissionRecords, ledger entries, payout events, evidence and audit
events remain historical facts; resolution is represented by new records or
append-only compensations, never a destructive update.
6.  Exact Partner withdrawal minimum.
7.  Exact Travel Package deposit percentage.
8.  Exact Travel Package balance rules.
9.  Exact high-value Event Hall threshold.
10. Exact category-specific JSON schemas for all 19 categories.
11. Exact existing production schema and migration mapping.
12. Exact Admin role list if additional roles exist in the separate
    Admin PRD.
13. Exact premium subscription price/plan configuration.
14. Exact real-time chat infrastructure.
15. Exact file-storage provider.
16. Exact notification provider/BSP.

These are not reasons to stop architecture work. They are explicit
decision points that must be resolved before the affected implementation
phase.

------------------------------------------------------------------------

# 52. Codex Database Implementation Rules

Codex must treat this document as a database contract.

When implementing:

1.  Read the PRD.
2.  Read `HANDYPROS_ENGINEERING_SPECIFICATION.md`.
3.  Read `HANDYPROS_API_DOCUMENTATION.md`.
4.  Read this database specification.
5.  Inspect the existing database/schema before migration.
6.  Never destroy existing production data.
7.  Create migrations, not ad-hoc production SQL.
8.  Add foreign keys and indexes intentionally.
9.  Use transactions for booking and financial state transitions.
10. Add tests for every financial invariant.
11. Never store card data.
12. Never store plaintext passwords or OTPs.
13. Never modify immutable financial records.
14. Never trust frontend financial totals.
15. Never use a mutable Partner balance without a ledger trail.
16. Never allow duplicate webhook processing to create duplicate money.
17. Never allow duplicate release to create duplicate money.
18. Never allow withdrawal to exceed available balance.
19. Never expose sensitive bank information to unauthorized roles.
20. Do not invent unresolved business rules; flag them for approval.

------------------------------------------------------------------------

# 53. Final Architecture Chain

The complete engineering source of truth is now:

``` text
PRODUCT REQUIREMENT
        ↓
HANDYPROS_ENGINEERING_SPECIFICATION.md
        ↓
HANDYPROS_API_DOCUMENTATION.md
        ↓
HANDYPROS_DATABASE_SPECIFICATION.md
        ↓
CODE
        ↓
TESTS
        ↓
STAGING
        ↓
PRODUCTION
```

The Web UI is treated as the existing presentation/design reference.

The backend/database are the authoritative implementation.

The future native mobile application must consume the same API and must
not contain a separate implementation of booking, payment, commission,
settlement, or withdrawal business rules.

------------------------------------------------------------------------

# 54. Next Engineering Artifact

After this document is approved, the next artifact should be the **Fresh
Codex Master Implementation Prompt**.

That prompt must instruct Codex to:

1.  start from a clean engineering context;
2.  read all approved project documents;
3.  inspect the existing React UI only for visual/product continuity;
4.  build the backend/database from the specifications;
5.  implement one phase at a time;
6.  stop after each phase for verification;
7.  never invent unresolved business rules;
8.  never rewrite the financial architecture for convenience;
9.  expose the documented API for future mobile clients;
10. preserve the existing production data through controlled migration.
