---
description: "Convene the council after /speckit.specify to review the spec for acceptance criteria, hidden assumptions, and right-problem framing"
tools:
  - 'llm-council/council_run'
  - 'llm-council/council_estimate'
  - 'llm-council/council_doctor'
---

# LLM Council Spec Review

Convene a multi-LLM consensus jury to review `spec.md` for the active feature **before** any planning begins. Catches the upstream poison case: a flawed spec compounds through plan + tasks + code. By the time the existing `before_tasks` gate fires, the bad framing is already inherited.

**The verdict is advisory only — it does not block `/speckit.plan`.** Same advisory invariant as the other review commands.

## User Input

```text
$ARGUMENTS
```

If the user passed an explicit slug as `$ARGUMENTS`, treat that slug as the feature.

## Prerequisites

- `llm-council` available via MCP server or CLI on `PATH`. Abort and link install docs if neither is present.
- An active feature directory under `specs/` containing `spec.md`. If `spec.md` is absent, abort: `"No spec.md in specs/<feature>/ — run /speckit.specify first."`

## Step 1 — Resolve the active feature

Same algorithm as `plan-review` Step 1. Strip common branch prefixes, match against `specs/<slug>/`, fail closed on ambiguity, accept explicit slug as `$ARGUMENTS` override.

## Step 2 — Load configuration

Read `.specify/extensions/llm-council/llm-council-config.yml`. Use the same defaults as `plan-review`. For this gate, the only required input is `spec.md`; the constitution is included if present. `plan.md` and `tasks.md` typically do not exist yet at this point — skip them silently if absent.

## Step 3 — Build the prompt

Council question:

> Review the attached spec. Identify (a) missing or vague acceptance criteria, (b) hidden assumptions that should be made explicit, (c) scope ambiguity (what's in vs. out, what's a non-goal), (d) "wrong-problem" framing — solution statements masquerading as requirements, (e) constitution-alignment gaps where the spec implicitly violates a stated project principle. End your response with `RECOMMENDATION: yes`, `RECOMMENDATION: no`, or `RECOMMENDATION: tradeoff`.

If `$ARGUMENTS` is non-empty, prepend `Reviewer focus: <args>`.

Resolve inputs:
- `spec.md` (always; relative to feature dir)
- `.specify/memory/constitution.md` (if present; from repo root)
- Skip `plan.md` and `tasks.md` (they don't exist at this gate)

Specs are typically short — a few hundred to a few thousand chars — so the prompt is cheap. This is one of the gates where token spend is most easily justified.

## Step 4 — Pre-flight cost estimate

Same as `plan-review` Step 4. Use `llm-council estimate --max-cost-usd <cap> --json` (CLI fallback) or `council_estimate` (MCP). Refuse non-silently if cap exceeded.

## Step 5 — Convene the council

Same as `plan-review` Step 5. Capture transcript path, recommendation label, per-peer responses. On provider failure or sub-quorum, record `council_label: degraded`.

## Step 6 — Write the evidence artifact

Write to `.specify/council/<feature>/spec-review.md` (sibling to `plan-review.md` and `implement-review.md`). Substitute `{feature}` and `<extension_version>` (read from `.specify/extensions/llm-council/extension.yml`).

Format:

```markdown
---
feature: <feature>
gate: after_specify
council_label: yes | no | tradeoff | degraded
quorum: <int>
participants: [claude, codex, gemini, ...]
mode: <mode>
transcript: <path under .llm-council/runs/>
extension_version: <extension_version>
timestamp: <ISO-8601 UTC>
---

# Council Spec Review — <feature>

**Verdict:** <one-sentence summary>

## Acceptance criteria gaps
- ...

## Hidden assumptions
- ...

## Scope ambiguity
- ...

## Wrong-problem framing
- ...

## Constitution alignment
- ...

> Full transcript: <transcript path>
```

Aggregate per-peer responses. If a section has no flagged items, write `- _(none flagged)_` rather than omitting the heading — the structure makes it clear what dimensions were considered.

## Step 7 — Surface to the user

```
[council] after_specify gate — verdict: <label> (<quorum>/<peers>)
[council] <one-line summary>
[council] Evidence: <evidence path>
```

If the label is `no` or `tradeoff`, append:

```
[council] Verdict is advisory — proceed to /speckit.plan anyway? [Y/n]
```

## Failure modes

Same as `plan-review`. The missing-prerequisite case is `spec.md` (point user at `/speckit.specify`).
