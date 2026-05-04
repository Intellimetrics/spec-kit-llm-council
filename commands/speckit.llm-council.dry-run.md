---
description: "Preview the council prompt, peer set, and cost estimate without spending tokens"
tools:
  - 'llm-council/council_estimate'
  - 'llm-council/council_doctor'
---

# LLM Council — Dry Run

Preview what the council *would* be asked, *which peers* would respond, *what it would cost*, and *which peers will fail* in the current environment — **without convening the jury or spending any tokens.**

## User Input

```text
$ARGUMENTS
```

If the user passed an explicit slug as `$ARGUMENTS`, treat that slug as the feature and skip directly to step 3.

## Step 1 — Resolve the active feature

Use the same algorithm as `speckit.llm-council.plan-review` (Step 1). Abort with the same error messages on ambiguity.

## Step 2 — Load configuration

Read `.specify/extensions/llm-council/llm-council-config.yml` (defaults if absent). Same defaults as `plan-review`. Honor `SPECKIT_COUNCIL_MODE` and `SPECKIT_COUNCIL_MAX_COST` env overrides.

## Step 3 — Build the prompt (same as `plan-review` Step 3)

Use the same fixed council question, prepend any `Reviewer focus:` from `$ARGUMENTS`, and assemble the input file list. Resolve and read each input file's character count for display.

## Step 4 — Pre-flight peer readiness

Run `llm-council doctor --json` (or the MCP equivalent if available) and parse its output to identify peers that will fail in the current environment. Common conditions to surface:

- **Missing CLI binary** for a `cli` peer (e.g., `gemini` not on `PATH`)
- **Missing API key** for an `openai_compatible` peer (e.g., `OPENROUTER_API_KEY` unset)
- **Trusted-directory restriction** for Gemini CLI in the current `cwd` (recoverable with `GEMINI_CLI_TRUST_WORKSPACE=true`)
- **Ollama daemon not running** for an `ollama` peer

For each affected peer, capture the peer name and a one-line remediation hint.

## Step 5 — Cost estimate

Estimate cost via:

- **MCP path (preferred):** call `council_estimate` with the question, mode, participants (if overridden), and resolved input paths.
- **CLI fallback:**

  ```bash
  llm-council estimate \
    --mode "<mode>" \
    --max-cost-usd "<max_cost_usd>" \
    --context "<plan.md>" \
    [--context "<other input>"]... \
    --json \
    "<question>"
  ```

  Note: with `--max-cost-usd`, the CLI exits non-zero if the estimate exceeds the cap. That's expected; capture the output (the breakdown is still printed) and treat the non-zero exit as the cap-exceeded signal.

Parse the estimate result for: list of peers, prompt char count, projected token total, projected cost (per peer and total).

## Step 6 — Compute effective quorum

Cross-reference the estimated peer list against the pre-flight failure list from Step 4. Peers expected to fail will not contribute to quorum. Compute:

- `expected_peers`: total peers configured for the mode
- `expected_quorum`: `expected_peers - failing_peers`
- `min_quorum`: from llm-council's config (default 2 when 2+ peers configured)

If `expected_quorum < min_quorum`, mark the run as **would-be-degraded** so the user knows running for real won't produce a trustworthy verdict in the current environment.

## Step 7 — Print the dry-run summary

Format:

```
[council] dry-run for <feature> (mode: <mode>)
[council] participants: <comma-separated peer list>
[council] inputs:
[council]   spec.md       (<N> chars)  ✓
[council]   plan.md       (<N> chars)  ✓
[council]   constitution  (<N> chars)  ✓
[council] prompt total: ~<N> tokens
[council] estimated cost: $<X.XXXX> (cap: $<cap>) [✓ under cap | ✗ over cap]
```

If any peers will fail, append a "peer readiness" block:

```
[council] peer readiness:
[council]   gemini             ✗ not in a trusted directory (set GEMINI_CLI_TRUST_WORKSPACE=true)
[council]   deepseek_v4_pro    ✗ missing OPENROUTER_API_KEY
[council] effective quorum: <N>/<expected> (min: <min_quorum>) — <will / will not> meet quorum
```

If estimate exceeds cap or quorum will not be met, append:

```
[council] ⚠ Running for real in this environment would [exceed the cost cap | fail to reach quorum / produce a degraded verdict].
[council]   Fix the issues above, raise the cap, or accept a degraded run before invoking plan-review.
```

## What dry-run does NOT do

- Does not call `council_run` / `llm-council run`
- Does not write to `.specify/council/<feature>/`
- Does not update `.llm-council/runs/`
- Does not consume any model tokens (the only subprocess calls are `llm-council estimate` and `llm-council doctor`, neither of which invokes the peer models)

## Failure modes

| Condition | Action |
|---|---|
| No `plan.md` for active feature | Abort, point at `/speckit.plan` |
| `llm-council` not installed | Abort, link install docs |
| `llm-council estimate` returns malformed JSON | Surface the raw output, do not crash |
| `llm-council doctor` not available | Skip Step 4 with a one-line note; proceed with cost estimate only |
| Pre-flight detects 0 peers will succeed | Print full readiness report, mark as would-be-fully-degraded |
