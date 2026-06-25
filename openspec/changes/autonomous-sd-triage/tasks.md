## 1. Remaining facts to gather (do first — they block actuation/routing)

- [x] 1.1 Intake JQL captured: `project = SD AND (labels is EMPTY OR labels not in (support-dev))`; `support-dev`-absent confirmed as the untriaged predicate
- [x] 1.2 Move semantics confirmed: cross-project move into one of the 11 team projects
- [x] 1.3 Data-handling decided: send as-is, no redaction (record in README)
- [x] 1.4 No-clear-owner fallback decided: best-guess route + `needs-review` label
- [x] 1.5 Rollout decided: start live; weekly override-audit job → override logfile
- [x] 1.6 Obtain the project **key** for each of the 11 team projects (needed for the move API)
- [x] 1.7 Field compatibility decided: assume Bug/Task/Query types in target projects are a superset of SD fields; no field remapping this pass (move-failure safety net retained)
- [ ] 1.9 Confirm each target project's TODO-equivalent status name; set `target_status` default + any per-team overrides in `routing-table.yml`
- [x] 1.8 Ownership surfaces confirmed for all 8 live targets; `routing-table.yml` drafted (keyword tuning deferred to post-live)

## 2. Repo + secrets skeleton (survivability foundation)

- [ ] 2.1 Lay out the `automate-sd-triage` repo structure (agent code, config, README)
- [ ] 2.2 Define the secrets contract (JIRA_BASE_URL, JIRA_USER, JIRA_API_TOKEN, CLAUDE_API_KEY) and read them via credential indirection — no hardcoded identity anywhere
- [ ] 2.3 Add Luke's credentials as GitHub repo/org secrets; document the one-step bot-account swap runbook in the README
- [ ] 2.4 Write the README explaining the system end-to-end for a future owner (how it runs, how to pause, how to edit policy)

## 3. Configuration as policy

- [ ] 3.1 Create `routing-table.yml` (keyword/signal → team project) for the 11 team projects, including each project key
- [ ] 3.2 Add intake-scope config (the SD intake JQL from 1.1)
- [ ] 3.3 Add a mode flag (`shadow` | `live`, defaulting to live) and the no-clear-owner best-guess config

## 4. Ticket intake

- [ ] 4.1 Implement the scheduled query for untriaged intake tickets using the configured scope
- [ ] 4.2 Implement idempotent selection (exclude `support-dev`-labeled tickets) and safe re-run behavior

## 5. Triage decisioning

- [ ] 5.1 Gather ticket content + linked Zendesk/CS context for decisioning
- [ ] 5.2 Implement priority decision: fetch the Confluence priority guide at runtime; fail-safe (flag, don't guess) if unavailable
- [ ] 5.3 Implement label decision (`support-dev` + helpful labels)
- [ ] 5.4 Implement routing: deterministic table lookup first, LLM fallback for the ambiguous tail, configured fallback for no-clear-owner
- [ ] 5.5 Produce written rationale + confidence (higher bar on priority than routing)

## 6. Triage actuation

- [ ] 6.1 Apply priority + labels (incl. `support-dev`) before the move; defer mechanical steps owned by Jira-native automation
- [ ] 6.2 Perform the cross-project move into the owning team's project, mapping issue type (Bug→Bug, Task→Task, Query→Task); on move failure, leave labeled + record + surface for human completion
- [ ] 6.2a After the move, force the ticket to the receiving team's configured TODO-equivalent status; if the transition is unreachable, record + surface for human completion
- [ ] 6.3 Apply the `needs-review` label on low-confidence decisions
- [ ] 6.4 Post the self-documenting audit comment (what changed + why + any low-confidence note)
- [ ] 6.5 Ensure write semantics are idempotent (no duplicate changes/comments on re-run)

## 7. Observability + feedback

- [ ] 7.1 Implement shadow mode (log would-be decisions and comment; no writes) as a test/pause switch
- [ ] 7.2 Implement the per-run audit trail (tickets considered, decisions, write outcomes incl. move failures)
- [ ] 7.3 Implement the separate weekly override-audit job: detect later human changes to priority/labels/team, append to the override logfile in the repo
- [ ] 7.4 Implement the periodic digest (triaged count, routing distribution, `needs-review` count, override rate)

## 8. Scheduling + rollout

- [ ] 8.1 Add the main GitHub Actions cron workflow (~every 20 min), mode defaulting to live
- [ ] 8.2 Add the separate weekly cron workflow for the override-audit job
- [ ] 8.3 Smoke-test in shadow mode against a few real tickets (reads, decisions, move behavior), then enable live
- [ ] 8.4 Review the override logfile weekly; tune routing table + priority prompting from the data
- [ ] 8.5 Verify the bot-account swap runbook (dry run of changing secrets) for when an account is available
