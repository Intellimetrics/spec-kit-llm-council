---
description: "Print the last recorded council verdict for the active feature without re-convening the jury"
---

# LLM Council — Last Verdict

Print the most recent council verdict for the active feature, read from `.specify/council/<feature>/`. **No subprocess calls. No tokens spent.** This is a pure read-only recall affordance.

## User Input

```text
$ARGUMENTS
```

If the user passed an explicit slug as `$ARGUMENTS` (e.g., the invocation ends with ` 003-user-auth`), treat that slug as the feature and skip directly to step 3.

## Step 1 — Resolve the active feature

Use the same algorithm as `speckit.llm-council.plan-review`:

1. Get the current git branch: `git rev-parse --abbrev-ref HEAD`.
2. Strip common workflow prefixes: `feat/`, `feature/`, `fix/`, `bugfix/`, `hotfix/`, `chore/`, `refactor/`, `docs/`. The remainder is the candidate slug.
3. If `specs/<slug>/` exists, use it.
4. If no match, list directories under `specs/`. If exactly one exists, use it.
5. Otherwise, abort with: `"Cannot resolve active feature from branch '<branch>'. Found <N> feature directories under specs/. Re-invoke this command with the feature slug as an argument (use whatever invocation syntax your spec-kit integration uses) or check out a feature branch."`

Let `<feature>` be the basename of the resolved directory.

## Step 2 — Locate the evidence file(s)

Look under `.specify/council/<feature>/` for evidence artifacts:

- `plan-review.md` (from the `before_tasks` gate; produced by `speckit.llm-council.plan-review`)
- Any future `<gate>-review.md` files (none in v0.2.0; designed for v0.3.0+ hooks)

If the directory does not exist, abort with: `"No council verdicts recorded for <feature>. Run the plan-review command to convene the council."`

If the directory exists but is empty, the same abort message applies.

If multiple files exist, sort by file modification time (newest first) and treat the newest as "the last verdict." Mention the count in the output if more than one is present.

## Step 3 — Read and summarize

Parse the YAML frontmatter of the chosen evidence file. Extract:

- `feature`
- `gate`
- `council_label` (`yes` / `no` / `tradeoff` / `degraded`)
- `quorum`
- `participants` (count for the inline summary)
- `mode`
- `transcript`
- `timestamp`

Then read the body for the one-sentence verdict (the line beginning with `**Verdict:**`).

## Step 4 — Stale-verdict check

Compare the modification time of the evidence file against the modification time of the inputs that informed it:

- `specs/<feature>/plan.md`
- `specs/<feature>/spec.md` (if present)
- `.specify/memory/constitution.md` (if present)

If any input file is **newer** than the evidence file, set a `stale: true` flag for the output. (Best-effort filesystem mtime; do not fail if files are missing.)

## Step 5 — Print the summary

Format:

```
[council] last verdict for <feature> — <council_label> (<quorum>/<participants>)
[council] <one-sentence verdict from the body>
[council] Recorded: <timestamp> (gate: <gate>, mode: <mode>)
[council] Evidence: .specify/council/<feature>/<filename>
[council] Transcript: <transcript>
```

If `stale: true`, append:

```
[council] ⚠ plan.md or spec.md has changed since this verdict was recorded — consider re-running the council.
```

If multiple evidence files exist, append:

```
[council] (<N> verdicts on file for this feature; this is the most recent.)
```

## Failure modes

| Condition | Action |
|---|---|
| No `.specify/council/<feature>/` directory | Abort with "no verdicts recorded; run plan-review first" |
| Evidence file present but malformed YAML | Warn, print whatever can be salvaged, link to file |
| Active feature unresolvable | Abort with the same message as `plan-review` |
