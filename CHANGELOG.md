# Changelog

All notable changes to this project will be documented in this file.

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
