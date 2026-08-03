---
mode: agent
description: Rate open GitLab issues for Copilot suitability (read-only, no writes)
---

# Phase 2 — Analyse

Rate how suitable each open issue is for being worked on with GitHub Copilot's help.
**Write nothing back to GitLab in this run.** The only output is a local CSV plus a
short summary.

Normally invoked by `ai-triage.prompt.md`. Can be run standalone.

## Configuration

Read `docs/ai-triage/triage-config.yml`. Use `project`, `issue_filter`, `batch_size`,
`output_csv`, `arch_doc`, `extra_context`, `existing_triage_label`.

If invoked for calibration, process only the IIDs in `calibration_issues` and write to
`calibration_csv` instead.

## Procedure

1. Read `arch_doc`, everything under `extra_context`, and
   `.github/copilot-instructions.md` first. Every rating must be consistent with the
   layering and testing rules described there.
2. Fetch issues matching `issue_filter` via the GitLab MCP, ordered by `updated_at`
   descending.
3. If `existing_triage_label: skip`, drop issues that already carry an
   `AI Support possible::*` label.
4. Process in batches of `batch_size`. After **each** batch, append rows to
   `output_csv` and print one progress line (`batch 3/9 done, 45 issues rated`). Never
   hold more than one batch in working context.
5. If `output_csv` already exists, read it first and skip issues already present. This
   makes the run resumable after an interruption.

## Per issue

Do this before rating — a rating without step (b) is invalid:

a. Read title, description, labels, and the last 3 comments.
b. **Search the codebase** for the modules the issue would touch. Record the actual
   file paths. If you cannot identify any concrete file, the rating is capped at `low`
   and `confidence` is `low`.
c. Check whether the touched area has existing test coverage.
d. Apply the rubric below.

## Rubric

Rate `high` only if **all** of these hold:

- The desired outcome is unambiguous — you could write the acceptance criteria yourself
  from the issue text without asking anyone
- Touches roughly 5 files or fewer, within one module
- The touched area has existing tests, or the change is trivially testable
- An established pattern for this kind of change already exists in the codebase
- No product, security, or architecture decision is required
- No knowledge is needed that lives only in someone's head

`med` — mostly clear, but exactly one of the above fails, and the gap is closeable in
under 30 minutes of human prep. Copilot can produce a usable first draft that needs
substantive review.

`low` — spans several modules, requires a design decision, is exploratory ("investigate
why…"), or the touched area has no test coverage at all.

`none` — not primarily a code task, blocked on stakeholder alignment, a security or
compliance design question, a duplicate, or stale (no activity > 12 months and no
milestone).

### Anti-inflation rules

- If you are hesitating between two ratings, take the lower one.
- Do not rate `high` because the issue *sounds* simple. Rate on what the code shows.
- A one-line issue description with no acceptance criteria is never `high`, however
  small the change appears. It is `med` at best, with `needs_prep = yes`.
- Expect roughly 10–20 % `high` across a normal backlog. If a batch runs well above
  `expected_high_share_max`, re-check it against these rules before writing the CSV.

## Output schema

CSV with header, one row per issue, `;` separated, UTF-8:

| Column | Content |
|---|---|
| `iid` | Issue IID |
| `title` | Issue title, truncated to 80 chars |
| `rating` | `none` \| `low` \| `med` \| `high` |
| `confidence` | `low` \| `med` \| `high` — your confidence in the rating itself |
| `pattern` | One of: `bugfix-with-repro`, `crud-endpoint`, `refactoring`, `test-gap`, `dependency-bump`, `config`, `docs`, `investigation`, `design`, `other` |
| `files_touched` | Comma-separated real paths from step (b), max 5. Empty only if rating is `low`/`none` |
| `test_coverage` | `yes` \| `partial` \| `no` \| `unknown` |
| `rationale` | Max 2 sentences, English. Why this rating |
| `needs_prep` | `yes` \| `no` — would better issue quality raise the rating? |
| `prep_action` | If `needs_prep = yes`: the single concrete thing that would raise it. Else empty |
| `recommendation` | Max 1 sentence: how to approach it with Copilot, or why not to |

## Final summary

After the last batch, print:

- count per rating, plus percentage
- count of `needs_prep = yes`, grouped by the most frequent `prep_action` themes
- the 5 strongest `high` candidates with a one-line reason each
- any issue where you were genuinely unsure — flag these for human review

Do not estimate hours, story points, or a percentage of time saved. You have no
calibration data for this codebase and any such number would be fabricated.
