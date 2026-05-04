# Changelog

All notable changes to this project will be documented in this file.

## [0.3.0] — 2026-05-04

### Added

- **`after_specify` hook + `speckit.llm-council.spec-review` command** — convene the jury *after* `/speckit.specify` writes the spec, *before* `/speckit.plan` operates on it. Catches the upstream poison case: a flawed spec compounds through plan + tasks + code, and by the time `before_tasks` fires the bad framing is already inherited. Council reviews acceptance-criteria gaps, hidden assumptions, scope ambiguity, wrong-problem framing, and constitution-alignment gaps. Evidence at `.specify/council/<feature>/spec-review.md`.
- **`speckit.llm-council.audit` command** — standalone, non-hook command that runs the council against whatever artifacts currently exist for the active feature. Question adapts to what's present (spec only / spec + plan / spec + plan + tasks). Use case: "I'm mid-stream and want a sanity check on my current state without waiting for a lifecycle gate." Audits can be run repeatedly; each writes a timestamped evidence file at `.specify/council/<feature>/audit-<YYYYMMDD-HHMMSS>.md`.

### Hook surface caps at 3

The extension now registers three hooks (`after_specify`, `before_tasks`, `before_implement`). **This is the ceiling.** Any future proposal for a fourth hook must come with a kill of one of these three — hook count is itself a budget. Adding more lifecycle gates beyond the three highest-blast-radius transitions trades signal for noise.

### Not planned (added to the existing kill list)

The following were considered for v0.3.0 and explicitly rejected. They will not return as "deferred":

- **`after_clarify` hook** — `/speckit.clarify` is itself structured Q&A; council review of its output is meta-on-meta.
- **`before_constitution` / `after_constitution` hooks** — constitution is set rarely (once per project); ROI per token is too low to justify a registered hook.
- **`before_checklist` / `after_checklist` hooks** — reviewing a quality checklist is meta and low-signal.
- **`before_taskstoissues` hook** — task list is already gated by `before_implement`; a second pass is redundant.
- **Reopening `before_analyze`** — already cut in v0.2.0 with the rationale that it overlaps `/speckit.analyze`. The v0.3.0 council pass surfaced no new evidence; re-confirmed cut.
- **Spec Kit presets integration** — speculative; would commit to a value-add we can't yet articulate.
- **Spec Kit workflow integration** — feature does not yet exist in spec-kit; can't design against vapor.

### Requires
- `llm-council >= 0.4.2` (unchanged from v0.2.0).
- `spec-kit >= 0.2.0` (unchanged).

## [0.2.0] — 2026-05-04

### Added

- **`before_implement` hook** + **`speckit.llm-council.implement-review` command** — convene the jury after `/speckit.tasks` to review whether the generated task list faithfully decomposes the plan. Catches missing tasks, phantom tasks, hidden complexity, sequencing risks, and constitution violations baked into decomposition. Evidence at `.specify/council/<feature>/implement-review.md`. Same advisory-only invariant as plan-review.
- **`speckit.llm-council.dry-run` command** — preview the council prompt, peer set, projected cost, and peer-readiness diagnostics (trusted-directory issues, missing API keys, missing CLIs, Ollama daemon status) without spending any tokens. Surfaces would-be-degraded runs so users can fix the environment before running for real.
- **`speckit.llm-council.last` command** — print the most recent council verdict for the active feature without re-convening the jury. Includes a stale-evidence check that warns if `plan.md` or `spec.md` has been modified since the verdict was recorded.

### Behavior

- v0.2.0 commands all reuse the deterministic active-feature resolver from v0.1.0 (strip common branch prefixes, fail closed on ambiguity, accept explicit slug as `$ARGUMENTS`).

### Not planned (decisively cut, not deferred)

These came up in v0.2.0 planning and are explicitly declared out of scope. They will not appear in a future "deferred" section:

- **`before_analyze` hook** — pre-implementation gates (plan-review, implement-review) are where jury input adds the most leverage. Reviewing already-written code overlaps with `/speckit.analyze`, which is a single-model but structured pass and adequate for that purpose.
- **Optional blocking semantics** — advisory-only is the design's load-bearing safety property. Promising opt-in blocking invites scope creep into escape-hatch UX (`--force`, env overrides, config precedence) and would compromise the "you decide" stance.
- **Profiles (`micro` / `compact` / `full`)** — adds configuration surface for a depth tradeoff that the existing `mode:` config already provides via llm-council's mode system.
- **CI / manifest validation** — out of scope for a solo-maintained extension. Catalog submission validation handles the same concern at the right layer.
- **Cap-raise UX with explicit confirmation** — the prose-level "do not silently override" guidance has held; verdict cap is informational and the agent surfaces estimate exceedances naturally.
- **Multi-feature project handling** — single-feature project + explicit slug argument covers the realistic use cases.

### Requires
- `llm-council >= 0.4.2` (unchanged from v0.1.0).
- `spec-kit >= 0.2.0` (unchanged).

## [0.1.0] — 2026-05-04

Initial release.

### Added
- `speckit.llm-council.plan-review` command — convenes a multi-LLM consensus jury to review `plan.md` for the active feature.
- `before_tasks` optional hook surfaces a prompt to the host agent ahead of `/speckit.tasks`; the agent asks the user and invokes the command on confirmation.
- File-based evidence artifact at `.specify/council/<feature>/plan-review.md` with frontmatter (label, quorum, participants, transcript path, extension version).
- Both `config-template.yml` (the upstream template) and `llm-council-config.yml` (the live config users edit) — spec-kit copies the extension tree without rendering, so both ship as siblings.
- "What gets sent where" privacy table in the README covering default / native-CLI-only / fully-offline postures.

### Behavior
- Verdict is **advisory only** — does not block `/speckit.tasks` regardless of label.
- Active-feature resolution is deterministic: strips common branch prefixes (`feat/`, `bugfix/`, etc.), matches against `specs/<slug>/`, and fails closed if ambiguous (never silently picks the most-recently-modified directory).
- Pre-flight cost estimation refuses to run if the estimate exceeds the configured cap; hosted peers without a catalog price refuse rather than slipping past as $0.
- Graceful degradation when providers are unavailable (records `council_label: degraded` rather than fabricating a verdict).

### Requires
- `llm-council >= 0.4.2` available as MCP server or CLI on PATH. The CLI fallback uses `llm-council estimate --max-cost-usd`, which lands in 0.4.2. (Declared as `requires.tools` metadata; the command file also checks availability at runtime.)
- `spec-kit >= 0.2.0`.
