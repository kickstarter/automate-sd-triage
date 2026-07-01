[← Index](README.md) | [Next: CHECK-304 →](check-304.md)

# CHECK-306 — PLOT installment errored, 2 successful Stripe charges

**Corrected 2026-07-01** — the original classification on this page (duplicate charge) was wrong. See
[the Jira thread](https://kickstarter.atlassian.net/browse/CHECK-306?focusedCommentId=152364) for the
EM's correction. Left the original reasoning below the corrected version rather than deleting it, since
that's the record of what went wrong and why.

| Field | Value |
|---|---|
| Priority | High |
| Classification | State drift (was: duplicate charge from 3DS retry — corrected) |
| Confidence | High (per EM correction) |
| System | rosie |
| Pledge | https://www.kickstarter.com/admin/pledges/194067334 |
| ids | kickstarter backing id `194067334` → rosie pledge via `Pledge.find_by(client_id: 1, foreign_key: 194067334)` |

## Why this classification (corrected)

The 2nd installment was collected in Stripe but incorrectly marked `errored` locally — that's state
drift, the `SyncIncrementalPledgeToStripe` class of fix, same as [CHECK-303](check-303.md). The
original read below over-weighted Rovo's "backer appears double-charged" auto-triage framing and the
ticket's "2 successful charges" phrasing into a duplicate-charge classification, without confirming
which Stripe PI actually links to which increment. Lesson: weigh the ticket description over comments
when they disagree, and flag ambiguity explicitly rather than picking the more dramatic-sounding
classification on a superficial match.

**Residual ambiguity:** two successful Stripe PIs are on record for this pledge the same day. It isn't
established from the ticket text alone which one backs the 2nd installment specifically —
`SyncIncrementalPledgeToStripe` resolves that itself (it looks up each increment's linked PI via its
own `funds_capture`/collection record, not by amount/date matching), so the dry-run report shows it
directly, per increment.

## Pre-flight

Confirm in the Stripe dashboard right now that both PIs below still show `succeeded`:

- https://dashboard.stripe.com/acct_104ATn4VvJ2PtfhK/payments/pi_3TliDU4VvJ2PtfhK0mJnsbqI
- https://dashboard.stripe.com/acct_104ATn4VvJ2PtfhK/payments/pi_3TliDX4VvJ2PtfhK1GP5gvOH

## Remediation

```ruby
require_relative 'lib/support_tasks/sync_incremental_pledge_to_stripe'
SupportTasks::SyncIncrementalPledgeToStripe.perform(pledge_id: <rosie pledge.id>)  # dry-run, then dry_run: false
```

Read the dry-run report before going live: confirm only the 2nd installment's
`PaymentIncrement`/`FundsCapture` transitions, and sanity-check that the *other* successful PI belongs
to a different, already-correctly-accounted-for increment rather than being a second attempt against
the same one.

## Code link

[rosie/lib/support_tasks/sync_incremental_pledge_to_stripe.rb@demo/stripe-sync-plan-apply](https://github.com/kickstarter/rosie/blob/demo/stripe-sync-plan-apply/lib/support_tasks/sync_incremental_pledge_to_stripe.rb)

## Verify-after

The 2nd installment's `PaymentIncrement`/`FundsCapture` flip to `collected`; confirm the other
successful PI's increment is unaffected (or already correctly `collected`) — not itself now
double-counted.

## Citation

Task docstring; no dedicated Guru card yet for this specific task.

---

## Original (incorrect) classification — kept for the record

Matches `SupportTasks::RefundDuplicateCharge`'s documented bug almost verbatim: a backer retries fixing
a card that needs 3DS authentication after the project has succeeded, and ends up with two successful
charges for the same installment.

This task has **no `dry_run` flag** — `.perform` refunds immediately. Manual read-only preview first,
replicating the task's internal selection logic without refunding:

```ruby
pledge = Pledge.find_by!(foreign_key: 194067334, client_id: 1)  # kickstarter backing id -> rosie pledge
pledge.payment_intents.where.not(succeeded_at: nil).map { |pi|
  { id: pi.id, stripe_pi: pi.stripe_payment_intent_id, intendable_id: pi.intendable_id, succeeded_at: pi.succeeded_at }
}
```

Live (production rosie console):

```ruby
require_relative 'lib/support_tasks/refund_duplicate_charge'
SupportTasks::RefundDuplicateCharge.perform(ksr_backing_id: 194067334)  # LIVE — no dry_run available
```

[rosie/lib/support_tasks/refund_duplicate_charge.rb@main](https://github.com/kickstarter/rosie/blob/main/lib/support_tasks/refund_duplicate_charge.rb)
— this does **not** apply to this ticket; kept only because the Jira thread references it directly.

---
[← Index](README.md) | [Next: CHECK-304 →](check-304.md)
