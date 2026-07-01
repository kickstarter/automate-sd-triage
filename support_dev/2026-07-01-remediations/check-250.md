[← Previous: CHECK-303](check-303.md) | [Index](README.md) | [Next: CHECK-236 →](check-236.md)

# CHECK-250 — HRC1 (Glass Cannon Unplugged): blank dialog on errored PLOT payment

| Field | Value |
|---|---|
| Priority | Medium (downgraded from High on 2026-06-17 — reserved for cases with money already out of sorts) |
| Classification | UI symptom, state varies |
| Confidence | Needs-diagnosis |
| System | rosie |
| Pledge | https://www.kickstarter.com/admin/pledges/194206753 |
| Project | https://www.kickstarter.com/admin/projects/5269345 (shared with CHECK-304) |
| ids | kickstarter backing id `194206753` → rosie pledge via `Pledge.find_by(client_id: 1, foreign_key: 194206753)` |

## Why this classification

A blank dialog trying to fix an errored PLOT payment is a UI symptom of an underlying state mismatch —
the mismatch itself varies per ticket. Lead with a dry-run diagnosis rather than assuming which fix applies.

## Remediation

Dry-run `SyncIncrementalPledgeToStripe` first:

```ruby
require_relative 'lib/support_tasks/sync_incremental_pledge_to_stripe'
SupportTasks::SyncIncrementalPledgeToStripe.perform(pledge_id: <rosie pledge.id>)
```

- Report shows a `succeeded` Stripe PI not reflected locally → go live with the sync (`dry_run: false`),
  same as [CHECK-303](check-303.md).
- Stripe shows no successful PI (healthy-not-succeeded) → this is the [CHECK-304](check-304.md)
  collection-retry path instead (`Pledges::Collect.call` / re-drive `IncrementalPledgeCollectionJob`),
  not a sync.

## Code link

[rosie/lib/support_tasks/sync_incremental_pledge_to_stripe.rb@demo/stripe-sync-plan-apply](https://github.com/kickstarter/rosie/blob/demo/stripe-sync-plan-apply/lib/support_tasks/sync_incremental_pledge_to_stripe.rb)
(or the CHECK-304 code links, depending on which branch the dry-run points to).

## Verify-after

The "fix payment" dialog in admin renders correctly post-fix — the blank dialog was likely a symptom of
the state mismatch, not a separate bug.

## Citation

- [Remediating Errored PLOT Payments](https://app.getguru.com/card/ibxroprT/Remediating-Errored-PLOT-Payments)
- [Troubleshooting PLOT collections](https://app.getguru.com/card/TgXyyRyc/Troubleshooting-PLOT-collections)

---
[← Previous: CHECK-303](check-303.md) | [Index](README.md) | [Next: CHECK-236 →](check-236.md)
