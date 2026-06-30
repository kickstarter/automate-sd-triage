# Remediation Catalog

Durable reference mapping CHECK support-dev ticket symptoms to known manual remediations. Consulted by
the `check-remediation` skill. Keep it current as support tasks and Guru runbooks change.

**Two systems, two consoles.** Remediations run against either the **rosie** or the **kickstarter**
support-task library, each in its own production console. Every entry below is tagged with its
`system`. The skill opens the console named by the tag — it never infers which system a task belongs
to. Confirm a task's branch/path at runtime before relying on a `require_relative`.

> ## Source of truth
>
> The **living `main` of the kickstarter and rosie GitHub repos is the authoritative list of which
> support tasks exist.** Verify availability there at runtime (e.g. `gh api`/`gh search code`, or
> `git fetch && git ls-tree origin/main -- lib/support_tasks/`) rather than trusting this file to be
> complete. **The task tables below are curated examples** — the common remediations and the
> symptom→task mapping — not an exhaustive or always-current inventory. A task that isn't on `main`
> (e.g. the rosie PLOT tasks) is a prototype on a working branch, not a generally-available task.

---

## ID conventions (READ FIRST — easy to get wrong)

**All ids in `kickstarter.com` admin URLs are _kickstarter_ ids, not rosie.** A `/admin/pledges/<id>`
id is a **kickstarter backing id**, not a rosie `Pledge` primary key.

- **kickstarter console** operates on kickstarter ids directly: `Backing.find(<backing id from URL>)`.
- **rosie console** operates on rosie `Pledge` ids — resolve from the kickstarter backing id:

  ```ruby
  pledge = Pledge.find_by(client_id: 1, foreign_key: <backing id from admin URL>)  # client_id = 1 in prod
  # pass pledge.id to rosie support tasks that expect a rosie pledge_id
  ```

- Reverse (rosie → kickstarter): `pledge.foreign_key` is the kickstarter backing id; `Backing.find(it)`
  in a kickstarter console.
- Per-increment commands need the `PaymentIncrement` id, read off the pledge inspect page — it is NOT
  in the URL.

Label which id is which (kickstarter backing id vs rosie pledge id vs Stripe PI) in every snippet.

---

## Consoles

| System | Open with | Loads tasks from |
|---|---|---|
| rosie | `cd rosie && ksr console production` (PRODUCTION — `exit` when done) | `rosie/lib/support_tasks/` |
| kickstarter | kickstarter production Rails console | `kickstarter/lib/support_tasks/` |

rosie console service objects are loaded with `require_relative`, exist only for the session, and
default to `dry_run: true`.

---

## rosie support tasks — `system: rosie` (examples — verify against live `main`)

| Service object | File | Branch | Use |
|---|---|---|---|
| `SyncIncrementalPledgeToStripe` | `lib/support_tasks/sync_incremental_pledge_to_stripe.rb` | `demo/stripe-sync-plan-apply` | Reconcile a PLOT pledge's increment / funds-capture / pledge state against the authoritative Stripe PaymentIntent. Flips `errored → collected`, un-cancels later increments, re-marks pledge, re-schedules collection. Entry: `.perform(pledge_id:, dry_run: true)`. |
| `RefundPaymentIncrementFromKsr` | `lib/support_tasks/refund_payment_increment_from_ksr.rb` | `demo/stripe-sync-plan-apply` | Refund a single increment, cost borne by KSR. |
| `RefundIncrementFromProjectPaymentSource` | `lib/support_tasks/refund_increment_from_project_payment_source.rb` | `demo/stripe-sync-plan-apply` | Refund remaining collectible from the creator's card. |
| `RefundIncrementFromDepositAccount` | `lib/support_tasks/refund_increment_from_deposit_account.rb` | `demo/stripe-sync-plan-apply` | Refund from the creator's Stripe connected account. |
| `ReactivatePledgeFromPledgedBacking` | `lib/support_tasks/reactivate_pledge_from_pledged_backing.rb` | `main` | (Draft, untested) Reactivate a pledge from a known-good backing's payment source (shared-pledge / double-charge). |

> The PLOT-sync/refund tasks are **not on `main`** yet — they live on the working branch
> `demo/stripe-sync-plan-apply`, which is their prototype branch (see "Prototype working branches").

---

## kickstarter support tasks — `system: kickstarter` (examples — verify against live `main`)

Run in the kickstarter production console; operate on **kickstarter** ids (`Backing`, etc.).

| Service object | File | Use |
|---|---|---|
| `ResyncBackingWithPledge` | `lib/support_tasks/resync_backing_with_pledge.rb` | "Unsettled Backings": backing failed to receive the pledge-collected signal while the pledge is in a correct final state. `.call(admin_id:, backing_id:)`. |
| `FixOrphanedBacking` | `lib/support_tasks/fix_orphaned_backing.rb` | Rosie pledge `active` but KSR backing stuck in pre-auth ($0, no `pledge_key`) after a failed checkout signal. `.new(backing_id:, project_id:, pledge_id:, payment_source_id:).call`. |
| `DropErroredPledges` | `lib/support_tasks/drop_errored_pledges.rb` | Drop errored pledges (often fraudulent). `.batch_run(admin_id:, backing_ids:, state_reason: nil)`. |
| `SyncRefundCheckout` | `lib/support_tasks/sync_refund_checkout.rb` | Sync a `RefundCheckout` to a successful `Rosie::Refund` (legacy 3DS-delayed refunds). `.call(refund_checkout:, rosie_refund:)`. |

> Earlier drafts mis-filed `ResyncBackingWithPledge` as a rosie task — it is **kickstarter**.

---

## Guru runbooks

| Card | URL | Covers |
|---|---|---|
| Remediating Errored PLOT Payments | https://app.getguru.com/card/ibxroprT/Remediating-Errored-PLOT-Payments | Suspended Stripe capabilities (→ T&S) and payment-already-succeeded (→ flip increment to `collected`). |
| Troubleshooting PLOT collections | https://app.getguru.com/card/TgXyyRyc/Troubleshooting-PLOT-collections | Pledge with no increments; re-run `Pledges::Collect`; collect outside the normal window. |
| Campaigns::CollectJob - CollectionBlocked | https://app.getguru.com/card/pc5L9ybc/CampaignsCollectJob-CollectionBlocked | Backing↔pledge count/amount conflicts, orphaned backings, shared pledge keys, resync. |
| Unsettled Backings | https://app.getguru.com/card/iedA66bT/Unsettled-Backings | Backings that missed the pledge-collected signal — `ResyncBackingWithPledge`. |
| Who can enter in Pledge Manager? | https://app.getguru.com/card/TxGM4pXc/Who-can-enter-in-Pledge-Manager- | Auto-exempt semantics; re-entry after completed checkout normally disallowed. |
| Pledge Manager - Backer Overview | https://app.getguru.com/card/cay9g5ei/Pledge-Manager-Backer-Overview | Admin "Refresh order" as first-line for stuck PM orders. |
| Manual payouts | https://app.getguru.com/card/9cebEq5c/Manual-payouts | Payout retries, wrong-account reversals, out-of-system payouts. |

Search Guru (Support team bot) for anything not listed — this catalog is not exhaustive.

---

## Symptom → remediation map

| Symptom (from ticket / CS) | Likely class | System | First move |
|---|---|---|---|
| Increment shows `errored` but Stripe shows the charge `succeeded` | State drift | rosie | `SyncIncrementalPledgeToStripe` (dry-run → live) after confirming PI `succeeded` in-console |
| PLOT errored, **no payment attempts found** | Collection never ran / increments missing | rosie | Troubleshooting-PLOT path: confirm capabilities, then `Pledges::Collect.call` or re-drive `IncrementalPledgeCollectionJob` |
| Blank/empty dialog or red box fixing an errored PLOT pledge | UI symptom; state varies | rosie | Dry-run sync to diagnose: succeeded → sync; healthy-not-succeeded → re-drive |
| Backer can't collect; Stripe capabilities **suspended** | Not an SD fix | — | Tag Trust & Safety (payments) — STOP |
| Backing active but pledge inactive (or vice versa); collection blocked | Backing↔pledge drift | kickstarter | `ResyncBackingWithPledge` (Unsettled Backings / CollectionBlocked) |
| Rosie pledge active, KSR backing stuck pre-auth ($0, no pledge_key) | Orphaned backing | kickstarter | `FixOrphanedBacking` |
| Errored / likely-fraudulent pledges to drop | Drop errored | kickstarter | `DropErroredPledges.batch_run` |
| RefundCheckout not matching a successful rosie refund (legacy 3DS) | Refund sync | kickstarter | `SyncRefundCheckout` |
| PM order stuck / voucher not applied / data stale | PM order needs refresh | — (admin UI) | Admin **Refresh order** (per-order; see blast-radius note) |
| Email/template rendering bug (e.g. Outlook dark mode) | Code fix, not data | — | No console remediation; route to engineering |

When the class is ambiguous, say so and lead with a diagnostic dry-run rather than a state change.

---

## Diagnosis via Looker (read-only — do this before opening a console)

Diagnose and confirm scope in Looker, not a production console. Useful explores:

- `rosie_payment_increments` — `state`, `state_reason`, `scheduled_collection` per increment.
- `rosie_pledges`, `rosie_payment_intents`, `funds_captures` — pledge/PI/capture state.
- `rosie_errored_pledges_over_time` — `is_collected` / `is_dropped` / `is_success_present` /
  `number_of_errors` by project/backer; affected-scope counts.
- `cs.model` — Zendesk ↔ backings/projects/users for cohort overlap.

**Looker lags production.** The authoritative just-before-write check (e.g. "is this Stripe PI
`succeeded` right now?") is taken in-console/Stripe immediately before the dry-run — never from Looker.

---

## Standard pre-flight for any PLOT state change

1. **Rule out suspended Stripe capabilities** (project admin → Rosie deposit account → Connect
   dashboard → capabilities). Suspended → STOP, tag T&S.
2. **Confirm the Stripe PI state** for the increment (in-console, not Looker):
   ```ruby
   payment_increment = PaymentIncrement.find(<INCREMENT_ID from pledge inspect>)
   payment_increment.funds_capture.stripe_payment_intent   # expect "succeeded" for a sync
   ```
3. **Always dry-run first.** Read the report before re-running with `dry_run: false`.

---

## Prototype working branches (link code from the Jira comment)

A remediation is backed by **code**, not just an inline snippet:

- **Existing task, used as-is** → link to it at its current branch/path (e.g. the rosie PLOT tasks on
  `demo/stripe-sync-plan-apply`); inline the call in the comment.
- **Would modify an on-`main` task, or warrant a new task file** → open a **working branch** in the
  relevant repo (rosie or kickstarter per the system tag) holding the prototype, so the team can
  examine and test it with one-off remediations. **Confirm before pushing** the branch to origin
  (pushing is outward-facing); then include the link.

The Jira comment carries **both** the prototype link(s) **and** the runnable code blocks — which may
define a new class inline for a single console session (the `require_relative` / paste-into-console
pattern `demo/stripe-sync-plan-apply` already demonstrates). Prototype code is for review and testing;
production runs remain human-only and dry-run-first. Promotion to `main` is ordinary PR review, outside
this toolkit.

---

## Admin-action blast-radius lesson (learned the hard way)

Before worrying that an admin button changed something project-wide, **trace what it actually writes.**
The **"Allow auto-exempt backers to re-enter PM checkout"** checkbox on the order admin page
(`engines/pledge_redemption/app/views/pledge_redemption/admin/orders/show.html.haml`) is the
`reset_must_acknowledges` field of the per-order **Refresh Order** form. Its controller
(`pledge_redemption/admin/orders_controller.rb#refresh`) calls `PR::RefreshOrder.call(order:, ...)`
under a per-row lock and sets `must_acknowledge_*` on **that single order** only. It writes **no**
project or `pledge_manager` record. The label sounds project-wide; the effect is one order.

General rule: the pledge-manager / orders domain lives in the `engines/pledge_redemption/` Rails engine
inside the kickstarter repo. When asked "did I change something project-wide?", grep the engine, trace
form field → controller action → service, and confirm the scope of the writes before speculating.
