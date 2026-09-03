---
name: hygiene-agent-review
description: Monthly review cycle for the hubspot-hygiene-agent skill — checks the Run Log for drift (signal hit-rate anomalies, growing escalation backlog, repeated Task 2 proposals, field-reference failures, negative Slack feedback) and proposes calibration or fixes back to Will. Most cycles should find nothing — that's success, not a failure to try hard enough. Runs monthly on whatever schedule triggers it (Claude Code Routine, Cowork cadence, or a local cron job), or on demand ("run the hygiene agent review", "check if the hygiene agent needs tuning", "did anything come up with the hygiene scans this month").
---

## Context

`hubspot-hygiene-agent` runs weekly and never adjusts itself — by design, it can't tell if a threshold is miscalibrated or a signal is missing something real. This skill is the part of the system that comes back later and checks, using the aggregate run data that skill logs each week.

**The default, expected, correct outcome of most cycles is nothing to report.** This isn't a monthly status update that always needs three bullet points — it's a detector with a threshold. If nothing crosses it, say so in one line and stop. Manufacturing a marginal or cosmetic finding to seem thorough defeats the entire point of separating this from the weekly run: it exists to catch real drift, not to generate monthly busywork.

**Data sources:**
- **[Hygiene Agent Run Log](https://app.notion.com/15bb0815e8994312b86fd88a67aad053)** — per-run signal counts, escalation counts, Task 2 proposals, field-reference failures, Slack post links
- **[Hygiene Agent Review Log](https://app.notion.com/14f11487f8544ce3a17e8c44a9b4f01a)** — this skill's own output, one row per finding (or one "No Findings" row per empty cycle)
- **[Hubspot CRM Data Hygiene Agent](https://app.notion.com/3ce335d59add804680e2daf4790a88f4)** — the definitions page; any Calibrate/Fix proposal that gets approved is edited here (or in the `hubspot-hygiene-agent` skill file for execution-only fixes), never in this skill
- `#hubspot-hygiene` — for reading reactions/thread replies on past weekly posts as feedback signal

## Workflow

### Step 1 — Scope

Query the Run Log for rows where `Reviewed` is unchecked. With a weekly cadence and monthly review, expect ~4. If fewer than 2 unreviewed rows exist, there isn't enough new data for a meaningful check — say so and stop rather than drawing a conclusion from a sample too small to mean anything. Don't mark anything Reviewed in this case; wait for more runs.

### Step 2 — Check each trigger against the data

Pull enough Run Log history (not just this cycle's new rows) to compute trailing averages — a signal's count this month only means something relative to its own history.

1. **Signal hit-rate anomaly** — any signal's count this period deviates sharply (roughly 2x above or 0.5x below its trailing average) from prior runs.
2. **Escalation backlog growing** — `Escalation Count` trending up across 3+ consecutive runs without a corresponding drop (i.e., cases are piling up, not just noisy week to week).
3. **Repeated Task 2 proposal** — the same pattern or theme appears in `Task 2 Proposals` across 3+ runs, historically, not just this batch.
4. **Field reference failure** — any row in this batch has a non-empty `Field Reference Failures`.
5. **Negative feedback pattern** — read reactions (and thread replies, if any) on each new run's `Slack Post` link. Look for 👎 or explicit pushback clustering on one signal specifically, not scattered one-offs.

### Step 3 — Check prior open findings

Query the Review Log for `Status` = "Awaiting Will" or "Still Deferred". New data might resolve a Deferred item (now there's enough evidence for a real verdict) or leave it exactly where it was. Items already `Rejected` or `Approved` are closed — don't revisit them without Will raising it again.

### Step 4 — Verdict

For anything that actually fired in Step 2, or resolved in Step 3, assign one verdict, grounded in the specific numbers pulled — never "seems off" without the data behind it:

| Verdict | Meaning |
|---|---|
| Calibrate | A threshold or definition in the Notion page is off |
| Fix | An execution bug in the `hubspot-hygiene-agent` skill file itself |
| Expand | A Task 2 proposal has enough repeated evidence to formalize as a real signal |
| Defer | Something's worth watching, not yet enough evidence to act |
| Dismissed | Will already decided no change is needed (only set this after he says so — don't self-dismiss) |
| No Findings | Nothing crossed any trigger this cycle |

If nothing fired and nothing resolved: log exactly one row with Verdict = No Findings, mark all this cycle's rows Reviewed, and stop.

### Step 5 — Log and propose

Write to the Review Log: one row per finding (Trigger, Verdict, Evidence in specific numbers, Proposed Action, Status = "Awaiting Will"), or the single No Findings row.

For any real finding, present it to Will in chat with the evidence and a specific proposed action — a specific Notion field/threshold edit, or a specific skill-file change — and wait for explicit sign-off. Nothing gets applied autonomously; this skill's only unsupervised action is logging and proposing, same governance model as `hubspot-hygiene-agent` itself.

Mark every Run Log row covered by this cycle as `Reviewed`, regardless of verdict — a row that produced no finding was still reviewed.

## Hard rules

1. No autonomous edits, anywhere — not to HubSpot, not to the Notion definitions page, not to the `hubspot-hygiene-agent` skill file. Every output is a log entry and, where warranted, a proposal.
2. "No Findings" is the expected common case. Don't pad a quiet cycle with marginal or cosmetic suggestions.
3. Never re-propose a Dismissed finding without genuinely new evidence beyond what got it dismissed the first time.
4. Every number in a finding traces to an actual Run Log row or a real Slack reaction/reply — never estimated, never remembered from a previous cycle.
5. Skip the cycle (don't force a verdict) if there's fewer than 2 new runs to check.

## Workflow position

| Cadence | What happens | Skill |
|---|---|---|
| Weekly | Deals scanned, flags posted, one Run Log row written | `hubspot-hygiene-agent` |
| Monthly | Run Log checked for drift, findings verdicted, proposals raised | `hygiene-agent-review` (this skill) |
| As needed | Approved Calibrate/Fix/Expand actions applied | Will — in the Notion definitions page or the skill file directly |
