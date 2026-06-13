# ReboundMan-ReleaseToolkit — Agent Instructions

This project follows the ReboundMan persona fleet. The global fleet lives in `~/.copilot/AGENTS.md`; this file lists project-specific overrides (rare) and the named-persona table for offline reference.

## House standards are binding
<!-- RM_standards_binding -->
This repo operates under `C:\Users\jeffjame\OneDrive\Code\ProjectPatterns\STANDARDS.md`; treat it as **binding, not advisory**. On starting work, run `Test-ProjectStandards`; do not regress any passing standard, and re-check affected standards when you add or change files.

## Cast (same as global, models are binding)

| Name | Role | Model |
|---|---|---|
| Hawk | security-auditor | `gpt-5.3-codex` |
| Bolt | performance-reviewer | `gpt-5.3-codex` |
| Sage | sceptical-architect | `claude-opus-4.7` |
| Forge | data-engineer | `claude-opus-4.7` |
| Atlas | ux-ui-researcher | `claude-opus-4.7` |
| Lens | ux-critic | `claude-sonnet-4.6` |
| Beacon | accessibility-reviewer | `claude-sonnet-4.6` |
| Rookie | new-engineer | `claude-haiku-4.5` |
| Chaos | qa-saboteur | `gpt-5.4-mini` |
| Scout | e2e-tester | `claude-sonnet-4.6` |

Any persona invocation switches to the pinned model first. The table in `~/.copilot/AGENTS.md` is the single source of truth; this copy is for offline reference.

## Project-specific guidance

- **Stack:** TODO: confirm (no manifest found)
- **Auth family:** `TODO: confirm (none if local-only tool)`
- **Domain:** TODO: confirm (none if local tool)
- **Hosting:** TODO: confirm

## Per-repo persona overrides

None by default. Drop overrides in `.copilot/personas/<name>.md` if a specific persona should behave differently in this repo.

## Reviews

Fleet output lands in `reviews/<ticket>-<persona>.md` (gitignored).
