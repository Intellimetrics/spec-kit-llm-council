---
description: "Run the council against the active feature's current .specify/ artifacts at any time, outside the standard lifecycle gates"
tools:
  - 'llm-council/council_run'
  - 'llm-council/council_estimate'
  - 'llm-council/council_doctor'
---

# LLM Council Audit

Convene the council against **whatever artifacts currently exist** for the active feature. Unlike `plan-review`, `implement-review`, and `spec-review`, this command is not bound to a lifecycle gate — invoke it any time you want a sanity check on the current state of `.specify/`.

The council's question adapts to what's present: if you only have a spec, it reviews the spec; if spec + plan, it reviews plan-spec consistency; if spec + plan + tasks, it reviews full traceability.

**The verdict is advisory only.** Audits can be run repeatedly; each writes its own timestamped evidence file.

## User Input

```text
$ARGUMENTS
```

If the user passed an explicit slug as `$ARGUMENTS`, treat that slug as the feature.

## Prerequisites

- `llm-council` available via MCP server or CLI on `PATH`.
- An active feature directory under `specs/` containing **at least** `spec.md`. If even `spec.md` is missing, abort: `"No spec.md in specs/<feature>/ — run /speckit.specify first; nothing to audit."`

## Step 1 — Resolve the active feature

Same algorithm as `plan-review` Step 1.

## Step 2 — Detect available artifacts

Probe the feature directory for which SDD artifacts exist:

| Artifact | Path | Required for audit? |
|---|---|---|
| `spec.md` | `specs/<feature>/spec.md` | Yes (abort if missing) |
| `plan.md` | `specs/<feature>/plan.md` | Optional |
| `tasks.md` | `specs/<feature>/tasks.md` | Optional |
| Constitution | `.specify/memory/constitution.md` | Optional |

Record which artifacts are present; this drives the question construction in Step 4.

## Step 3 — Load configuration

Read `.specify/extensions/llm-council/llm-council-config.yml`. Use the same defaults as `plan-review`. The `inputs` config is **not** consulted directly here — audit decides inputs based on what's present in the feature dir, not based on a fixed list.

## Step 4 — Construct the audit question

Tailor the council question to the artifacts available:

**spec.md only** (very early in the SDD lifecycle):

> Review the attached spec. Identify acceptance-criteria gaps, hidden assumptions, scope ambiguity, and wrong-problem framing. The plan and task list have not been generated yet — focus on whether this spec is ready for `/speckit.plan` to operate on it. End with `RECOMMENDATION: yes / no / tradeoff`.

**spec.md + plan.md** (post-plan, pre-tasks):

> Review the attached plan against the spec and constitution. Identify design flaws, requirements drift between spec and plan, missing acceptance criteria, and risks worth flagging before task decomposition. End with `RECOMMENDATION: yes / no / tradeoff`.

**spec.md + plan.md + tasks.md** (post-tasks, full traceability check):

> Review the attached spec, plan, and task list as a coherent set. Identify (a) tasks that don't trace back to a stated requirement in the spec or plan, (b) requirements without corresponding tasks, (c) plan elements not reflected in tasks, (d) sequencing or dependency risks, (e) constitution violations baked into any artifact. End with `RECOMMENDATION: yes / no / tradeoff`.

If `$ARGUMENTS` is non-empty, prepend `Reviewer focus: <args>` to whichever question is selected.

Always include the constitution as context if present.

## Step 5 — Pre-flight cost estimate

Same as `plan-review` Step 4. Refuse non-silently if cap exceeded.

## Step 6 — Convene the council

Same as `plan-review` Step 5. Capture transcript, label, per-peer responses. Record `council_label: degraded` on provider failure or sub-quorum.

## Step 7 — Write the evidence artifact

Audit can be run repeatedly, so the evidence file is **timestamped** (unlike the hook commands which write to fixed paths and overwrite).

Path: `.specify/council/<feature>/audit-<YYYYMMDD-HHMMSS>.md`

Format:

```markdown
---
feature: <feature>
gate: audit
council_label: yes | no | tradeoff | degraded
quorum: <int>
participants: [claude, codex, gemini, ...]
mode: <mode>
artifacts_reviewed: [spec.md, plan.md]   # whatever was actually included
transcript: <path under .llm-council/runs/>
extension_version: <extension_version>
timestamp: <ISO-8601 UTC>
---

# Council Audit — <feature>

**Verdict:** <one-sentence summary>

**Artifacts reviewed:** spec.md, plan.md (and constitution)

## Findings
- ...

## Cross-artifact traceability
- ...

## Recommendations
- ...

> Full transcript: <transcript path>
```

The body sections adapt to what was reviewed; for a spec-only audit, drop "cross-artifact traceability" and emphasize spec-quality findings.

## Step 8 — Surface to the user

```
[council] audit for <feature> — verdict: <label> (<quorum>/<peers>)
[council] Reviewed: <comma-separated artifact list>
[council] <one-line summary>
[council] Evidence: <evidence path>
```

The `last` command will find this audit (it sorts evidence files by mtime within the feature dir and returns the newest), so users can recall the audit verdict the same way they recall hook verdicts.

## Failure modes

| Condition | Action |
|---|---|
| No `spec.md` in active feature | Abort, point user at `/speckit.specify` |
| `llm-council` not installed | Abort, link install docs |
| Pre-flight cost > cap | Prompt user, never silently override |
| Provider failure / quorum not reached | Write `council_label: degraded`, surface error |
| Multiple feature dirs (ambiguous) | Same fail-closed message as other commands; require explicit slug |
