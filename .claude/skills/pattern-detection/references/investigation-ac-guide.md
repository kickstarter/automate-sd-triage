# Writing Good Acceptance Criteria for Investigation Tickets

This reference is for the `pattern-detection` skill. Read it if you're uncertain how to frame the acceptance criteria for a root-cause investigation ticket.

---

## Core Principle: Investigate, Don't Implement

An investigation ticket is done when the team knows enough to make a decision — not when a fix is shipped. The AC should close off ambiguity, not deliver a feature.

A good set of AC answers:
1. What is the root cause?
2. How many users are actually affected?
3. What should we do about it? (fix now / workaround / defer / won't fix)
4. If we fix it: what does that work look like? (captured in a new ticket)

---

## What Good AC Looks Like

✅ **Good:**
```
[ ] Root cause identified: the specific code path, system, or data condition responsible for the failure is documented in this ticket
[ ] Affected user scope confirmed: total count or reasonable estimate with methodology
[ ] Decision documented: fix, workaround, or defer — with rationale
[ ] If fix warranted: follow-on implementation ticket created and linked here
[ ] Symptom tickets updated with a summary of findings
```

✅ **Good (for a timing-related issue):**
```
[ ] Confirmed whether this is a regression (if so: linked to the deploy or change that introduced it)
[ ] Reproduction conditions documented: deterministic or intermittent? Under what conditions?
[ ] Established whether a temporary mitigation is possible while a fix is scoped
```

---

## What Bad AC Looks Like

❌ **Too prescriptive (assumes the fix before investigating):**
```
[ ] Retry logic added to the payment gateway client
[ ] Database index added to the user_sessions table
```
*Why bad: You don't know these are the right fixes yet. This constrains the investigator.*

❌ **Too vague (doesn't close off ambiguity):**
```
[ ] Issue investigated
[ ] Team aligned on next steps
```
*Why bad: These could mean anything. They don't force a concrete answer.*

❌ **Confusing investigation with implementation:**
```
[ ] Bug fixed and deployed to production
```
*Why bad: A fix is out of scope for an investigation ticket. Create a follow-on.*

---

## Optional AC (add when relevant)

Only include these if there's genuine reason to think they apply:

```
[ ] Monitoring/alerting gap identified — is there a way we would have caught this earlier?
[ ] Related edge cases documented — are there adjacent scenarios that share the failure mode?
[ ] Data cleanup needed — are there users in a bad state who need intervention?
[ ] Postmortem warranted — was this severe enough to warrant a blameless postmortem?
```

---

## Framing the "What We Don't Know" Section

This section is often the most useful part of an investigation ticket. It tells the engineer exactly what they're trying to find out, rather than leaving them to infer the question from the symptoms.

Good questions to include:
- Is this deterministic or intermittent? What are the exact reproduction conditions?
- Is this a regression? If so, what changed in the relevant time window?
- What is the failure point? (Client? API? Payment processor? DB?)
- Are there relevant logs or traces we haven't looked at yet?
- Are the CS tickets a representative sample, or is there a larger silent population?
- Is there a safe workaround that CS can use while this is investigated?

Bad questions to include:
- "Why is this happening?" (too generic)
- "Is this fixable?" (not a useful scoping question)
- "Should we prioritize this?" (that's a product/EM decision, not part of the investigation)

---

## Tone

Investigation tickets will be read by senior engineers who don't need hand-holding. Write the ticket as you would write notes to a peer who's smart, busy, and hasn't been following this thread closely. Be precise, not exhaustive. If in doubt, leave it out.
