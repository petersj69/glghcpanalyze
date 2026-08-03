# Runbook: AI triage of the GitLab backlog with GitHub Copilot

Goal: every open issue gets an `AI Support possible::none|low|med|high` label plus a
comment with the reasoning and a recommendation. In sprint planning, every person
picks up at least one `::high` item.

Three phases. Phase 1 writes **nothing** back to GitLab.

---

## How to run it

Put the files in the repo like this:

```
.github/
  copilot-instructions.md          # your existing repo context
  prompts/
    ai-triage.prompt.md            # orchestrator — this is the one you invoke
    ai-triage-analyze.prompt.md    # phase 2, read-only
    ai-triage-writeback.prompt.md  # phase 3, writes to GitLab
docs/
  ai-triage/
    RUNBOOK-ai-triage.md           # this file
    triage-config.yml              # fill this in first
```

Then, in VS Code with Copilot in **agent mode**:

```
/ai-triage
GO
```

The orchestrator runs preflight, asks you everything it needs in one round, and then
walks the phases. It stops at four gates and waits for you — it will not write to
GitLab on its own.

Fill in `triage-config.yml` before the first run. Anything left as `<...>` fails
preflight, which is intentional: the alternative is Copilot quietly triaging the wrong
project.

If this folder lives in a public repo, don't fill in the tracked file: copy it to
`triage-config.local.yml` next to it and fill that in instead. The prompts prefer the
`.local` copy when it exists, and it is gitignored — your GitLab host and project path
never get committed. The generated CSVs and `triage-report.md` are gitignored for the
same reason: they contain your issue titles.

The rest of this document explains *why* the process is shaped this way. You do not
need to read it to run the triage, but you do need it to interpret the results.

---

## Setup (one-off, ~30 min)

### 1. Create the scoped labels in GitLab

Create them at **group** level, not per project — then they apply across all repos.
The double colon matters: GitLab treats `Key::Value` as a *scoped label* and
automatically removes the previous one when a new one is set. Without it, an issue
accumulates multiple ratings across runs.

| Label | Suggested colour |
|---|---|
| `AI Support possible::high` | `#2da160` |
| `AI Support possible::med` | `#c9a227` |
| `AI Support possible::low` | `#d99530` |
| `AI Support possible::none` | `#8c8c8c` |

Plus one more, recommended:

| Label | Purpose |
|---|---|
| `AI Support possible::needs-prep` | Would be high/med, but issue quality isn't there yet. This is the real finding. |

### 2. Give Copilot the context

Create `.github/copilot-instructions.md` (if you don't have one) covering:

- path to the architecture doc (markdown)
- layering / module rules that must not be violated
- test strategy: where tests live, what's mandatory for an MR
- your team's definition of done

Without this, Copilot rates issues against a generic idea of your codebase and
produces far too many `::high`.

### 3. Verify the GitLab MCP

Tools needed: list issues, read issue, create comment, set labels. Check the actual
parameter names of your MCP server first — they often differ from the GitLab REST API.
One test call against a single issue saves an aborted batch run later.

---

## Phase 1 — Calibration (~45 min)

**Don't skip this.** Without it you don't know whether `::high` actually means `::high`.

1. Pick 15 **already closed** issues whose real effort you know.
2. Put their IIDs in `calibration_issues` in the config — or leave it empty and let the
   orchestrator propose a set for you to confirm.
3. Compare as a team: where was the rating off? Usually in one of two directions:
   - too optimistic on issues carrying hidden domain knowledge
   - too pessimistic on routine refactorings
4. Adjust the rubric in `ai-triage-analyze.prompt.md` — typically one or two extra
   exclusion criteria under `none`. The orchestrator will propose the edits; you approve
   them.

---

## Phase 2 — Dry run across the backlog (~1–2 h for 50–200 issues)

Prompt: **`ai-triage-analyze.prompt.md`**

Runs in batches of `batch_size`. `triage-results.csv` is appended after each batch — if the run
breaks, you resume from the last completed batch.

Check the output for:

- **Share of `::high`**: 10–20 % is realistic. Above 35 % almost always means the
  rubric is too loose, or Copilot didn't actually look at the code.
- **Spot check**: read 5 random `::high` yourself. Do the listed files exist? If
  Copilot names paths that aren't real, it guessed — tighten the prompt and rerun.
- **`needs-prep` clustering**: this is your actual result. If 40 % of issues fail to
  qualify only because acceptance criteria are missing, that's a bigger lever than any
  Copilot feature.

---

## Phase 3 — Write back to GitLab (~30 min)

Prompt: **`ai-triage-writeback.prompt.md`**

Reads `triage-results.csv` (after your manual corrections) and writes comment + label.
It does not re-evaluate anything — that's what stops your corrections being overwritten.

For the first pass, keep `write_ratings: [high, med]`. `::none` creates a lot of noise on
issues nobody will touch anyway.

---

## Operating it: sprint planning

**Rule**: every person takes at least one `::high` item per sprint.

Filter for the planning board:

```
label=AI Support possible::high&state=opened&assignee_id=None
```

Then a short retro item, two minutes, three questions:

- Did Copilot support actually save time?
- How much extra review effort did it add?
- Was the `::high` rating justified?

That's your calibration loop — after about 3 sprints you have real numbers for the
efficiency gain instead of a model's guess.

### New issues

Weekly run over issues carrying no `AI Support possible::*` label:

```
state=opened&not[labels]=AI Support possible::high,AI Support possible::med,AI Support possible::low,AI Support possible::none
```

Then straight to phase 2 + 3 — no calibration needed.

---

## Known pitfalls

| Problem | Mitigation |
|---|---|
| Copilot rates without looking at the code | Prompt requires `files_touched`; no concrete paths means `::med` at most |
| Context window collapses on a large backlog | Batches of 15, intermediate state in CSV |
| Repeat runs post duplicate comments | `<!-- ai-triage -->` marker in the comment, checked before posting |
| `::high` becomes self-reinforcing ("the AI said it's easy, so it's easy") | Manual spot check, retro question each sprint |
| Review effort goes unmeasured | Ask about it explicitly in the retro, otherwise the numbers look too good |
| Issue content is sent to the Copilot backend | Get written confirmation of your enterprise tenant's data handling before the run |
