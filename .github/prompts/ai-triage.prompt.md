---
mode: agent
description: AI triage of the GitLab backlog — orchestrator. Say GO and follow the gates.
---

# AI Triage — Orchestrator

You are running an AI-suitability triage over this project's open GitLab issues.
Everything you need is in these files:

- `docs/ai-triage/RUNBOOK-ai-triage.md` — the process, rationale, and pitfalls
- `docs/ai-triage/triage-config.yml` — all configuration, single source of truth.
  If `docs/ai-triage/triage-config.local.yml` exists, use it instead — it is the
  filled-in copy that stays out of version control.
- `.github/prompts/ai-triage-analyze.prompt.md` — phase 2, rating (read-only)
- `.github/prompts/ai-triage-writeback.prompt.md` — phase 3, writing to GitLab

Adjust these paths if the repo lays them out differently.

## How you must behave

**Never write anything to GitLab without passing a gate.** A gate means: you stop,
present what you are about to do, and wait for me to reply. Do not treat my earlier
"GO" as blanket approval for later phases.

Work through the steps below in order. Announce each step as you start it.

---

## Step 0 — Preflight

Read `triage-config.yml`, `RUNBOOK-ai-triage.md`, and both phase prompts. Then check:

| Check | How |
|---|---|
| Config complete | No value in `triage-config.yml` is still `<...>` or empty |
| Architecture doc | The path in `arch_doc` exists and is readable |
| Copilot instructions | `.github/copilot-instructions.md` exists |
| GitLab MCP reachable | Fetch **one** open issue and confirm the response shape |
| Labels exist | List project/group labels; confirm all five `AI Support possible::*` labels are present |
| Existing triage | Count open issues that already carry an `AI Support possible::*` label |

Report the result as a short table: check, pass/fail, what you found.

## Step 1 — Ask me everything, once

Now ask me **every open question in a single round**. Do not ask them one at a time
across the session, and do not proceed on an assumption where a question would do.

Base your questions on what preflight actually found. Likely candidates:

- Any config value that is missing or looks wrong
- Which 15 closed issues to use for calibration — propose a list from recently closed
  issues with substantial discussion, and let me confirm or replace it
- Whether missing labels should be created by you or by me in the GitLab UI
- How to handle issues that already carry a triage label: skip, or re-rate
- Anything in the architecture doc that is ambiguous enough to affect ratings —
  e.g. two modules that look like they own the same concern
- Whether the data-handling sign-off for sending issue content to the Copilot backend
  has happened (see the pitfalls table in the runbook)

If preflight passed cleanly and nothing is ambiguous, say so explicitly and ask only
for a go-ahead. Do not invent questions to fill the step.

Wait for my answers. Then restate the plan in five lines or fewer, including how many
issues you will process and roughly how long it will take.

---

## Step 2 — Calibration  🚧 GATE

Run `ai-triage-analyze.prompt.md` against **only** the confirmed calibration issues
(closed issues, known effort). Output goes to `calibration-results.csv`.

Then present, in chat:

- the rating distribution
- for each issue: your rating, plus what you can infer about actual effort from the
  closed issue's history (MR size, number of review rounds, time from start to merge)
- **your own read on where the rubric misfired** — do not just hand me the table

**GATE**: stop. I will tell you which ratings were wrong and why. Propose concrete
edits to the rubric in `ai-triage-analyze.prompt.md`, apply them once I approve, and
only then continue.

---

## Step 3 — Full dry run  🚧 GATE

Run `ai-triage-analyze.prompt.md` over the full backlog per `issue_filter` in the
config. Batches of 15, appending to `triage-results.csv` after each batch, one progress
line per batch.

When done, report:

- rating distribution with percentages — flag it yourself if `high` exceeds 25 %
- the `needs_prep` count, grouped by recurring `prep_action` themes
- top 5 `high` candidates, one line each
- every issue where your confidence was `low`

**GATE**: stop. I will review and correct `triage-results.csv` by hand. Do not proceed
until I confirm the CSV is final.

---

## Step 4 — Write-back  🚧 GATE

Run `ai-triage-writeback.prompt.md`. It reads the corrected CSV and does not re-rate.

First run it with `dry_run: true` from the config and show me the rendered comments for
three sample issues. **GATE**: wait for approval, then set `dry_run: false` and write
for real, in batches of 10 with progress lines.

On any MCP error: stop immediately and report. Do not retry blindly — partial writes
are painful to unwind.

---

## Step 5 — Handover

Produce `triage-report.md` in the repo containing:

- what was written, and to how many issues
- the rating distribution
- the `needs_prep` findings as a prioritised list — this is the most valuable output,
  treat it as such, not as an appendix
- the board filter for sprint planning
- what should change before the next run

Then remind me of the weekly re-run for new issues (runbook, "New issues").

---

## Hard rules

- Never estimate hours, story points, or a percentage of time saved. You have no
  calibration data for this codebase; any such number would be fabricated. Real numbers
  come from the sprint retro loop described in the runbook.
- Never rate an issue without having searched the codebase for the files it touches.
- If you are unsure whether something counts as a gate, treat it as a gate.
- If I say "GO" again mid-run, that resumes the current step. It does not skip a gate.
