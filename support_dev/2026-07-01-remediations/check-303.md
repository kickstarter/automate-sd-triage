[← Previous: CHECK-304](check-304.md) | [Index](README.md) | [Next: CHECK-250 →](check-250.md)

# CHECK-303 — PLOT installments errored, successful in Stripe (2 backers, 2 projects)

| Field | Value |
|---|---|
| Priority | High |
| Classification | State drift |
| Confidence | High |
| System | rosie |
| Projects | https://www.kickstarter.com/admin/projects/5265630, https://www.kickstarter.com/admin/projects/5275028 |

## Why this classification

Textbook `SyncIncrementalPledgeToStripe` case: successful Stripe charges not reflected locally.
CS's own triage rationale cites CHECK-256 and CHECK-251 as prior instances of this exact symptom.

## Pledges (run independently)

1. kickstarter backing `193272761` → `Pledge.find_by(client_id: 1, foreign_key: 193272761)`
   Stripe PI: https://dashboard.stripe.com/acct_104ATn4VvJ2PtfhK/payments/pi_3TjT6O4VvJ2PtfhK0LeBp2VU
2. kickstarter backing `192099912` → `Pledge.find_by(client_id: 1, foreign_key: 192099912)`
   Stripe PI: https://dashboard.stripe.com/acct_104ATn4VvJ2PtfhK/payments/pi_3Tk2GQ4VvJ2PtfhK0un5bcgw
   **Wrinkle:** this pledge's 3rd installment was [partially refunded](https://www.kickstarter.com/refunds/115017)
   by the creator in April. The task's refund short-circuit only fires on a *full* refund
   (`refund_total >= funds_capture.capture_amount`), so this should fall through to normal state sync —
   but confirm the dry-run report doesn't show anything unexpected for that installment before going live.

## Pre-flight

Confirm both Stripe PIs above still show `succeeded`.

## Remediation

Per pledge:

```ruby
require_relative 'lib/support_tasks/sync_incremental_pledge_to_stripe'
SupportTasks::SyncIncrementalPledgeToStripe.perform(pledge_id: <rosie pledge.id>)  # dry-run, then dry_run: false
```

## Code link

[rosie/lib/support_tasks/sync_incremental_pledge_to_stripe.rb@demo/stripe-sync-plan-apply](https://github.com/kickstarter/rosie/blob/demo/stripe-sync-plan-apply/lib/support_tasks/sync_incremental_pledge_to_stripe.rb)
— not yet on `main`.

## Verify-after

Increment/funds_capture/pledge states flip to `collected`; for the second pledge, confirm the
partial-refund amount is still reflected correctly post-sync.

## Citation

Task docstring; no dedicated Guru card yet for this specific task.

---
[← Previous: CHECK-304](check-304.md) | [Index](README.md) | [Next: CHECK-250 →](check-250.md)
