# HANDYPROS HOSPITALITY

## Engineering Specification

**Document:** `HANDYPROS_ENGINEERING_SPECIFICATION.md`\
**Version:** 1.0\
**Status:** Engineering Baseline\
**Source Documents:** HandyPros Hospitality PRD and HandyPros
Hospitality Admin Dashboard PRD

------------------------------------------------------------------------

## 1. Purpose

This document converts the HandyPros Hospitality product requirements
into an implementation-level engineering contract.

The Product Requirements Documents remain the source of truth for
product scope, user experience, features, and business intent.

This document defines the engineering interpretation required to
implement those requirements consistently across the backend, web
application, mobile application, and Admin Dashboard.

The implementation must not invent new business rules where the PRDs
already define them.

Where the PRDs do not define an implementation detail, the engineering
team may choose a technically sound implementation, but must document
the decision rather than silently changing product behavior.

------------------------------------------------------------------------

# 2. Product System

HandyPros Hospitality is a managed hospitality marketplace connecting
Clients with Partners offering hospitality services.

The platform supports:

-   Client registration and discovery
-   Partner onboarding
-   Partner listing submission
-   Admin listing approval
-   Category-specific listings
-   Availability management
-   Booking
-   Paystack payment collection
-   Commission calculation
-   Protected Partner settlement
-   Auto-release
-   Early release through Confirm Experience
-   Disputes
-   Refunds
-   Partner withdrawals
-   WhatsApp operational messaging
-   Email fallback
-   Admin operations
-   Audit logging
-   Financial reporting

The existing PRD defines the primary booking lifecycle as:

`Pending Payment → Upcoming → Active → Completed`

with alternative transitions including cancellation, dispute, and
refund.

------------------------------------------------------------------------

# 3. System Actors

## 3.1 Client

A customer who:

-   creates an account
-   browses listings
-   checks availability
-   books services
-   makes payments
-   views bookings
-   confirms experience
-   raises disputes
-   receives notifications
-   manages profile information

## 3.2 Partner

A hospitality service provider who:

-   registers as a Partner
-   submits listings
-   manages listing information
-   manages availability
-   manages pricing
-   receives bookings
-   communicates through the platform
-   receives settlement after release
-   requests withdrawals

## 3.3 Full Admin

Full operational administrator with authority over:

-   Partners
-   listings
-   bookings
-   disputes
-   withdrawals
-   commission settings
-   refunds
-   reports
-   notifications
-   system health
-   audit logs
-   configuration

## 3.4 Customer Support

Support-focused administrative role with restricted permissions compared
with Full Admin.

Permissions must be enforced by backend authorization and must not
depend only on frontend visibility.

## 3.5 System

Automated system processes include:

-   Paystack webhook processing
-   auto-release scheduler
-   booking reminders
-   notification delivery
-   notification retry
-   financial reconciliation
-   operational alerts

------------------------------------------------------------------------

# 4. Architecture Principles

## 4.1 Backend is the source of truth

The backend owns:

-   authentication state
-   authorization
-   listing state
-   availability
-   booking state
-   payment state
-   commission
-   settlement
-   Partner balance
-   withdrawals
-   disputes
-   refunds
-   notification state

Neither the React web application nor the mobile application may
independently determine authoritative financial or booking state.

## 4.2 API-first architecture

The system must expose versioned APIs for both:

-   Web
-   Mobile

The web and mobile clients consume the same business APIs.

Recommended API namespace:

`/api/v1/...`

## 4.3 External integrations are isolated

Paystack and Meta Business/WhatsApp integrations must be isolated behind
application services.

Business logic must not be tightly coupled to provider-specific SDK
calls.

## 4.4 Financial operations are transactional

Any operation that changes financial state must use a database
transaction where applicable.

External provider calls must use durable workflow/state handling because
external APIs cannot participate directly in the application's database
transaction.

## 4.5 Idempotency is mandatory

Repeated requests, webhook deliveries, scheduled jobs, or retry attempts
must not duplicate financial effects.

------------------------------------------------------------------------

# 5. Core Domain Model

The major domain entities are:

-   User
-   Partner
-   Admin User
-   Service Category
-   Listing
-   Listing Image
-   Pricing Rule
-   Availability
-   Booking
-   Payment
-   Payment Event
-   Commission Rate
-   Commission Record
-   Partner Ledger Entry
-   Payout Event
-   Withdrawal
-   Dispute
-   Refund
-   Notification
-   Notification Attempt
-   Audit Log
-   Admin Alert
-   Chat Conversation
-   Chat Message
-   Wishlist
-   Subscription

The final database specification will define exact columns, constraints,
indexes, and relationships.

------------------------------------------------------------------------

# 6. Listing Lifecycle

The Partner listing workflow is:

`Draft → Pending Review → Approved/Active`

Alternative path:

`Pending Review → Changes Requested → Pending Review`

A listing must not appear publicly as an approved marketplace listing
before Admin approval.

A Partner may create and edit listings according to the permissions
defined by the product.

Admin actions include:

-   Approve
-   Reject / request changes
-   Suspend
-   Reactivate

The system must preserve listing status independently from Partner
account status.

------------------------------------------------------------------------

# 7. Partner Lifecycle

The Partner onboarding flow is:

`Registered → Pending Approval → Approved`

A Partner account may be suspended by Admin.

Suspension must affect the appropriate operational permissions without
silently deleting the Partner's historical bookings or financial
records.

Existing financial records must remain auditable.

------------------------------------------------------------------------

# 8. Booking Lifecycle

The primary booking state machine is:

``` text
PENDING_PAYMENT
      |
      v
UPCOMING
      |
      v
ACTIVE
      |
      +------> DISPUTED
      |
      v
COMPLETED
```

Alternative paths include:

``` text
UPCOMING → CANCELLED
ACTIVE → DISPUTED
DISPUTED → COMPLETED
DISPUTED → REFUNDED
```

The exact product-defined booking states remain those established by the
PRD.

## 8.1 PENDING_PAYMENT

Booking has been initiated but payment has not yet been authoritatively
confirmed.

The booking must not be treated as a paid confirmed booking solely
because the frontend receives a successful payment-page response.

## 8.2 UPCOMING

Payment has been successfully confirmed and the service has not yet
started.

## 8.3 ACTIVE

The service has started.

The protection/release window is now relevant.

## 8.4 COMPLETED

The booking has reached its successful completion/release state.

A completed booking must not be released again.

## 8.5 DISPUTED

A Client dispute has been raised.

Automatic release must be frozen while an active dispute exists.

## 8.6 CANCELLED

Booking has been cancelled according to applicable cancellation rules.

## 8.7 REFUNDED

Financial resolution has resulted in a refund.

------------------------------------------------------------------------

# 9. Booking Invariants

The following rules are mandatory:

1.  A booking cannot be confirmed without verified payment.
2.  A booking cannot be paid twice.
3.  A completed booking cannot be released twice.
4.  An active dispute prevents automatic release.
5.  Booking availability must be revalidated server-side.
6.  Frontend availability results are advisory only.
7.  Booking creation must protect against concurrent double booking.
8.  Historical booking records must not be deleted merely to correct an
    operational mistake.
9.  Financial corrections must be represented by correcting
    transactions/events.

------------------------------------------------------------------------

# 10. Availability

Availability is a first-class domain concern.

The system must support the relevant availability model for the listing
category, including date-based accommodation and time/slot-based
experiences or rentals where applicable.

The availability process is:

``` text
Client selects dates/time
        ↓
Server checks availability
        ↓
Inventory/slot is temporarily protected
        ↓
Payment process
        ↓
Verified payment
        ↓
Booking confirmed
```

Availability must be checked again on the server immediately before
final confirmation.

Concurrent requests must not allow the same inventory to be successfully
confirmed twice.

------------------------------------------------------------------------

# 11. Pricing

The backend is responsible for calculating authoritative booking prices.

For accommodation, the PRD defines pricing categories including:

-   Standard
-   Weekend
-   Public Holiday
-   December/January
-   Custom Period

The pricing priority is:

`Custom Period → Holiday Season → Public Holiday → Weekend → Standard`

The frontend may display calculated pricing but must not be trusted to
submit an authoritative total.

The backend must calculate or verify:

-   gross booking value
-   applicable pricing rule
-   applicable commission
-   Partner net entitlement

------------------------------------------------------------------------

# 12. Payment Architecture

Paystack is the payment provider for:

-   Client payment collection
-   payment verification/webhooks
-   Partner transfers
-   applicable refunds
-   subscription billing where required

The application remains responsible for its internal business state.

The payment flow is:

``` text
Client
  ↓
Create/prepare booking
  ↓
Initialize Paystack transaction
  ↓
Client completes payment
  ↓
Paystack webhook
  ↓
Verify webhook signature
  ↓
Verify transaction/event
  ↓
Idempotency check
  ↓
Update Payment
  ↓
Confirm Booking
  ↓
Create CommissionRecord
  ↓
Create Partner pending entitlement
```

------------------------------------------------------------------------

# 13. Paystack Webhook Rules

The Paystack webhook endpoint must:

1.  Receive the provider event.
2.  Validate the webhook signature.
3.  Identify the provider event/transaction.
4.  Check whether the event has already been processed.
5.  Verify transaction details where required.
6.  Perform the relevant state transition.
7.  Record the event as processed.
8.  Return an appropriate response.

The same webhook must never create duplicate:

-   payments
-   bookings
-   commission records
-   settlement records
-   withdrawals
-   refunds

------------------------------------------------------------------------

# 14. Payment State

The internal Payment record should distinguish at minimum:

-   initiated
-   pending
-   successful
-   failed
-   reversed/refunded where applicable

The exact provider status must not be exposed as the sole application
business state.

A payment provider event is an external fact.

The application maps that fact into its own domain state.

------------------------------------------------------------------------

# 15. Commission Engine

The commission rate is determined from the applicable booking
category/rules.

The PRD requires the commission rate to be frozen at booking
confirmation.

Therefore the system must create an immutable CommissionRecord
containing the values needed to explain the booking's financial
calculation.

Conceptually:

``` text
gross_booking_amount
commission_rate
commission_amount
partner_net_payout
booking_id
created_at
```

Formula:

``` text
commission_amount =
gross_booking_amount × commission_rate
```

``` text
partner_net_payout =
gross_booking_amount - commission_amount
```

Once created, the CommissionRecord must not change because an Admin
later changes the category's commission rate.

------------------------------------------------------------------------

# 16. Partner Financial Model

Partner money must be separated into financial states.

At minimum:

``` text
PENDING EARNINGS
AVAILABLE EARNINGS
WITHDRAWAL RESERVED
```

Only Available Earnings can be withdrawn.

A booking payment does not immediately make the Partner's amount
withdrawable.

------------------------------------------------------------------------

# 17. Settlement Flow

The standard settlement model is:

``` text
Client Payment
      ↓
Verified Payment
      ↓
Commission Calculation
      ↓
Partner Net Entitlement
      ↓
PENDING
      ↓
Release Condition
      ↓
AVAILABLE
      ↓
Withdrawal
```

The release condition is determined by the booking lifecycle and the
applicable protection window.

------------------------------------------------------------------------

# 18. Auto-Release

The PRD defines an automated release scheduler running every 15 minutes.

The scheduler identifies eligible bookings.

Conceptually:

``` text
booking.status = ACTIVE
AND auto_release_at <= current_time
AND no active dispute
AND not already released
```

For an eligible booking, the settlement operation must:

1.  Verify booking is still eligible.
2.  Verify no active dispute.
3.  Verify release has not already happened.
4.  Mark the booking completed.
5.  Create the appropriate PayoutEvent/settlement record.
6.  Move the Partner's entitlement from pending to available.
7.  Record the completion/release timestamp.
8.  Commit the changes atomically.

If the operation fails, the database transaction must roll back.

The job must be safe to retry.

------------------------------------------------------------------------

# 19. Category Release Windows

The PRD defines category-specific protection windows.

Current product values include:

  Category                  Release Window
  ----------------------- ----------------
  Spa                             24 hours
  Gym                             24 hours
  Tennis                          24 hours
  Netball                         24 hours
  Paddle Court                    24 hours
  Swimming                        24 hours
  Wine Tasting                    24 hours
  Hotel                           24 hours
  Short-Stay Apartment            24 hours
  Beach-House Apartment           24 hours
  Event Hall                      48 hours
  Boat Cruise                     48 hours
  House Party                     48 hours
  Travel Package                  72 hours
  Car Rental                      24 hours
  Private Jet Charter             24 hours

The exact release datetime should be calculated and stored against the
booking when the booking reaches the relevant state.

The system must not retroactively change an existing booking's release
time merely because an Admin later changes a category configuration.

------------------------------------------------------------------------

# 20. Confirm Experience

The Client may use Confirm Experience as an early release mechanism
where allowed.

The system must:

1.  Verify the authenticated Client owns the booking.
2.  Verify the booking is eligible for confirmation.
3.  Verify the booking has not already been released.
4.  Verify no active dispute exists.
5.  Execute the settlement/release transaction.
6.  Mark the booking appropriately.
7.  Make the Partner entitlement available.
8.  Prevent repeated release.

Confirm Experience is not permission for the Client to modify arbitrary
financial amounts.

------------------------------------------------------------------------

# 21. Partner Ledger

Partner balance must be represented by an auditable ledger rather than
being inferred only from current bookings.

Conceptual ledger fields:

``` text
id
partner_id
booking_id
withdrawal_id
transaction_type
direction
amount
reference
description
created_at
```

Examples of transaction types include:

-   booking entitlement
-   release
-   withdrawal reservation
-   withdrawal completion
-   withdrawal reversal
-   refund adjustment
-   dispute resolution adjustment

The exact accounting model will be finalized in the Database
Specification.

------------------------------------------------------------------------

# 22. Financial Invariants

These are non-negotiable.

1.  A Partner cannot withdraw pending earnings.
2.  A Partner cannot withdraw more than available earnings.
3.  A release operation cannot execute twice.
4.  A withdrawal reservation cannot reserve the same funds twice.
5.  A failed withdrawal must return reserved funds exactly once.
6.  A duplicate webhook cannot create duplicate money.
7.  A commission rate change cannot change historical CommissionRecords.
8.  A disputed booking cannot auto-release.
9.  Financial corrections must create traceable adjustment entries.
10. Financial history must remain auditable.

------------------------------------------------------------------------

# 23. Withdrawal Lifecycle

The Partner withdrawal process is:

``` text
REQUESTED
   ↓
ADMIN REVIEW
   ↓
APPROVED
   ↓
PAYSTACK PROCESSING
   ↓
COMPLETED
```

Alternative paths include:

``` text
ADMIN REVIEW → HELD
ADMIN REVIEW → REJECTED
PAYSTACK PROCESSING → FAILED
FAILED → FUNDS RETURNED
```

The exact Admin workflow must follow the Admin PRD.

------------------------------------------------------------------------

# 24. Withdrawal Safety

When a withdrawal is created, the requested amount must be reserved
against available Partner funds.

Example:

``` text
Available = ₦100,000

Withdrawal requested = ₦100,000

Available withdrawable amount must no longer
allow another ₦100,000 withdrawal.
```

The reservation must be atomic.

If Paystack transfer fails, the system must restore the reserved amount
exactly once.

------------------------------------------------------------------------

# 25. Dispute Lifecycle

The dispute flow is:

``` text
Client raises dispute
        ↓
Booking = DISPUTED
        ↓
Auto-release frozen
        ↓
Partner notified
        ↓
Admin investigates
        ↓
Admin resolves
```

The Admin PRD defines three resolution paths:

### A. Full Partner Release

Partner receives the applicable net entitlement.

### B. Partial Partner Release + Client Refund

A defined portion is released to the Partner and the defined portion is
refunded to the Client.

### C. Full Client Refund

The Client receives the applicable refund and the Partner receives no
release for the disputed amount.

Every resolution must be auditable.

------------------------------------------------------------------------

# 26. Dispute Financial Safety

A dispute resolution must validate all amounts.

For a partial resolution:

``` text
client_refund + partner_release
must equal the applicable resolved booking amount
```

The system must reject:

-   negative amounts
-   amounts greater than the booking amount
-   unresolved amount mismatches
-   duplicate resolution attempts

External Paystack operations must be represented as durable workflow
steps.

The system must not pretend that two independent Paystack calls are one
database transaction.

------------------------------------------------------------------------

# 27. Cancellation and Refunds

Cancellation behavior must follow the product rules applicable to the
booking/listing/category.

The financial workflow must distinguish:

-   cancellation requested
-   cancellation approved/accepted where applicable
-   refund initiated
-   refund confirmed
-   refund failed

A refund must not silently alter the original payment record.

The refund should be represented as its own financial event/record
linked to the original payment/booking.

------------------------------------------------------------------------

# 28. Notification Architecture

Notifications are asynchronous operational events. A domain service records a
durable, idempotent business-event/outbox fact in its own transaction; a
separate delivery worker selects the template and channel only after commit.
Controllers and financial/domain services must not call Meta or an email
provider directly.

The current target schema is delivered by the ordered versioned migration
chain `001` through `016`. Phase 9's `014_phase9_notification_delivery.sql`
adds the durable delivery model and versioned template registry;
`015_phase9_booking_cancellation_template.sql` adds the explicit
`booking_cancelled` email and WhatsApp template rows without rewriting an
applied migration. Phase 10's additive
`016_phase10_admin_operations.sql` adds durable operational-alert facts; it
does not create a financial mutation path.

The current Phase 9 integration emits the following stable lower-case event
types where the source transaction has committed the corresponding fact:

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

The Notification Service handles delivery. The same semantic event must use a
stable event key that includes the intended recipient where an event has more
than one audience. A replay therefore returns the existing notification rather
than creating another operational message. Provider attempts, delivery
receipts, retries, and manual Admin resend are delivery history, not new
business events.

Business operations must not fail merely because a notification provider
is temporarily unavailable. Notification failure, fallback, delivery receipt,
or retry may not reverse, delay, or otherwise decide booking, payment,
settlement, dispute, refund, or withdrawal state.

The event record contains a recipient-scoped deterministic key. It is a
target-schema outbox record, not a direct provider call. A recipient without a
configured delivery contact remains a `suppressed` operational fact visible to
authorized Admin operations, rather than causing the originating transaction
to fail.

------------------------------------------------------------------------

# 29. WhatsApp Primary + Email Fallback

The operational communication requirement is:

``` text
WhatsApp
   ↓
If successful → complete
   ↓
If failed
   ↓
Email fallback
```

Meta Business API/WhatsApp must be isolated behind a notification provider
interface. WhatsApp is attempted first. A successful/accepted WhatsApp
attempt does not also send email by default. Email is created only after the
bounded WhatsApp retry policy reaches final failure or WhatsApp cannot be
safely initiated. Both channels keep separate append-only attempts linked to
the same notification.

Application template key/version, channel, language, and provider template
identifier are registry data. Controllers and domain services do not embed
provider template strings. A live Meta dispatch requires a configured eligible
provider template identifier, apart from the separately configured approved
OTP template. Missing configuration/template eligibility is a delivery failure
handled by the worker and never a business-transaction failure.

When a domain event writes an outbox fact, it persists the active application
template version. The delivery worker and any eligible Admin resend resolve
that exact stored version rather than silently rendering a newer template
version. This preserves the delivery fact's template/parameter contract across
later template-registry changes.

The notification system should record:

``` text
notification_id
event_type
recipient
channel
status
provider_message_id
attempt_count
error
sent_at
delivered_at
```

A failed WhatsApp delivery must not roll back a successful booking or
financial operation.

Email is fallback communication, not a replacement for business-state
persistence.

The notification projection states are `queued`, `processing`, `sent`,
`delivered`, `read`, `fallback_email_sent`, `failed`, and `suppressed`.
Attempt states are `queued`, `sending`, `sent`, `delivered`, `read`, and
`failed`. Provider delivery receipts can only advance compatible state;
duplicate, out-of-order, unknown, malformed, or unauthenticated receipts are
harmless.

The worker uses a bounded retry count and configured delay. It appends a new
attempt for each delivery try, records only safe failure codes/metadata, and
creates at most one email fallback path for a notification after terminal
WhatsApp failure. A scheduled reminder pass creates recipient-scoped,
idempotent `booking.reminder` and `travel.balance_due` facts; repeated runs do
not create duplicate reminders.

For OTP delivery, the verification record remains hash-only. Any plaintext OTP
needed by the renderer is encrypted separately as short-lived private
notification context and is never stored in ordinary payloads, attempts,
webhook facts, audits, logs, or API responses.
The worker clears that encrypted context after provider acceptance, successful
email fallback, or terminal failure. Its scheduled run also purges any
remaining `auth_otp` private context once the linked OTP verification has
expired, without changing the OTP verification record or its attempt limits.
An OTP notification is deliverable only while its linked verification record
is unverified and unexpired. Verification atomically suppresses queued or
processing OTP delivery and clears its private context; a delayed worker cannot
make a later OTP valid or send an already verified/expired one. Admin resend is
not permitted for OTP notifications.

Meta callback verification uses the configured verify token for the `GET`
subscription handshake and HMAC-SHA-256 of the raw `POST` body using the app
secret. Webhook evidence is sanitized and deduplicated before it can advance a
matching existing WhatsApp attempt; it cannot create financial or booking
state.

------------------------------------------------------------------------

# 30. Notification Idempotency

The same business event must not cause uncontrolled duplicate
notifications.

A notification should have a deterministic event/reference key where
appropriate.

Retries should update the notification attempt history rather than
create unrelated duplicate business events.

------------------------------------------------------------------------

# 31. Admin Operations

The Admin Dashboard is an operational command centre.

Core operational areas include:

-   Dashboard/Command Centre
-   Vendor/Partner management
-   Listing approval
-   Booking management
-   Dispute management
-   Withdrawal management
-   Commission management
-   Financial reporting
-   System health
-   WhatsApp template/notification management
-   Audit trail

Admin APIs must enforce permissions server-side.

------------------------------------------------------------------------

# 32. Audit Logging

Sensitive administrative and financial actions must generate append-only
audit records.

Audit information should include:

``` text
actor
action
entity_type
entity_id
previous_state/value where appropriate
new_state/value where appropriate
reason where required
timestamp
request/context metadata where appropriate
```

Examples:

-   Partner approved
-   Listing approved
-   Listing rejected
-   Partner suspended
-   Commission changed
-   Withdrawal approved
-   Withdrawal held
-   Withdrawal rejected
-   Dispute resolved
-   Refund initiated
-   Financial adjustment made

Audit history must not be silently overwritten.

------------------------------------------------------------------------

# 33. System Health

The Admin system must expose operational failures including:

-   failed Paystack webhooks
-   unmatched Paystack records
-   failed auto-release events
-   overdue scheduler jobs
-   failed WhatsApp deliveries
-   failed email fallback
-   failed withdrawals
-   failed notification retries

Operational alerts must be distinguishable from normal business records.

------------------------------------------------------------------------

# 34. Authentication and Authorization

The backend must enforce:

-   authentication
-   role checks
-   resource ownership
-   permission checks
-   account status

Examples:

A Client may confirm/dispute only their own eligible booking.

A Partner may access only their own listings, bookings, earnings, and
withdrawals.

A Customer Support user must not automatically inherit Full Admin
financial permissions.

The frontend may hide unauthorized controls for UX, but backend
authorization remains authoritative.

------------------------------------------------------------------------

# 35. API Versioning

All public application APIs should use a versioned namespace.

Recommended:

`/api/v1/...`

Breaking API changes require a versioning strategy rather than silently
changing an existing contract.

The API documentation will become the contract consumed by:

-   React web
-   React Native/Expo mobile
-   Admin frontend
-   future integrations where applicable

------------------------------------------------------------------------

# 36. API Response Principles

APIs should return predictable structures.

Recommended successful structure:

``` json
{
  "success": true,
  "data": {}
}
```

Recommended error structure:

``` json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message",
    "details": {}
  }
}
```

The exact contract will be finalized in the API Documentation.

------------------------------------------------------------------------

# 37. Scheduled Jobs

The system requires scheduled/background processing for applicable
operations.

Known scheduled responsibilities include:

-   auto-release
-   booking reminders
-   notification retries
-   financial reconciliation
-   stale payment/booking cleanup
-   system health checks

Every scheduled job must be:

-   retry-safe
-   observable
-   logged
-   idempotent where applicable

------------------------------------------------------------------------

# 38. Source of Truth

The following sources of truth apply:

  Domain                  Source of Truth
  ----------------------- ------------------------------------------------------
  Authentication          Backend
  Authorization           Backend
  Listing status          Backend database
  Availability            Backend database
  Booking status          Backend database
  Payment confirmation    Verified Paystack event mapped into backend
  Commission              Immutable CommissionRecord
  Partner earnings        Financial ledger/settlement records
  Withdrawal status       Backend + verified Paystack transfer state
  Dispute state           Backend database
  Notification delivery   Notification records/provider events
  UI display              API response, never independent business calculation

------------------------------------------------------------------------

# 39. External Integration Boundaries

## Paystack

Responsible for provider-side payment/transfer operations.

Application responsible for:

-   mapping provider events
-   internal state
-   idempotency
-   reconciliation
-   financial records

## Meta Business / WhatsApp

Responsible for operational messaging delivery.

Application responsible for:

-   notification event
-   template selection
-   recipient
-   delivery tracking
-   retries
-   fallback

## Email

Fallback notification channel.

Email delivery failure must be recorded and surfaced operationally where
appropriate.

------------------------------------------------------------------------

# 40. Data Integrity Rules

The system must use:

-   foreign keys where appropriate
-   unique constraints
-   appropriate indexes
-   database transactions
-   server-side validation
-   immutable financial records
-   state transition validation

The database must not allow obvious invalid states where a constraint
can prevent them.

------------------------------------------------------------------------

# 41. Error Handling Principles

Errors must be categorized.

### Client errors

Examples:

-   invalid request
-   unauthorized
-   forbidden
-   resource not found
-   availability conflict
-   validation failure

### Provider errors

Examples:

-   Paystack timeout
-   WhatsApp provider failure
-   email provider failure

### Internal errors

Examples:

-   database failure
-   unexpected exception
-   scheduler failure

Provider failure must not automatically be treated as business failure.

For example:

``` text
WhatsApp failed
```

does not mean:

``` text
Booking failed
```

------------------------------------------------------------------------

# 42. Concurrency Rules

The system must specifically protect:

-   inventory/availability
-   booking creation
-   payment confirmation
-   release
-   Partner withdrawals
-   dispute resolution

The implementation must prevent race conditions such as:

``` text
Two clients booking the same room simultaneously
```

or:

``` text
Two withdrawal requests spending the same balance
```

or:

``` text
Auto-release and Confirm Experience running simultaneously
```

------------------------------------------------------------------------

# 43. Release Concurrency

Auto-release and manual Confirm Experience must use the same release
service or equivalent shared transactional logic.

Do not implement two independent payout paths.

Both must ultimately call the same protected settlement mechanism.

This ensures:

``` text
Auto-release + Confirm Experience
```

cannot release the same entitlement twice.

------------------------------------------------------------------------

# 44. Financial Correction Principle

If a financial mistake occurs, do not modify historical financial
records simply to make a displayed balance look correct.

Instead:

``` text
Original transaction
       ↓
Correction/Adjustment transaction
       ↓
Updated balance
```

This preserves auditability.

------------------------------------------------------------------------

# 45. Testing Requirements

Every major domain must have tests.

## Authentication

-   valid login
-   invalid login
-   unauthorized access
-   role restrictions

## Listings

-   Partner creates listing
-   validation
-   submission
-   Admin approval
-   rejection/changes requested
-   suspension

## Availability

-   available dates
-   unavailable dates
-   concurrent booking
-   expired temporary hold

## Payments

-   successful Paystack payment
-   failed payment
-   duplicate webhook
-   delayed webhook
-   invalid webhook signature

## Booking

-   state transitions
-   invalid transitions
-   cancellation
-   dispute
-   completion

## Settlement

-   commission calculation
-   frozen commission
-   pending balance
-   auto-release
-   manual release
-   duplicate release

## Withdrawals

-   insufficient funds
-   reservation
-   Admin approval
-   transfer success
-   transfer failure
-   reversal
-   duplicate request

## Disputes

-   raise dispute
-   freeze release
-   full release
-   partial resolution
-   full refund
-   duplicate resolution

## Notifications

-   WhatsApp success
-   WhatsApp failure
-   email fallback
-   retry
-   duplicate notification prevention

------------------------------------------------------------------------

# 46. Master Financial Test

The minimum financial scenario is:

``` text
Booking value = ₦100,000
Commission = 10%
Partner net entitlement = ₦90,000
```

After verified payment:

``` text
Partner pending = ₦90,000
Partner available = ₦0
```

After release:

``` text
Partner pending = ₦0
Partner available = ₦90,000
```

After a successful ₦90,000 withdrawal:

``` text
Partner available = ₦0
Withdrawal = COMPLETED
```

Repeating any release or webhook must not increase the balance.

------------------------------------------------------------------------

# 47. Master End-to-End Test

A complete successful journey must be possible:

``` text
Partner registers
    ↓
Admin approves Partner
    ↓
Partner creates listing
    ↓
Admin approves listing
    ↓
Client browses listing
    ↓
Client selects availability
    ↓
Client books
    ↓
Client pays through Paystack
    ↓
Paystack webhook verified
    ↓
Booking confirmed
    ↓
CommissionRecord created
    ↓
Partner entitlement enters pending state
    ↓
Service starts
    ↓
Booking becomes Active
    ↓
Client confirms OR auto-release occurs
    ↓
Booking becomes Completed
    ↓
Partner funds become Available
    ↓
Partner requests withdrawal
    ↓
Funds reserved
    ↓
Admin approves where required
    ↓
Paystack transfer
    ↓
Transfer confirmed
    ↓
Withdrawal Completed
```

------------------------------------------------------------------------

# 48. Master Dispute Test

``` text
Client pays
    ↓
Booking confirmed
    ↓
Partner entitlement pending
    ↓
Service becomes Active
    ↓
Client raises dispute
    ↓
Booking = DISPUTED
    ↓
Auto-release blocked
    ↓
Admin investigates
    ↓
Admin resolves
```

The test suite must verify each supported resolution path.

------------------------------------------------------------------------

# 49. Development Phase Gates

Development must proceed sequentially.

## Phase 1

Architecture + Business Rules

**Gate:** State machines, financial rules, permissions, integration
boundaries and edge cases are defined.

## Phase 2

Database Schema

**Gate:** Schema migrations, constraints, reference data, and indexes are
verified.

## Phase 3

Authentication + Roles

**Gate:** Authentication and authorization tests pass.

## Phase 4

Partner Marketplace

**Gate:** Partner listing lifecycle works end-to-end.

## Phase 5

Booking Engine

**Gate:** Availability and booking concurrency tests pass.

## Phase 6

Paystack Payment Integration

**Gate:** Verified payment/webhook flow works.

## Phase 7

Settlement / Wallet / Ledger

**Gate:** Financial invariants and duplicate-event tests pass.

## Phase 8

Partner Withdrawals

**Gate:** Withdrawal reservation and Paystack transfer lifecycle pass.

## Phase 9

WhatsApp Notification Service

**Gate:** WhatsApp + email fallback works independently of core
transactions.

## Phase 10

Admin Operations

**Gate:** Operational workflows and permissions pass.

## Phase 11

Refunds / Cancellations / Edge Cases

**Gate:** Critical exception paths pass.

## Phase 11 implemented protection boundary

Phase 11 uses one cancellation/refund service, one existing settlement service,
and the existing isolated Paystack client. Cancellation is policy-driven and
effective-dated; absent commercial policy resolves to Admin review rather than
inventing a percentage. Quotes freeze the selected policy calculation for a
short validity period. Every refund starts as a durable intent, has a verified
provider lifecycle, and is reconciled through signed webhook/provider facts.

Refund corrections reserve recoverable Partner funds before provider terminal
success. Shortfall creates a recovery obligation rather than a negative
wallet. Active disputes hold the booking and fail-close all release paths.
Full Admin may resolve only through a server-calculated retained entitlement,
the Refund Service, or the shared Settlement Service. Late successful payment
after expiry remains cancelled and enters refund reconciliation; it never
reopens inventory. Client, Partner and Admin use the same canonical `/api/v1`
surface with role/ownership controls and append-only audit/notification facts.

## Phase 12

Testing

**Gate:** Full regression passes.

## Phase 13

Production Hardening

**Gate:** Security, observability, backup, reconciliation and deployment
checks pass.

------------------------------------------------------------------------

# 50. AI Coding Agent Rules

Codex or another coding agent must follow these rules.

1.  Read all project specification documents before implementation.
2.  Do not invent business rules.
3.  Do not change financial rules without explicit approval.
4.  Do not treat frontend state as authoritative.
5.  Do not trust frontend financial amounts.
6.  Do not bypass API authorization.
7.  Do not create duplicate financial effects.
8.  Do not skip webhook verification.
9.  Do not skip idempotency.
10. Do not silently modify historical financial records.
11. Do not refactor unrelated functionality while implementing a phase.
12. Do not mark a phase complete because the application compiles.
13. A phase is complete only when its acceptance tests pass.
14. Stop when an unresolved requirement materially affects
    implementation.
15. Report unresolved assumptions instead of silently choosing product
    behavior.

------------------------------------------------------------------------

# 51. Codex Task Method

For every implementation task:

## Step 1 --- Inspect

Inspect the relevant:

-   source code
-   database
-   routes
-   controllers
-   services
-   models
-   middleware
-   migrations
-   environment configuration
-   tests

## Step 2 --- Plan

State:

-   files to modify
-   database changes
-   APIs affected
-   business rules affected
-   tests required

## Step 3 --- Implement

Make the smallest coherent implementation.

## Step 4 --- Test

Run relevant:

-   lint
-   type checks
-   unit tests
-   integration tests
-   end-to-end tests

## Step 5 --- Verify

Compare implementation against the specification.

## Step 6 --- Report

Return:

-   implemented changes
-   files changed
-   migrations
-   API changes
-   tests passed
-   tests failed
-   unresolved issues
-   next phase

------------------------------------------------------------------------

# 52. Existing React Application

The existing React application is an existing UI/UX asset.

It should be treated as:

-   visual reference
-   UX reference
-   component/design reference
-   existing frontend implementation to be assessed

It must not automatically be treated as the authority for:

-   backend architecture
-   financial logic
-   booking state
-   database design
-   API behavior
-   authorization

Before reuse, the existing implementation should be classified into:

1.  Keep
2.  Refactor
3.  Rebuild
4.  Reference only

The new backend architecture must not be distorted merely to preserve
flawed assumptions in the existing frontend.

------------------------------------------------------------------------

# 53. Mobile Application

The mobile application must consume the same backend APIs as the web
application.

The mobile app must not contain duplicated business logic that can
produce a different financial or booking result from the web
application.

Examples of backend-owned logic:

-   final booking price
-   availability
-   commission
-   Partner balance
-   withdrawal eligibility
-   release eligibility
-   dispute eligibility

The mobile application should primarily:

-   collect user input
-   call APIs
-   display server state
-   manage client-side UX state

------------------------------------------------------------------------

# 54. API Documentation Requirement

A dedicated API contract must be maintained as a first-class project
artifact.

It must document every endpoint including:

-   HTTP method
-   URL
-   authentication
-   required role
-   path parameters
-   query parameters
-   request body
-   validation
-   success response
-   error response
-   HTTP status codes
-   pagination
-   filtering
-   sorting
-   idempotency requirements
-   webhook behavior
-   examples

Recommended structure:

``` text
/docs/api/
    README.md
    authentication.md
    users.md
    partners.md
    listings.md
    availability.md
    bookings.md
    payments.md
    wallet.md
    withdrawals.md
    disputes.md
    refunds.md
    notifications.md
    admin.md
    webhooks.md
```

The final API contract should be machine-readable through
OpenAPI/Swagger where practical.

------------------------------------------------------------------------

# 55. Database Specification Requirement

The database specification must be derived from:

1.  Product requirements
2.  Engineering business rules
3.  API contract

It must document:

-   tables
-   columns
-   types
-   relationships
-   indexes
-   unique constraints
-   foreign keys
-   enum/state values
-   immutable fields
-   financial records
-   audit records

Database design must not be generated independently of the business
state machines.

------------------------------------------------------------------------

# 56. Final Engineering Principle

The application must be built in this direction:

``` text
PRODUCT REQUIREMENTS
        ↓
ENGINEERING BUSINESS RULES
        ↓
STATE MACHINES
        ↓
API CONTRACT
        ↓
DATABASE SPECIFICATION
        ↓
BACKEND IMPLEMENTATION
        ↓
WEB / MOBILE / ADMIN CLIENTS
        ↓
TESTING
        ↓
PRODUCTION HARDENING
```

Not:

``` text
UI
 ↓
Random API
 ↓
Random database
 ↓
Feature patches
 ↓
Financial fixes
```

The objective is to make the business rules explicit before the coding
agent implements them.

The system should be considered successful only when the same business
rules produce the same result regardless of whether the request
originates from Web, Mobile, Admin, a scheduler, or an external provider
webhook.

------------------------------------------------------------------------

# 57. Next Engineering Artifact

After this document is approved, the next document to create is:

`HANDYPROS_API_DOCUMENTATION.md`

That document will define the complete `/api/v1` contract
endpoint-by-endpoint.

After the API contract is approved, create:

`HANDYPROS_DATABASE_SPECIFICATION.md`

Only after those three engineering artifacts are established should the
fresh Codex implementation project begin.
