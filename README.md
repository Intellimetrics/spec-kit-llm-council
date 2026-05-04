# spec-kit-llm-council

Multi-LLM consensus jury for [Spec-Driven Development](https://github.com/github/spec-kit). Convenes a parallel jury of models to vote on `plan.md` before `/speckit.tasks` runs, and records the verdict as durable evidence.

## What it does

Wires [`llm-council`](https://github.com/Intellimetrics/llm-council) into the SDD lifecycle. At the `before_tasks` gate, spec-kit surfaces an optional-hook prompt to the host agent (Claude Code, Codex, Gemini, etc.). The agent asks you whether to convene the council; if you accept, multiple models read the active feature's `plan.md`, `spec.md`, and constitution and each casts a verdict (`yes` / `no` / `tradeoff`). Their aggregated label, dissent, and full transcript land at `.specify/council/<feature>/plan-review.md` as an audit artifact.

The verdict is **advisory only** — nothing blocks `/speckit.tasks`. You read the summary and decide whether to proceed, refine the plan, or override.

## Why a jury?

A single model reviewing a plan often rationalizes its own design choices. Independent models with different training distributions catch different failure modes. The jury pattern is most valuable at high-blast-radius gates: `before_tasks` is one such gate, because plan flaws compound through 30+ generated tasks before they're caught.

## Install

```bash
# 1. Install the underlying llm-council tool (one-time, system-wide)
uv tool install llm-council

# 2. Install the spec-kit extension
specify extension add llm-council
```

## Choose your participants

The council is **bring-your-own-models**. You decide who sits on the jury — from one model up to a dozen — by configuring `.llm-council.yaml` in your project (or accepting `llm-council`'s defaults). The extension doesn't care how many or which; it invokes whatever you've set up.

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

## Usage

Spec-kit registers `before_tasks` as an optional hook. When you run `/speckit.tasks` (or its equivalent in your integration), spec-kit surfaces a notice to the host agent in this format:

```
**Optional Hook**: llm-council
Command: `<invocation>`
Description: Multi-model consensus review of plan.md before /speckit.tasks

Prompt: Convene LLM council to review the plan before task generation?
To execute: `<invocation>`
```

The agent typically asks you, then invokes the command if you confirm. (Some agents may proceed automatically based on context — that's a property of the agent, not of this extension.)

The literal `<invocation>` depends on your spec-kit integration mode:

| Integration | Invocation |
|---|---|
| Claude Code (skills mode, default) | `/speckit-llm-council-plan-review` |
| Codex CLI (skills mode) | `$speckit-llm-council-plan-review` |
| Cursor (skills mode) | `/speckit-llm-council-plan-review` |
| Kimi | `/skill:speckit-llm-council-plan-review` |
| Claude Code / Codex (legacy commands, no skills) | `/speckit.llm-council.plan-review` |
| Gemini, Copilot, Windsurf, and other non-skill agents | `/speckit.llm-council.plan-review` |

You can also bypass the hook and invoke the command directly at any time using whichever invocation form your integration uses.

When the council runs, it reads the active feature's plan, casts verdicts, writes evidence to `.specify/council/<feature>/plan-review.md`, and prints an inline summary:

```
[council] before_tasks gate — verdict: yes (4/4)
[council] Plan is consistent with spec; council flags missing acceptance criteria for §3.2 — non-blocking.
[council] Evidence: .specify/council/003-user-auth/plan-review.md
```

The hook is **always advisory**: nothing blocks `/speckit.tasks`. A `no` or `tradeoff` verdict is information, not a stop sign — you decide whether to refine the plan or proceed.

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

```yaml
---
feature: 003-user-auth
gate: before_tasks
council_label: yes | no | tradeoff | degraded
quorum: 4
participants: [claude, codex, gemini, deepseek_v4_pro]
mode: plan
transcript: .llm-council/runs/20260504_050434_plan-review.md
extension_version: 0.1.0
timestamp: 2026-05-04T05:04:34Z
---
```

`council_label: degraded` means a provider failed or quorum was not reached — the run did not produce a trustworthy verdict and the artifact is informational only.

## Roadmap

v0.1.0 deliberately ships one hook, one command, and advisory verdicts. Candidates for v0.2.0 (driven by real-world usage):

- `before_implement` and `before_analyze` hooks
- Compact-first input packaging (briefs + diff excerpts) to reduce token spend
- Profiles (`micro` / `compact` / `full`) for varying depth
- Optional blocking semantics with a documented escape hatch

## License

MIT — see [LICENSE](LICENSE).
