[← Previous: CHECK-306](check-306.md) | [Index](README.md) | [Next: CHECK-303 →](check-303.md)

# CHECK-304 — PLOT installment not attempted, still errored

| Field | Value |
|---|---|
| Priority | High |
| Classification | Collection never ran |
| Confidence | Medium |
| System | rosie |
| Pledge | https://www.kickstarter.com/admin/pledges/194526562 |
| Project | https://www.kickstarter.com/admin/projects/5269345 (shared with CHECK-250) |
| ids | kickstarter backing id `194526562` → rosie pledge via `Pledge.find_by(client_id: 1, foreign_key: 194526562)` |

## Why this classification

Ticket already confirms: first PLOT installment was set to collect on 6/4, backer received errored
messaging, but there's no payment attempt in Stripe at all. That's "collection never ran," not state
drift — nothing to sync, something needs to be (re-)kicked off.

## Pre-flight

Neither downstream call below has a dry-run mode, so inspection **is** the dry run:

```ruby
pledge = Pledge.find_by!(foreign_key: 194526562, client_id: 1)
pledge.active_dibs.map { |d| { id: d.id, state: d.state, payment_source_id: d.payment_source_id } }
pledge.payment_increments.map { |pi| { id: pi.id, state: pi.state, state_reason: pi.state_reason, scheduled_collection: pi.scheduled_collection } }
```

## Remediation

Pick based on the inspection above:

```ruby
# increment exists, just stuck — re-drive it
payment_increment = PaymentIncrement.find(<ID from inspection>)
FundsCaptures::IncrementalPledgeCollectionJob.create(pledge.id, payment_increment.id, pledge.payment_source&.id, run_at: Time.current)

# OR nothing was ever created — kick off collection fresh
Pledges::Collect.call(pledge: pledge)
```

## Code link

- [rosie/app/services/pledges/collect.rb@main](https://github.com/kickstarter/rosie/blob/main/app/services/pledges/collect.rb)
- [rosie/app/jobs/funds_captures/incremental_pledge_collection_job.rb@main](https://github.com/kickstarter/rosie/blob/main/app/jobs/funds_captures/incremental_pledge_collection_job.rb)

Both existing, used as-is.

## Verify-after

A Stripe PaymentIntent gets created for the backer's card; the increment leaves `errored`/`unattempted`.

## Citation

[Troubleshooting PLOT collections](https://app.getguru.com/card/TgXyyRyc/Troubleshooting-PLOT-collections)

---
[← Previous: CHECK-306](check-306.md) | [Index](README.md) | [Next: CHECK-303 →](check-303.md)
