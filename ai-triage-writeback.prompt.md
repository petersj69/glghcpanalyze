---
mode: agent
description: Write reviewed AI-triage results back to GitLab as comments and labels
---

# Phase 3 — Write-back

Read the triage CSV and write results back to GitLab: one comment and one scoped label
per issue. **Do not re-evaluate any issue.** The CSV has been reviewed by a human and
is the source of truth.

Normally invoked by `ai-triage.prompt.md`. Can be run standalone.

## Configuration

Read `docs/ai-triage/triage-config.yml`. Use `project`, `output_csv`, `write_ratings`,
`write_needs_prep_label`, `writeback_batch_size`, `dry_run`, `labels`.

## Procedure

1. Read `output_csv`. Filter to rows whose `rating` is in `write_ratings`.
2. Print the number of issues about to be modified and **wait for confirmation** before
   the first write.
3. Per issue, in this order:
   a. Fetch existing comments. If any contains the marker `<!-- ai-triage -->`,
      **update** that comment instead of adding a new one. Never post a second one.
   b. Post or update the comment using the template below.
   c. Set the label from `labels[rating]`. GitLab's scoped labels remove any previous
      `AI Support possible::*` automatically — do not remove it manually.
   d. If `needs_prep = yes` and `write_needs_prep_label` is true, additionally set
      `labels.needs_prep`.
4. Process in batches of `writeback_batch_size` and print progress after each. On any
   MCP error, stop and report — do not retry blindly, partial writes are hard to unwind.
5. At the end, list every issue written, and separately any that failed.

## Comment template

```markdown
<!-- ai-triage -->
**AI support assessment: {rating}** · confidence: {confidence}

{rationale}

**Suggested approach:** {recommendation}

| | |
|---|---|
| Pattern | {pattern} |
| Likely files | {files_touched} |
| Test coverage | {test_coverage} |

{prep_block}

<sub>Automated assessment generated with GitHub Copilot. It is a starting point, not a
verdict — if you disagree, change the label and note why. That feedback improves the
next run.</sub>
```

`{prep_block}` is included only if `needs_prep = yes`:

```markdown
> **Prep needed first:** {prep_action}
> Doing this would likely move the issue up one rating.
```

Otherwise `{prep_block}` is empty — no placeholder text.

## Rules

- English, no emoji.
- Keep the comment exactly as templated. Do not add commentary, do not expand the
  rationale, do not soften it.
- If a CSV field is empty, drop the corresponding table row rather than writing "n/a".
- If `dry_run: true`, print each comment and label to the console instead of writing.
  Always do one dry pass before the first real run against a project.
