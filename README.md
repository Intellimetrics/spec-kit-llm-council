# spec-kit-llm-council

**A code-review jury for AI-driven development workflows.** When you use Spec Kit to drive a coding agent through "spec → plan → tasks → implement," this extension pauses at the high-leverage moments and asks a panel of LLMs to weigh in *before* the agent commits to a path that compounds expensively.

If you've never heard of Spec Kit or `llm-council`, read the next two sections first. If you've used both, jump to [Install](#install).

## What you need to know first

This tool is glue between two existing tools.

### 1. Spec Kit

[GitHub Spec Kit](https://github.com/github/spec-kit) is GitHub's open-source toolkit for **Spec-Driven Development** — driving an AI coding agent (Claude Code, Codex, Gemini, Cursor, etc.) through structured phases instead of one-shot prompting. The agent walks a pipeline:

```
/speckit.constitution → /speckit.specify → /speckit.plan → /speckit.tasks → /speckit.implement
```

Each step is a slash command and produces a markdown artifact in `specs/<feature>/`: a constitution (project principles), a spec (what to build), a plan (how), a task list (decomposed work), and finally implemented code. Spec Kit installs into your repo as a `.specify/` directory plus per-agent skills (e.g., `.claude/skills/`).

If you haven't installed Spec Kit yet, **[start there](https://github.com/github/spec-kit)**. This extension only makes sense once you've seen that workflow.

### 2. `llm-council`

[`llm-council`](https://github.com/Intellimetrics/llm-council) is a separate tool that asks multiple LLMs the same question in parallel and aggregates their verdicts. It's a CLI and an [MCP](https://modelcontextprotocol.io/) server. You give it a question and some context files; it spawns Claude, Codex, Gemini, and any other LLMs you've configured (OpenRouter peers, OpenAI-compatible endpoints, local Ollama models); each model returns a verdict (`yes` / `no` / `tradeoff`); `llm-council` aggregates them with quorum rules and writes a transcript.

The point: independent models with different training data, different priors, and different failure modes catch things any single model misses.

### 3. This extension

`spec-kit-llm-council` registers three hooks in the Spec Kit lifecycle, plus a few extra commands:

- **After `/speckit.specify` runs** (the `after_specify` gate) — it offers to convene the council to review your fresh `spec.md`. Catches the upstream poison case: a flawed spec compounds through plan + tasks + code, and by the time later gates fire the bad framing is already inherited.
- **Before `/speckit.tasks` runs** (the `before_tasks` gate) — it offers to convene the council to review your `plan.md`. A flawed plan compounds into 30+ flawed tasks; catching it here is cheap.
- **Before `/speckit.implement` runs** (the `before_implement` gate) — it offers to convene the council to review `tasks.md` against the plan. A flawed task list generates correspondingly bad code.

When the council runs, it writes its verdict to `.specify/council/<feature>/<gate>-review.md` as a durable audit artifact in your repo, and prints a summary to your agent's session. The verdict is **always advisory** — nothing blocks the SDD lifecycle. You read the summary and decide.

Three additional commands ship alongside the hook commands:

- **`audit`** — runs the council against whatever artifacts exist in `.specify/` at any time, outside the lifecycle gates. Use for ad-hoc reviews when you land mid-stream.
- **`dry-run`** — previews the council prompt, peer set, projected cost, and which peers will succeed in your environment, without spending any tokens.
- **`last`** — prints the most recent verdict for the active feature without re-running the council; warns if the inputs have changed since the verdict was recorded.

## Why this exists

When an AI agent writes a plan and then implements that plan, errors compound silently. Small misunderstandings at the planning stage become structural bugs at the implementation stage, and by the time the issue surfaces you've spent significant tokens and hours. The same model that wrote the plan tends to *rationalize* the choices it made when reviewing them — so single-model review catches less than you'd expect.

A panel of models — with different training data, different priors, different failure modes — catches things one model misses. That's the value of the jury pattern at decision-point gates, rather than as a general code-review tool.

## Install

You'll install two things: the `llm-council` tool itself, and this Spec Kit extension that wires it into your project.

```bash
# 1. Install the llm-council tool (one-time, system-wide).
#    This gives you the `llm-council` CLI and the MCP server binary.
uv tool install llm-council

# 2. Install this extension into your spec-kit project.
#    Run this from inside your spec-kit-initialized repo.
specify extension add llm-council
```

If you don't yet have Spec Kit installed (i.e., the `specify` command isn't on your `PATH`), [install it first](https://github.com/github/spec-kit).

## Choose your participants

The council is **bring-your-own-models**. You decide who sits on the jury — from one model up to a dozen — by configuring `.llm-council.yaml` in your project root (or accepting `llm-council`'s built-in defaults). This extension doesn't care how many or which models; it invokes whatever you've set up.

> Note: `.llm-council.yaml` is the **`llm-council` tool's** configuration (which models, which modes, which API endpoints). It's separate from this extension's configuration at `.specify/extensions/llm-council/llm-council-config.yml` (which controls how this extension *uses* `llm-council` — cost cap, evidence-file location, prompt knobs). You'll typically touch the extension config first; only edit `.llm-council.yaml` if you want to add custom peers or change the jury composition.

| Source | Examples | Setup |
|---|---|---|
| Native CLIs (`cli` adapter) | Claude Code, Codex, Gemini | Install the CLI, sign in once. |
| Hosted via OpenRouter (`openai_compatible` adapter) | DeepSeek, Qwen, GLM, Kimi (incl. `:free` tiers) | Set `OPENROUTER_API_KEY`. |
| Other OpenAI-compatible endpoints (`openai_compatible` adapter) | Any HTTP API that mirrors the OpenAI chat-completions shape | Configure base URL + API key in `.llm-council.yaml`. |
| Local / offline models (`ollama` adapter) | Ollama (e.g. `qwen3-coder`, `llama3`) | Run `ollama serve`, reference the model in `.llm-council.yaml`. |

You can run with **as few as one peer** (no consensus, but you still get an evidence artifact from a single judge) or as many as your config declares. Mix and match freely.

**Quick recipes:**

- **Single judge, just one CLI:** set `participants: ["claude"]` (or whichever CLI you have) in the extension config. The artifact records a one-judge advisory verdict — useful as a sanity gate even without true consensus.
- **Free-tier consensus:** in `.llm-council.yaml`, enable a couple of OpenRouter `:free` peers (e.g., `qwen_coder_free`) and set `mode: review-cheap`. Full jury at near-zero cost.
- **Fully private / offline:** set `mode: private-local` in the extension config to use only Ollama-backed peers. No data leaves the machine.
- **Cross-lab diversity:** `mode: diverse` curates a jury spanning multiple model labs.
- **US-only:** `mode: us-only` restricts to US-origin native CLIs.

Run `llm-council list` to see every participant and mode currently available in your environment. See the [`llm-council` configuration docs](https://github.com/Intellimetrics/llm-council#configuration) for the full `.llm-council.yaml` schema and how to add custom peers.

## What gets sent where

By default the council runs `mode: plan`, which sends the active feature's `spec.md`, `plan.md`, and `.specify/memory/constitution.md` to **multiple third-party LLMs** (Claude, Codex, and Gemini CLIs, plus DeepSeek via OpenRouter if `OPENROUTER_API_KEY` is set). If your spec contains anything sensitive — IP, customer data, regulated content, internal-only architecture — confirm where it goes before running. Run `llm-council list` to see the exact participants `plan` mode resolves to in your environment.

| Posture | How | What leaves the machine |
|---|---|---|
| Default (`mode: plan`) | No config change | spec/plan/constitution → plan-mode's curated peer set (premium CLIs + DeepSeek) |
| Native CLIs only | `participants: ["claude", "codex", "gemini"]` | spec/plan/constitution → CLI providers per their TOS, no OpenRouter |
| Fully offline | `mode: private-local` (Ollama-backed peers) | Nothing — everything runs locally |

Council transcripts always land in `.llm-council/runs/` regardless of posture. Add that directory to your project's `.gitignore` if you don't want transcripts committed.

## Commands

Six commands ship in v0.3.0. The four review/audit commands convene the council and spend tokens; `dry-run` and `last` are read-only.

| Command | Purpose | Spends tokens? |
|---|---|---|
| `speckit.llm-council.spec-review` | Convene jury on `spec.md` (fired automatically at the `after_specify` gate) | Yes |
| `speckit.llm-council.plan-review` | Convene jury on `plan.md` (fired automatically at the `before_tasks` gate) | Yes |
| `speckit.llm-council.implement-review` | Convene jury on `tasks.md` against `plan.md` (fired automatically at the `before_implement` gate) | Yes |
| `speckit.llm-council.audit` | Run the council against the current `.specify/` state at any time, outside the lifecycle gates | Yes |
| `speckit.llm-council.dry-run` | Preview prompt + cost + peer-readiness diagnostics for any gate | No |
| `speckit.llm-council.last` | Print the last recorded verdict for the active feature; warns if `spec.md`/`plan.md` changed since | No |

All six take an optional feature-slug argument (e.g., `... 003-user-auth`) to override branch-based active-feature resolution.

## Usage

When you run `/speckit.specify`, `/speckit.tasks`, or `/speckit.implement` (or their equivalents in your integration), spec-kit surfaces a notice to the host agent in this format:

```
**Optional Hook**: llm-council
Command: `<invocation>`
Description: Multi-model consensus review of plan.md before /speckit.tasks

Prompt: Convene LLM council to review the plan before task generation?
To execute: `<invocation>`
```

The agent typically asks you, then invokes the command if you confirm. (Some agents may proceed automatically based on context — that's a property of the agent, not of this extension.)

The literal `<invocation>` depends on your spec-kit integration mode:

| Integration | Invocation (for `plan-review`; rename `plan-review` to the command you want) |
|---|---|
| Claude Code (skills mode, default) | `/speckit-llm-council-plan-review` |
| Codex CLI (skills mode) | `$speckit-llm-council-plan-review` |
| Cursor (skills mode) | `/speckit-llm-council-plan-review` |
| Kimi | `/skill:speckit-llm-council-plan-review` |
| Claude Code / Codex (legacy commands, no skills) | `/speckit.llm-council.plan-review` |
| Gemini, Copilot, Windsurf, and other non-skill agents | `/speckit.llm-council.plan-review` |

You can bypass the hooks and invoke any command directly at any time using whichever invocation form your integration uses.

When a review command runs, the council reads the relevant artifacts, casts verdicts, writes evidence, and prints an inline summary:

```
[council] before_tasks gate — verdict: yes (4/4)
[council] Plan is consistent with spec; council flags missing acceptance criteria for §3.2 — non-blocking.
[council] Evidence: .specify/council/003-user-auth/plan-review.md
```

The hooks are **always advisory**: nothing blocks `/speckit.tasks` or `/speckit.implement`. A `no` or `tradeoff` verdict is information, not a stop sign — you decide whether to refine or proceed.

### Preview without spending tokens

Run `dry-run` to see exactly what the council would be asked, which peers will succeed in your current environment, and the projected cost — without convening the jury:

```
[council] dry-run for 003-user-auth (mode: plan)
[council] participants: claude, codex, gemini, deepseek_v4_pro
[council] inputs:
[council]   spec.md       (542 chars)  ✓
[council]   plan.md       (612 chars)  ✓
[council]   constitution  (3041 chars) ✓
[council] prompt total: ~877 tokens
[council] estimated cost: $0.0017 (cap: $0.50) ✓ under cap
[council] peer readiness:
[council]   gemini             ✗ not in a trusted directory (set GEMINI_CLI_TRUST_WORKSPACE=true)
[council] effective quorum: 3/4 (min: 2) — will meet quorum
```

### Recall the last verdict

Run `last` to surface the most recent verdict for the active feature without re-running:

```
[council] last verdict for 003-user-auth — yes (4/4)
[council] Plan is consistent with spec; minor gap on acceptance criteria for §3.2.
[council] Recorded: 2026-05-04T05:04:34Z (gate: before_tasks, mode: plan)
[council] Evidence: .specify/council/003-user-auth/plan-review.md
```

If `plan.md` or `spec.md` has been modified since the verdict was recorded, `last` prints a stale-evidence warning so you know to re-run the council.

## Configuration

After install, edit `.specify/extensions/llm-council/llm-council-config.yml`:

| Key | Default | Notes |
|---|---|---|
| `mode` | `plan` | Council mode (run `llm-council list`) |
| `max_cost_usd` | `0.50` | Pre-flight cost cap; refuses to run if exceeded |
| `participants` | `[]` | Override peers; empty = mode defaults |
| `inputs` | `[spec.md, plan.md, constitution.md]` | Files included in council prompt |
| `evidence_path` | `.specify/council/{feature}/plan-review.md` | Output location |
| `prompt_user` | `true` | Surface inline summary after run |

Environment overrides for one-off runs:
- `SPECKIT_COUNCIL_MODE` — overrides `mode`
- `SPECKIT_COUNCIL_MAX_COST` — overrides `max_cost_usd`

## Evidence schema

Each gate writes to its own evidence file under `.specify/council/<feature>/`:

| Gate | Evidence file | Source command |
|---|---|---|
| `after_specify` | `spec-review.md` | `speckit.llm-council.spec-review` |
| `before_tasks` | `plan-review.md` | `speckit.llm-council.plan-review` |
| `before_implement` | `implement-review.md` | `speckit.llm-council.implement-review` |
| (any time) | `audit-<YYYYMMDD-HHMMSS>.md` | `speckit.llm-council.audit` |

```yaml
---
feature: 003-user-auth
gate: before_tasks                          # or before_implement
council_label: yes | no | tradeoff | degraded
quorum: 4
participants: [claude, codex, gemini, deepseek_v4_pro]
mode: plan
transcript: .llm-council/runs/20260504_050434_plan-review.md
extension_version: 0.2.0
timestamp: 2026-05-04T05:04:34Z
---
```

`council_label: degraded` means a provider failed or quorum was not reached — the run did not produce a trustworthy verdict and the artifact is informational only.

## Not planned

These came up during planning and are explicitly out of scope. They will not appear in a future "deferred" roadmap. The hook surface caps at three; any future proposal for a fourth hook must come with a kill of one of the existing three.

**Hooks that won't be added:**

- **`before_analyze` / `after_implement` (post-implementation review).** Reviewing already-written code overlaps with `/speckit.analyze`. The council adds the most leverage *before* expensive work, not after. Re-confirmed cut in v0.3.0 planning.
- **`after_clarify`.** `/speckit.clarify` is itself structured Q&A; council review of its output is meta-on-meta.
- **`before_constitution` / `after_constitution`.** Constitution is set rarely; ROI per token is too low.
- **`before_checklist` / `after_checklist`.** Reviewing a quality checklist is meta and low-signal.
- **`before_taskstoissues`.** Task list is already gated by `before_implement`; a second pass is redundant.

**Other features that won't ship:**

- **Optional blocking semantics.** Advisory-only is the design's load-bearing safety property. Promising blocking invites scope creep into escape-hatch UX (`--force` flags, env overrides, config precedence).
- **Profiles (`micro` / `compact` / `full`).** Adds configuration surface for what the existing `mode:` config already provides via llm-council.
- **CI / manifest validation.** Solo-maintainer overhead; catalog submission validation handles the same concern.
- **Multi-feature project handling.** Single-feature project + explicit slug argument covers the realistic use cases. The deterministic resolver fails closed when it can't decide.
- **Spec Kit presets integration.** Speculative; would commit to a value-add we can't yet articulate.
- **Spec Kit workflow integration.** Feature does not yet exist in spec-kit; can't design against vapor.

## License

MIT — see [LICENSE](LICENSE).
