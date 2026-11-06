# Migration kit — HubSpot Hygiene Agent → Claude Code Routines

Everything below is what you need to manually create both routines. Nothing here needs to be built; it's ready to paste.

## 0. Repo setup (one-time)

1. Create a new repo (e.g. `revops-agents`) — private is fine, this holds no secrets.
2. Push this folder's contents as-is: `CLAUDE.md` at the root, `.claude/skills/` below it.
3. Connect GitHub to Claude Code if you haven't (Settings → Connectors → GitHub), and confirm the **Claude GitHub App** is installed with access to this repo specifically — `github.com/settings/installations`. OAuth-connecting alone sometimes skips the app-install step, which is the most common reason a repo doesn't show up in the routine's repo picker later.

## 1. Routine — `hubspot-hygiene-scan` (weekly)

**Repository:** the repo from step 0.

**Connectors:** HubSpot, Slack, Notion. Remove everything else the routine picker includes by default — this routine doesn't need Google Drive, Gmail, etc.

**Trigger:** Scheduled, weekly. Suggested: Monday 8:00am local time. Adjust to whatever slot makes sense once you see how long a run actually takes.

**Prompt:**

```
Run the hubspot-hygiene-agent skill now, exactly as specified in its SKILL.md — Task 1 (Weekly Hygiene Scan), Task 2 (Standing Hygiene Mandate), and Task 3 (Demo Completed Reconciliation), in that order.

This is a fully unattended run — no one is here to answer questions. Follow every escalation and governance rule in the skill exactly rather than skipping a step because it's ambiguous: flag ambiguous cases, don't guess; never edit or create anything in HubSpot; write the Run Log row before finishing.

If the HubSpot, Slack, or Notion connector is still missing after the skill's Connectors retry rule, or a Notion page/database the skill depends on can't be found, stop immediately and DM Will (Slack user U0BQ5AZK284) a clear failure note instead of proceeding partially or guessing at missing data. Post nothing to #hubspot-hygiene. If Slack itself is unreachable, send the failure note as a push notification instead.
```

## 2. Routine — `hygiene-agent-review` (monthly)

**Repository:** same repo.

**Connectors:** Notion, Slack. No HubSpot needed — this routine only reads the Run Log, the Review Log, and Slack reactions/threads; it never touches HubSpot directly.

**Trigger:** Scheduled, monthly, day 1, same time as the weekly routine.

**Prompt:**

```
Run the hygiene-agent-review skill now, exactly as specified in its SKILL.md.

Check the Run Log against all five triggers (signal hit-rate anomaly, escalation backlog growth, repeated Task 2 proposals, field-reference failures, negative Slack feedback via reactions/threads on past #hubspot-hygiene posts). If fewer than 2 unreviewed Run Log rows exist, say so and stop rather than forcing a conclusion from too little data.

If nothing crosses a trigger threshold, log exactly one "No Findings" row and stop — that is the correct, expected outcome most cycles, not a failure to find something. Do not manufacture a marginal or cosmetic finding to have something to report.

If something does cross a threshold, log it to the Review Log with the specific evidence, and present the finding and proposed action to Will in chat rather than applying anything automatically — this skill never edits HubSpot, the Notion definitions page, or the hubspot-hygiene-agent skill file on its own.
```

## 3. Context / memory — what each routine actually needs, and why there isn't more of it

Routines don't share memory across runs. Each one is a fresh session that only knows: the prompt above, whatever's in the repo, and its connectors. That's deliberate, and it's why:

- **`CLAUDE.md`** (repo root) orients any session that clones this repo — what the two skills do, and a hard pointer to Notion as the source of truth for anything business-logic-shaped. You don't need to paste field names or thresholds into the routine prompt; the skill itself fetches them from Notion every run.
- **The skill files** carry the actual task logic, escalation rules, and output format. That's why the prompts above are short — they just say "run the skill," they don't restate what the skill already says.
- **Nothing else needs to be added.** If you find yourself wanting to paste extra context into a prompt, that's usually a sign something belongs in the Notion definitions page instead, so it doesn't have to be repeated by hand every time you touch a routine.

## 4. One thing worth remembering before you turn these on

Routine connectors get full tool access — including writes — with no per-action confirmation during a run. Both skills are written with "never edit HubSpot" as a hard rule, but that rule lives in the prompt/skill, not in a platform permission wall. Separately: the HubSpot connector currently reports `REQUIRES_PERMISSION_MODIFICATION` for write access on the Deal object, so even an attempted write would fail right now — a real backstop, just not one to assume will always be there if HubSpot write access ever gets granted for something unrelated later.

## 5. Test before you trust it

Use **Run now** on each routine the first time rather than waiting for the schedule. Read the actual Slack output and the actual Run Log row it writes before letting either one run unattended on a timer.
