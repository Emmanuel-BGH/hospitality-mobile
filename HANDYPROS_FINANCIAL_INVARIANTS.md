# HandyPros Financial Invariants

Each invariant is mandatory in production.

| Invariant | Database enforcement | Application/locking | Required test |
|---|---|---|---|
| Money is integer minor units; no floating point | BIGINT/check conventions | Validate/calculate integers | rounding and serialization |
| One verified payment is applied once | unique provider reference/event; payment rows | lock payment/booking; idempotency key | duplicate/late webhook |
| Confirmed booking has frozen immutable commission | commission record unique per booking/payment; no update path | calculate at verified confirmation | later rate change |
| Net payout equals gross less frozen commission | constraints where practical | calculate in one service | ₦100k/10% scenario |
| Payment creates pending, not withdrawable earnings | ledger bucket and account rows | transaction creates pending credit | payment does not increase available |
| A booking cannot produce two releases | unique payout event/release reference | lock booking; shared release service | auto-release vs confirm race |
| Active dispute blocks release | dispute/booking state and indexes | lock booking and check active dispute | dispute vs release race |
| Available, pending and reserved reconcile to ledger | account per partner; append-only entries | account lock and reconciliation | reconciliation mismatch |
| Withdrawal cannot exceed available | account lock | validate/recompute inside reservation transaction | concurrent withdrawals |
| Reservation moves available to reserved once | withdrawal/ledger references unique | atomic create + balance bucket update | duplicate create request |
| Withdrawal target is a verified server-created recipient | bank account FK; recipient/fingerprint uniqueness | resolve account/create recipient after a durable pending marker | duplicate bank-account create / pending provider replay |
| Unknown transfer outcome remains reserved | durable transfer attempt/reference | no terminal ledger transition until provider reconciliation matches facts | provider timeout / unknown transfer |
| Failed/rejected/reversed transfer restores reserved funds once | unique withdrawal+entry type; unique provider event/reference | lock withdrawal/account; idempotent compensating state transition | duplicate transfer webhook / reconciliation race |
| Successful withdrawal debits reserved once | unique withdrawal+entry type; completed transfer reference unique | verified provider workflow matches reference/amount/currency/recipient | repeated result / Admin double approval |
| Refund/dispute corrections are explicit records | immutable refunds/ledger/audit | compensating entries, no rewrites | refund after payment/payout |
| Historical financial/audit records are not deleted | no destructive workflow; retention policy | append corrections only | attempted mutation/deletion |
| Provider calls do not create unrecorded money | provider references/events persisted | durable outbox/workflow after commit | provider timeout/retry |
| Operational notification delivery never determines a financial outcome | durable notification event/outbox is separate from financial records | commit booking/payment/settlement/withdrawal first; dispatch asynchronously afterwards | WhatsApp/email timeout, retry, fallback, and duplicate webhook while financial result remains unchanged |

For a Phase 8 withdrawal the only permitted ledger effects are:

```text
request:  WITHDRAWAL_AVAILABLE_DEBIT + WITHDRAWAL_RESERVED_CREDIT
success:  WITHDRAWAL_RESERVED_DEBIT_PAYOUT
return:   WITHDRAWAL_RESERVED_DEBIT_RETURN + WITHDRAWAL_AVAILABLE_CREDIT_RETURN
```

`return` applies to rejection before transfer, verified provider failure, and
verified provider reversal. Every financial transition requires database
transactions, row locks for booking/account/withdrawal as applicable,
idempotency records for commands/events, durable provider evidence, and
audit/request correlation. Tests must assert both returned result and
persisted ledger/account/booking consistency.

## Phase 11 refund and dispute corrections

Refund authorization never edits a historical CommissionRecord, payment,
settlement entry, or payout. It writes explicit append-only compensation
facts:

```text
pending/available -> REFUND_*_DEBIT -> REFUND_RESERVED_CREDIT
provider success  -> REFUND_RESERVED_DEBIT
```

An authorized refund therefore cannot remain withdrawable. A provider failure
or uncertainty leaves captured funds reserved; it never re-credits available
earnings. Where an already-settled account cannot cover the correction,
`partner_recovery_obligations` records the unrecovered amount without a
negative wallet. Refund capacity is verified payment amount less all
non-rejected refund intents. Multiple partial refunds are oldest-payment first
and settlement releases only frozen net less durable refund adjustments.

An unresolved dispute blocks settlement. Full Admin resolution invokes either
the single Refund Service or the existing Settlement Service. A verified late
payment after expiry never reopens inventory; it stays cancelled, creates a
durable refund/reconciliation obligation, and raises an alert.
