---
description: "Convene the council before /speckit.implement to review whether the generated tasks faithfully decompose the plan"
tools:
  - 'llm-council/council_run'
  - 'llm-council/council_estimate'
  - 'llm-council/council_doctor'
---

# LLM Council Implement Review

Convene a multi-LLM consensus jury to review the active feature's `tasks.md` against `plan.md` (and spec/constitution) before code generation. Catches design-to-tasks drift: missing tasks for stated requirements, phantom tasks that don't trace back to the plan, hidden complexity that should have been broken down further.

**The verdict is advisory only — it does not block `/speckit.implement`.** Same advisory invariant as `plan-review`.

## User Input

```text
$ARGUMENTS
```

If the user passed an explicit slug as `$ARGUMENTS`, treat that slug as the feature.

## Prerequisites

- Same as `plan-review`: `llm-council` available via MCP or CLI.
- An active feature with `plan.md` AND `tasks.md` produced by `/speckit.plan` and `/speckit.tasks`. If `tasks.md` is absent, abort: `"No tasks.md in specs/<feature>/ — run /speckit.tasks first."`

## Step 1 — Resolve the active feature

Same algorithm as `plan-review` Step 1. Abort with the same messages on ambiguity.

## Step 2 — Load configuration

Read `.specify/extensions/llm-council/llm-council-config.yml`. Use the same defaults as `plan-review`. The `inputs` list is augmented for this gate: append `tasks.md` (always included whether listed or not, same way `plan.md` is always included for `plan-review`).

## Step 3 — Build the prompt

Council question:

> Review the attached tasks against the plan, spec, and constitution. Identify (a) tasks that don't trace back to a stated requirement in the plan, (b) plan requirements with no corresponding task, (c) tasks that hide significant complexity and should be split, (d) sequencing risks (tasks that depend on each other but aren't ordered), and (e) constitution violations (e.g., test-first principles, simplicity rules) baked into the task list. End your response with `RECOMMENDATION: yes`, `RECOMMENDATION: no`, or `RECOMMENDATION: tradeoff`.

If `$ARGUMENTS` is non-empty, prepend `Reviewer focus: <args>` to the question.

Resolve inputs:
- `tasks.md` (relative to feature dir; always included)
- `plan.md` (relative to feature dir; always included)
- Other `inputs` from config (typically `spec.md` and `.specify/memory/constitution.md`)

## Step 4 — Pre-flight cost estimate

Same as `plan-review` Step 4. Use `llm-council estimate --max-cost-usd <cap> --json` (CLI fallback) or `council_estimate` (MCP). If estimate exceeds cap, surface the breakdown to the user and ask whether to proceed, raise the cap for this run only, or skip. Do not silently override.

## Step 5 — Convene the council

Same as `plan-review` Step 5. Capture transcript path, recommendation label, per-peer responses.

If the run fails (provider down, no API keys, quorum below `min_quorum`), record a degraded artifact with `council_label: degraded` and the failure reason.

## Step 6 — Write the evidence artifact

Write to `.specify/council/<feature>/implement-review.md` (sibling to `plan-review.md`). Substitute `{feature}` and `<extension_version>` (read from `.specify/extensions/llm-council/extension.yml` `extension.version`).

Format:

```markdown
---
feature: <feature>
gate: before_implement
council_label: yes | no | tradeoff | degraded
quorum: <int>
participants: [claude, codex, gemini, ...]
mode: <mode>
transcript: <path under .llm-council/runs/>
extension_version: <extension_version>
timestamp: <ISO-8601 UTC>
---

# Council Implement Review — <feature>

**Verdict:** <one-sentence summary>

## Tasks → plan traceability
- ...

## Missing or hidden complexity
- ...

## Sequencing / dependency risks
- ...

## Constitution compliance
- ...

> Full transcript: <transcript path>
```

Aggregate per-peer responses; do not paste verbatim.

## Step 7 — Surface to the user

```
[council] before_implement gate — verdict: <label> (<quorum>/<peers>)
[council] <one-line summary>
[council] Evidence: <evidence path>
```

If the label is `no` or `tradeoff`, append:

```
[council] Verdict is advisory — proceed to /speckit.implement anyway? [Y/n]
```

## Failure modes

Same as `plan-review`, but the missing-prerequisite case is `tasks.md` (point user at `/speckit.tasks`) rather than `plan.md`.
