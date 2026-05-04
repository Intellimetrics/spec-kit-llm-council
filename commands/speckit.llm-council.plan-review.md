---
description: "Convene a multi-LLM jury to review the active feature's plan.md and record an advisory verdict"
tools:
  - 'llm-council/council_run'
  - 'llm-council/council_estimate'
---

# LLM Council Plan Review

Convene a multi-LLM consensus jury to review `plan.md` for the active feature and record an advisory verdict in `.specify/council/<feature>/plan-review.md`.

**The verdict is advisory only — it does not block `/speckit.tasks`.** Surface the label and a one-line summary to the user before they proceed.

## User Input

```text
$ARGUMENTS
```

If the user supplied additional context (e.g., "focus on determinism risks", "skip the cost cap"), pass it through to the council as a `Reviewer focus:` prefix on the question.

## Prerequisites

- `llm-council` available via either (a) the `llm-council` MCP server registered in the host agent's `.mcp.json`, or (b) the `llm-council` CLI on `PATH`. If neither is present, abort and tell the user to install per https://github.com/Intellimetrics/llm-council.
- An active feature directory under `specs/` containing `plan.md`. If no `plan.md` exists, abort: "No plan.md found for active feature — run /speckit.plan first."

## Step 1 — Resolve the active feature

Determine the active feature directory **deterministically**. Fail closed if the heuristic can't unambiguously identify one — never silently default to "most recent" (that has burned users by reviewing the wrong plan).

If the user passed an explicit slug as `$ARGUMENTS` (e.g., the invocation ends with ` 003-user-auth`), treat that slug as the feature and skip directly to step 6.

Otherwise:

1. Get the current git branch: `git rev-parse --abbrev-ref HEAD`.
2. Strip common workflow prefixes from the branch name: `feat/`, `feature/`, `fix/`, `bugfix/`, `hotfix/`, `chore/`, `refactor/`, `docs/`. The remainder is the candidate slug. Example: `feat/003-user-auth` → `003-user-auth`; `bugfix/auth-token-rotation` → `auth-token-rotation`.
3. If `specs/<slug>/` exists, use it.
4. If no match, list directories under `specs/`. If exactly one exists (single-feature project), use it.
5. Otherwise, abort with: `"Cannot resolve active feature from branch '<branch>'. Found <N> feature directories under specs/. Re-invoke this command with the feature slug as an argument (use whatever invocation syntax your spec-kit integration uses — e.g. ending with ' 003-user-auth') or check out a feature branch."`
6. Verify `plan.md` exists in the resolved directory. If not, abort: `"No plan.md in specs/<feature>/ — run /speckit.plan first."`

Let `<feature>` be the basename of the resolved directory (e.g., `003-user-auth`).

## Step 2 — Load configuration

Read `.specify/extensions/llm-council/llm-council-config.yml`. If the file is absent or malformed, fall back to these defaults and warn the user once:

| Key | Default |
|---|---|
| `mode` | `plan` |
| `max_cost_usd` | `0.50` |
| `participants` | `[]` (use mode defaults) |
| `inputs` | `[spec.md, plan.md, .specify/memory/constitution.md]` |
| `evidence_path` | `.specify/council/{feature}/plan-review.md` |
| `prompt_user` | `true` |

Environment overrides:
- `SPECKIT_COUNCIL_MODE` overrides `mode`
- `SPECKIT_COUNCIL_MAX_COST` overrides `max_cost_usd`

## Step 3 — Build the prompt

Council question:

> Review the attached plan against the spec and constitution. Identify design flaws, missing acceptance criteria, requirements drift, hidden complexity, and risks worth flagging before task decomposition. End your response with `RECOMMENDATION: yes`, `RECOMMENDATION: no`, or `RECOMMENDATION: tradeoff`.

If `$ARGUMENTS` is non-empty, prepend `Reviewer focus: <args>` to the question.

Resolve each entry in `inputs`:
- Paths starting with `.specify/` resolve from repo root.
- Other paths resolve relative to the active feature directory.
- `plan.md` is always included whether listed or not.
- Skip missing files silently; do not fail.

## Step 4 — Pre-flight cost estimate

Estimate cost before convening:

- **MCP path (preferred):** call `council_estimate` with the question, `mode`, and resolved input paths. The MCP tool name follows the host's MCP convention (e.g., `mcp__llm-council__council_estimate` for Claude Code).
- **CLI fallback:**

  ```bash
  llm-council estimate \
    --mode "<mode>" \
    --max-cost-usd "<max_cost_usd>" \
    --context "<plan.md>" \
    [--context "<other input>"]... \
    "<question>"
  ```

If the estimate exceeds `max_cost_usd`, surface the estimate to the user and ask whether to (a) proceed, (b) raise the cap for this run only, or (c) skip. **Do not silently override the cap.**

## Step 5 — Convene the council

- **MCP path (preferred):** call `council_run` with `question`, `mode`, `participants` (omit if empty), `context_files` (the resolved inputs), and `max_cost_usd`. Capture the returned transcript path, recommendation label, and per-peer responses.
- **CLI fallback:**

  ```bash
  llm-council run \
    --mode "<mode>" \
    --context "<plan.md>" \
    [--context "<other input>"]... \
    --max-cost-usd "<max_cost_usd>" \
    --json \
    "<question>"
  ```

  Parse the JSON output to extract: aggregated label (`yes`/`no`/`tradeoff`), quorum, participant list, transcript path under `.llm-council/runs/`.

If the run fails (provider down, no API keys, quorum below `min_quorum`), record a degraded artifact with `council_label: degraded` and the failure reason in the body. **Do not fabricate a verdict.**

## Step 6 — Write the evidence artifact

Substitute `{feature}` in `evidence_path`. Create parent directories as needed.

Before writing, resolve these placeholders in the template below:
- `<feature>` — the resolved feature slug from Step 1
- `<extension_version>` — the value of `extension.version` from `.specify/extensions/llm-council/extension.yml` (read at runtime, do not hardcode)
- `<ISO-8601 UTC>` — the current UTC time in ISO-8601 (e.g., `2026-05-04T05:04:34Z`)
- `peer_labels:` — for each participant in the council run, write one mapping entry. The value is the peer's recommendation label (`"yes"` / `"no"` / `"tradeoff"`) if the peer succeeded and produced a label; otherwise write `"unlabeled"` (peer succeeded but no recognizable label), `"timeout"` (peer timed out), or `"error"` (any other failure). **Always write values as quoted strings** — bare `yes` and `no` are YAML booleans and will deserialize as `True`/`False` in most parsers. Do NOT omit failed peers — recording them is the point.
- All other `<...>` fields without literal angle brackets in the YAML below

Format:

```markdown
---
feature: <feature>
gate: before_tasks
council_label: "yes" | "no" | "tradeoff" | "degraded"   # always quote; bare yes/no are YAML booleans
quorum: <int>
participants: [claude, codex, gemini, ...]
peer_labels:
  claude: "yes" | "no" | "tradeoff" | "unlabeled" | "timeout" | "error"
  codex: "yes" | "no" | "tradeoff" | "unlabeled" | "timeout" | "error"
  gemini: "yes" | "no" | "tradeoff" | "unlabeled" | "timeout" | "error"
  # ... one entry per participant; quote every value
mode: <mode>
transcript: <path under .llm-council/runs/>
extension_version: <extension_version>
timestamp: <ISO-8601 UTC>
---

# Council Plan Review — <feature>

**Verdict:** <one-sentence summary of the consensus>

## Strongest reasons
- ...

## Dissent / risks flagged
- ...

## Recommendations
- ...

> Full transcript: <transcript path>
```

The "strongest reasons", "dissent", and "recommendations" sections should aggregate the peer responses — not paste them verbatim. Lift the highest-signal points; the full transcript is one click away.

## Step 7 — Surface to the user

If `prompt_user` is `true`, print exactly this block (substituting values):

```
[council] before_tasks gate — verdict: <label> (<quorum>/<peers>)
[council] <one-line summary>
[council] Evidence: <evidence path>
```

If the label is `no` or `tradeoff`, append one line:

```
[council] Verdict is advisory — proceed to /speckit.tasks anyway? [Y/n]
```

The user may proceed regardless of label. Do not auto-block. Do not loop the council.

## Failure modes — be explicit, never silent

| Condition | Action |
|---|---|
| No `plan.md` for active feature | Abort, tell user to run `/speckit.plan` |
| `llm-council` not installed | Abort, link to install docs |
| Pre-flight cost > cap | Prompt user, never silently override |
| Provider failure / quorum not reached | Write `council_label: degraded` artifact, surface error |
| Malformed config file | Warn once, use defaults, continue |
| Transcript write fails | Surface error with the inline summary so the user has the verdict in console |
