# HandyPros Authoritative State Machines

All transitions are backend-owned, validated and recorded. “Lock” means a database transaction with the relevant row(s) locked; notifications are dispatched only after commit.

## User/account

`pending → active` on successful required phone verification; `active → suspended|blocked` by authorized Admin; `suspended|blocked → active` only by authorized Admin. Validation: actor/role, status and OTP purpose/expiry. Effects: OTP/account audit and notification records. A protected operation requires active status.

## Partner onboarding

`registered → pending_approval → active`; `pending_approval|active → suspended|blacklisted`; suspended may be reactivated by authorized Admin. Validation: required onboarding/profile/bank requirements and Admin authority. Effects: Partner audit event, notification, independent preservation of listings/bookings/financial history.

## Listing

`draft → pending_review → approved/active`; `pending_review → changes_requested → pending_review`; `approved/active → suspended`; suspended may be reactivated by Admin. Validation: Partner ownership, active Partner, category-specific required data, and Admin authority. Effects: review event/audit; only approved active listings are publicly discoverable/bookable.

## Booking

`pending_payment → upcoming` only after verified payment. `upcoming → active` when service starts. `active → completed` only through the shared settlement service; `active → disputed` only for the owning Client before release. `disputed → completed|refunded` only through Admin resolution. `pending_payment|upcoming → cancelled` under applicable cancellation rules; pending expiry is recommended to become `cancelled(payment_expired)` per D-003. Every transition writes status history.

## Payment and payment webhook

Payment: `pending → successful|failed|reversed|refunded|partially_refunded`. A successful provider fact is not enough unless signature/reference/amount/currency/booking eligibility validate. Webhook: `received → signature_verified → processing → processed|failed`; duplicate stable provider event ID returns the recorded result without another domain effect.

## Partner settlement and ledger

Verified booking payment creates `pending` partner entitlement and immutable commission record. Eligible release transitions `pending → available` once through a payout event (`auto_release`, `client_confirmed`, or `admin_resolution`). A dispute freezes release. Ledger entries are append-only events (`pending_credit`, `release_credit`, withdrawal reservation/release/debit, refund/dispute/manual adjustment); they are never state-mutated or deleted.

## Withdrawal

`pending_review → held|approved|rejected`; `held → approved|rejected`; `approved → processing`; `processing → completed|failed|reversed`. Create: lock and reconcile the Partner ledger account, validate a verified Partner-owned payout recipient and available funds, write the withdrawal plus `available` debit / `reserved` credit entries, then commit. Hold leaves the reservation unchanged. Reject, verified failure, and verified reversal each return reserved funds once through compensating ledger entries. Verified transfer success debits reserved once. Admin approval creates one durable Paystack transfer attempt; an unknown provider outcome remains `processing`/reserved until signed-webhook or reconciliation verification establishes a matching terminal provider fact. Terminal statuses cannot be transitioned again.

## Dispute and refund

Dispute: `open → under_review → resolved_partner_release|resolved_partial|resolved_refund` (persisted implementation may retain `resolved` plus outcome). Creation locks an active owned booking, confirms no released funds/active dispute and window eligibility, then marks booking disputed. Refund: `pending → processing → successful|failed`; it is its own immutable record linked to payment/booking, never a mutation of the original payment.

### Phase 11 protection refinement

```text
cancellation quote: active -> consumed|expired
refund: requested -> processing -> successful
                         |-> failed -> processing (Full-Admin retry)
                         |-> provider_uncertain -> reconciliation -> processing|successful|failed
dispute: open -> under_review -> resolved
```

Provider uncertainty is fail-closed: the same intent and Partner correction
remain reserved until matching provider facts are verified. An unresolved
dispute holds the booking; Full Admin resolution alone clears that hold and
may invoke the existing settlement engine. Travel Package cancellation marks
an unpaid future balance schedule `cancelled` without inventing a commercial
penalty.

## Notification

Notification records are durable outbox facts, created in the same database
transaction as the business event and dispatched only after commit. The
notification lifecycle is backend-owned:

`queued → processing → sent → delivered → read`

Alternative terminal paths are:

`queued|processing|sent → fallback_email_sent` when WhatsApp reaches its
final retryable/non-retryable failure and the fallback email is accepted by its
provider; and `queued|processing|sent → failed` when all permitted delivery
paths fail. `suppressed` is terminal and is allowed only when a server-side
eligibility/safety rule intentionally prevents delivery; it is not a silent
deletion of the event.

An individual attempt is append-only and moves
`queued → sending → sent → delivered|read`, or `queued|sending|sent →
failed`. Provider delivery receipts may advance `sent` to `delivered` or
`read`; duplicate, out-of-order, or lower-precedence provider statuses are
harmless. A failed WhatsApp attempt is retried only within the bounded retry
policy. After its final failure, the service creates one email fallback attempt
for the same notification. A successful WhatsApp attempt never creates a
default email duplicate.

Event creation is idempotent by the stable event key, which includes the
semantic recipient where one business event informs more than one actor. A
manual Admin resend creates a new attempt for the existing notification; it
does not create a second business event or a second financial effect. Worker
leases/locked transitions make duplicate worker execution harmless. Delivery
failure, retry, webhook processing, read acknowledgement, or resend cannot
reverse or delay booking, payment, settlement, dispute, refund, or withdrawal
state.

## WhatsApp delivery webhook

The Meta webhook handshake is read-only and succeeds only for a valid verify
token. POSTed delivery events must pass the current provider authenticity
check before a sanitized, deduplicated provider-event record is persisted.
The handler finds an attempt by provider message ID, maps only recognized
provider statuses into the attempt lifecycle, and then derives the
notification lifecycle. Unknown provider message IDs, malformed payloads, and
unauthenticated requests must not mutate notification or business state.

## Race-condition rules

- **Confirm Experience vs auto-release:** both call the same lock-protected settlement service; one creates the unique payout, the other returns the prior result/no-op.
- **Dispute vs auto-release:** both lock booking. Whichever commits first determines eligibility; an active dispute prevents release.
- **Duplicate payment webhook:** unique provider event/reference plus payment/booking lock prevents duplicate payment, commission or entitlement.
- **Concurrent withdrawals:** lock one ledger account, recompute available balance inside the transaction, reserve before return.
- **Admin double approval:** lock one withdrawal and create at most one unique transfer attempt/reference; a later command returns the already-advanced withdrawal and cannot send another transfer.
- **Transfer webhook vs reconciliation:** unique provider-event identity and the locked withdrawal/ledger transition allow either worker to finalize the matching transfer once; the other observes the terminal state without another financial entry.
- **Expiry vs payment:** both lock booking. Payment can confirm only while pending and unexpired; expiry releases inventory and records cancellation. A later provider success is exceptional/reconciliation/refund work, not booking confirmation.
