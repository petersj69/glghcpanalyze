# AI Triage for GitLab Backlogs

A prompt toolkit that rates every open GitLab issue for how well GitHub Copilot can
help with it — `AI Support possible::none|low|med|high` — and writes the rating back
as a scoped label plus a reasoned comment. Runs in VS Code with Copilot in **agent
mode**, talking to GitLab through an MCP server. No code, no pipeline: just prompts,
a config file, and a runbook.

The process is deliberately gated: nothing is written to GitLab without your explicit
approval, and every rating must be backed by real file paths found in your codebase.

## What's in here

| File | Role |
|---|---|
| [`ai-triage.prompt.md`](ai-triage.prompt.md) | Orchestrator — the prompt you invoke. Walks all phases, stops at four approval gates |
| [`ai-triage-analyze.prompt.md`](ai-triage-analyze.prompt.md) | Phase 2: rates issues against a strict rubric, read-only, outputs a CSV |
| [`ai-triage-writeback.prompt.md`](ai-triage-writeback.prompt.md) | Phase 3: writes the human-reviewed CSV back as comments + labels |
| [`triage-config.yml`](triage-config.yml) | All configuration — fill this in first |
| [`RUNBOOK-ai-triage.md`](RUNBOOK-ai-triage.md) | The full process: setup, calibration, pitfalls, sprint-planning loop |

## How to use it

1. **Copy the files into your target repo** in the layout the orchestrator expects:

   ```
   .github/prompts/ai-triage.prompt.md
   .github/prompts/ai-triage-analyze.prompt.md
   .github/prompts/ai-triage-writeback.prompt.md
   docs/ai-triage/RUNBOOK-ai-triage.md
   docs/ai-triage/triage-config.yml
   ```

2. **One-off setup** (~30 min, details in the [runbook](RUNBOOK-ai-triage.md)):
   create the five `AI Support possible::*` scoped labels at GitLab **group** level,
   make sure `.github/copilot-instructions.md` describes your architecture and test
   rules, and verify your GitLab MCP server can list issues, comment, and set labels.

3. **Fill in `triage-config.yml`.** Any value still containing `<...>` fails
   preflight on purpose.

4. **Run it.** In VS Code, with Copilot in agent mode:

   ```
   /ai-triage
   GO
   ```

   The orchestrator runs preflight, asks everything it needs in one round, then walks
   the phases — calibration on closed issues, a dry run over the backlog, and only
   after your review of the CSV, the write-back.

5. **Use the results in sprint planning.** Filter the board for
   `AI Support possible::high` and unassigned; each person picks at least one. A
   two-minute retro question per sprint ("did it actually save time?") is what turns
   the ratings into real numbers.

## Good to know

- Expect 10–20 % `::high` on a normal backlog. Much more usually means the rubric is
  too loose or the model didn't look at the code.
- The `::needs-prep` findings are the most valuable output: issues that would be
  AI-suitable if they had acceptance criteria. Fixing that is a bigger lever than any
  tooling.
- Issue content is sent to the Copilot backend — check your tenant's data-handling
  terms before the first run.
