[← Previous: CHECK-306](check-306.md) | [Index](README.md) | [Next: CHECK-303 →](check-303.md)

# CHECK-304 — PLOT installment not attempted, still errored

**Refined 2026-07-01, then confirmed in production the same day** — the remediation posted in
[comment 152361](https://kickstarter.atlassian.net/browse/CHECK-304?focusedCommentId=152361) picked
between two branches without ever inspecting the pledge's own `state`. The EM pointed to the Guru "PLOT
Retry/Resurrect" card, ran the corrected sequence, and it worked: increment #1007462 collected on the
visa ending `0643`, future installments rescheduled for 7/4 and 8/4 (see the ticket's Decision Log and
[comment 152370](https://kickstarter.atlassian.net/browse/CHECK-304?focusedCommentId=152370)). See "Why
this refinement" below — the original two branches are kept, now as explicit fallbacks gated behind a
state check instead of a guess.

| Field | Value |
|---|---|
| Priority | High |
| Classification | Collection never ran — confirmed **dropped pledge** (`state: inactive`, dibs released; no payment source, no funds capture ever created) |
| Confidence | High (confirmed via the ticket's Decision Log: `pledge.payment_source` was `nil`, `increment.funds_capture` was `nil`) |
| Status | **Resolved 2026-07-01** — reactivation, redib, and collection all succeeded in production |
| System | rosie |
| Pledge | https://www.kickstarter.com/admin/pledges/194526562 |
| Project | https://www.kickstarter.com/admin/projects/5269345 (shared with CHECK-250) |
| ids | kickstarter backing id `194526562` → rosie pledge via `Pledge.find_by(client_id: 1, foreign_key: 194526562)` |

## Why this refinement

The ticket's own evidence — "no payment attempt in Stripe at all," "backer does not want to lose the
pledge" — matches the Guru card's "undrop a dropped pledge" scenario (a backer's payment source was
removed or went invalid, so the PLOT pledge silently stopped attempting) better than a generic "stuck
increment." In rosie's `Pledge` state machine, the *only* path to a pledge going quiet is the
`deactivate` event (`[ACTIVE, COLLECTING] → INACTIVE`), whose `after_transition` callback releases
active dibs — an `INACTIVE` pledge with no attempts is the expected shape of "dropped," not an anomaly.
The original remediation's inspection step never printed `pledge.state`, so it couldn't distinguish
"dropped" from "genuinely stuck."

This isn't cosmetic: `PaymentIncrements::CollectPaymentIncrement` raises
`IncrementNotAttemptableStateException` unless `pledge.state == Pledge::COLLECTING`, and
`Pledges::Update` raises `Rosie::PledgeLocked` if `pledge.inactive?`. If the pledge is dropped,
collection can't succeed until it's reactivated **in this order**: raw state write → `Pledges::Update`
(redib) → `CollectPaymentIncrement`. Verified directly against `origin/main` — all three are already on
rosie `main`, none are prototype code.

## Pre-flight

Neither downstream call has a dry-run mode, so inspection **is** the dry run — read this first, now
including `pledge.state` (the original inspection didn't print it):

```ruby
pledge = Pledge.find_by!(foreign_key: 194526562, client_id: 1)
puts "Pledge state: #{pledge.state} (reason: #{pledge.state_reason})"
pledge.active_dibs.map { |d| { id: d.id, state: d.state, payment_source_id: d.payment_source_id } }
pledge.payment_increments.map { |pi| { id: pi.id, state: pi.state, state_reason: pi.state_reason, scheduled_collection: pi.scheduled_collection } }
```

## Remediation

Branch on `pledge.state` from the inspection above.

**If `pledge.state == "inactive"` (dropped — matches this ticket's symptom):**

> **Required input: `target_last4`.** Which saved card the backer wants used is a customer-facing
> decision this remediation cannot infer from rosie state alone — name it as a prerequisite, don't treat
> the placeholder below as a formality. On this ticket it was **inferred**, not asked fresh — from the
> Zendesk ticket text plus the customer's recent Stripe activity (`target_last4 = "0643"`). State
> explicitly whether your value is *confirmed* (backer said so) or *inferred* (from context): a wrong
> card here reattempts a real charge against the wrong instrument.

```ruby
target_payment_source = pledge.user.payment_sources.active.find { _1.provider&.credit_card&.last4 == target_last4 }

# Raw state write — there is no state-machine event that reverses `deactivate`, and
# Pledges::Update below rejects pledge.inactive? pledges, so this has to happen first.
pledge.update!(state: Pledge::COLLECTING, state_reason: "manual_remediation_retry_after_dropped")

# Reattaches the payment source and creates/activates a new dib (via Pledges::Redib internally).
Pledges::Update.new(pledge:, payment_source: target_payment_source, intent_client_secret: nil).call

errored_increment = pledge.payment_increments.errored.order(scheduled_collection: :asc).last
result = pledge.with_lock do
  errored_increment.with_lock do
    PaymentIncrements::CollectPaymentIncrement.new(
      pledge_id: pledge.id,
      payment_increment_id: errored_increment.id,
      payment_source_id: nil, # uses the pledge's now-current payment source
      allow_early_collection: true
    ).call
  end
end

# A dropped pledge's future-increment jobs don't survive reactivation — they are NOT recreated
# automatically. Check, then (re)schedule the next one:
next_payment_increment = pledge.payment_increments.collected.last&.next_attemptable
existing_jobs = T.unsafe(FundsCaptures::IncrementalPledgeCollectionJob).jobs_with_args(pledge_id: pledge.id)
raise "a job is already scheduled for this pledge — investigate before proceeding" if existing_jobs.present?
FundsCaptures::IncrementalPledgeCollectionJob.create(pledge.id, next_payment_increment.id, nil, run_at: next_payment_increment.scheduled_collection) if next_payment_increment
```

**If `pledge.state` is already `"active"` or `"collecting"` (increment itself is just stuck — no
reactivation needed):**

```ruby
errored_increment = pledge.payment_increments.errored.order(scheduled_collection: :asc).last
result = pledge.with_lock do
  errored_increment.with_lock do
    PaymentIncrements::CollectPaymentIncrement.new(
      pledge_id: pledge.id,
      payment_increment_id: errored_increment.id,
      payment_source_id: nil,
      allow_early_collection: true
    ).call
  end
end
```

**If increments don't exist at all yet** (original second branch — kept as-is):

```ruby
Pledges::Collect.call(pledge: pledge)
```

## Then: sync the kickstarter backing

Once the rosie pledge is no longer `inactive`, the backing needs to reflect it — but
`ResyncBackingWithPledge` silently no-ops for `ACTIVE`/`COLLECTING` (it only handles
`COLLECTED`/`INACTIVE`; see the catalog's `ResyncBackingWithPledge` gap note). Use the Guru card's
pattern instead:

```ruby
backing = Backing.find(194526562)
new_status = case backing.rosie_pledge.state
when Rosie::Pledge::ACTIVE, Rosie::Pledge::COLLECTING
  Backing::STATE_PLEDGED
when Rosie::Pledge::COLLECTED
  Backing::STATE_COLLECTED
else
  backing.status
end
backing.update!(status: new_status, status_reason: "manual_remediation_retry_after_dropped")
```

## Code link

- [rosie/app/services/payment_increments/collect_payment_increment.rb@main](https://github.com/kickstarter/rosie/blob/main/app/services/payment_increments/collect_payment_increment.rb)
- [rosie/app/services/pledges/update.rb@main](https://github.com/kickstarter/rosie/blob/main/app/services/pledges/update.rb)
- [rosie/app/services/pledges/collect.rb@main](https://github.com/kickstarter/rosie/blob/main/app/services/pledges/collect.rb)
- [rosie/app/jobs/funds_captures/incremental_pledge_collection_job.rb@main](https://github.com/kickstarter/rosie/blob/main/app/jobs/funds_captures/incremental_pledge_collection_job.rb)

All confirmed on `main` as of 2026-07-01 — no prototype branch involved, and no `require_relative`
needed (these are autoloaded `app/services` classes, not `lib/support_tasks/`).

## Verify-after

`result` is `PaymentIncrements::CollectPaymentIncrement::Outcomes::Success`/`Failure`/`NoOp` — check
which. On success: a Stripe PaymentIntent appears for the backer's card, and the increment flips to
`collected`. On failure: the increment goes back to `errored` with a `state_reason` (e.g. the card was
genuinely declined) — that's a legitimate outcome, not a bug in the remediation, and routes back to the
backer for a different card. Also confirm a job now exists for the *next* increment (see the reschedule
step above) — don't assume it survived reactivation.

**Confirmed on this ticket:** `Outcomes::Success` on both `Pledges::Update` and
`PaymentIncrements::CollectPaymentIncrement`; increment #1007462 collected on the visa ending `0643`;
`FundsCaptures::IncrementalPledgeCollectionJob` recreated for increment #1007463 (scheduled 7/4).

## Citation

- https://app.getguru.com/card/cAzenGei/PLOT-Retry-Resurrect-How-to-manually-update-payment-source-for-PLOT-Pledge-Over-Time-and-retry-payment-for-a-dropped-pledge (primary — undrop + backing resync)
- https://app.getguru.com/card/TgXyyRyc/Troubleshooting-PLOT-collections (original citation; still applies to the "increments don't exist yet" branch)

---
[← Previous: CHECK-306](check-306.md) | [Index](README.md) | [Next: CHECK-303 →](check-303.md)
