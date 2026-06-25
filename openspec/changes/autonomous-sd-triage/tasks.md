## 1. Resolve open questions (do first — they shape the build)

- [ ] 1.1 Capture the exact intake JQL behind quick filters 1230 + 1264; confirm `support-dev`-absent is a sufficient "untriaged" predicate
- [ ] 1.2 Determine what "move into a team's space" means mechanically in Jira (Team field vs component vs project/board) and record the write semantics
- [ ] 1.3 Get an explicit, documented data-handling decision for sending ticket content (incl. PII) to the Claude API; decide on any redaction
- [ ] 1.4 Decide the no-clear-owner fallback (best-guess + flag vs designated holding team)
- [ ] 1.5 Agree the would-be override-rate threshold that gates flipping from shadow to live

## 2. Repo + secrets skeleton (survivability foundation)

- [ ] 2.1 Lay out the `automate-sd-triage` repo structure (agent code, config, README)
- [ ] 2.2 Define the secrets contract (JIRA_BASE_URL, JIRA_USER, JIRA_API_TOKEN, CLAUDE_API_KEY) and read them via credential indirection — no hardcoded identity anywhere
- [ ] 2.3 Add Luke's credentials as GitHub repo/org secrets; document the one-step bot-account swap runbook in the README
- [ ] 2.4 Write the README explaining the system end-to-end for a future owner (how it runs, how to pause, how to edit policy)

## 3. Configuration as policy

- [ ] 3.1 Create `routing-table.yml` (component/keyword/signal → team) from the Team Directory
- [ ] 3.2 Add intake-scope config (board/filter criteria) per task 1.1
- [ ] 3.3 Add a mode flag (`shadow` | `live`) and the no-clear-owner fallback config

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

- [ ] 6.1 Apply priority, labels, and team assignment/move to Jira (defer mechanical steps owned by Jira-native automation)
- [ ] 6.2 Post the self-documenting audit comment (what changed + why + any low-confidence flag)
- [ ] 6.3 Ensure write semantics are idempotent (no duplicate changes/comments on re-run)

## 7. Observability + feedback

- [ ] 7.1 Implement shadow mode (log would-be decisions and comment; no writes)
- [ ] 7.2 Implement the per-run audit trail (tickets considered, decisions, write outcomes)
- [ ] 7.3 Implement override-rate tracking (detect later human changes to priority/labels/team)
- [ ] 7.4 Implement the periodic digest (triaged count, routing distribution, low-confidence flags, override rate)

## 8. Scheduling + rollout

- [ ] 8.1 Add the GitHub Actions cron workflow (~every 20 min), starting shadow-only
- [ ] 8.2 Run shadow mode for the agreed window; review decisions + would-be override rate with the team
- [ ] 8.3 Tune routing table and priority prompting from shadow findings
- [ ] 8.4 Flip to live; monitor override rate + weekly digest
- [ ] 8.5 Verify the bot-account swap runbook (dry run of changing secrets) for when an account is available
