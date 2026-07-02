# StackIQ

A weekly AI/SaaS utilization diagnostic for Claude (Cowork / Claude Code). StackIQ audits how well an organization's AI and SaaS tools are actually used — finding idle seats, tier mismatches, duplicate tools, orphaned accounts, and shadow IT — sizes every finding in dollars, seats, or accounts, ships an Excel workbook and a CEO-level deck, and then executes approved remediations behind a hard human-approval gate.

It is a five-stage, read-then-act pipeline. Stages 1–4 are strictly read-only; only stage 5 writes or acts, and never without an explicit human YES.

## Pipeline

| # | Skill | Role |
|---|-------|------|
| 1 | `stackiq-orchestrator` | Entry point. Boot checks, captures the problem statement, owns shared state, routes stages 1→5. Pure routing. |
| 2 | `stackiq-intake` | Turns the problem statement into a structured intake (scope, access ceiling, priority, scale, lookback). Infer-first, confirm-gated. |
| 3 | `stackiq-diagnostic` | Read-only analysis engine. Source ladder (MCP → files → signed-in Chrome), six analysis modes, dollar sizing, week-over-week diffing. Always emits output. |
| 4 | `stackiq-insight` | Renders findings money-descending into an Excel workbook (`xlsx`) and a category-structured deck (`pptx`). |
| 5 | `stackiq-action` | The only writer. Billing-model-aware remediations + a fully-wired notify-by-email action, all behind a hard approval gate. Records outcomes to three dated sinks. |

## Install

From Claude Code or Cowork:

```
/plugin marketplace add CloudMotivGlobal/stackiq
/plugin install stackiq-plugin@stackiq
```

Then start a run by stating a problem, e.g. "audit my SaaS stack for wasted spend" or "run StackIQ".

## How it works

- **Shared state.** All stages communicate through `state/session.json` (the live run) and `state/history/run_<n>.json` (per-run snapshots for week-over-week diffing). Each skill reads on entry and appends only its own block. The canonical schema is in `skills/01_stackiq-orchestrator/references/state-schema.md`.
- **Read-only until approved.** Diagnostic navigates but never clicks destructive controls. Action is the sole writer and holds every side-effectful step behind an explicit user confirmation, respecting the access ceiling captured at intake.
- **Confirmed action: notify-by-email.** Emails the responsible party the required fix, then tracks it to closure (`awaiting_confirmation` → `executed`). Send paths: a send-capable email connector, the Outlook desktop app (computer use), signed-in webmail via Chrome, or a drafted email the user sends. It asks which channel to use rather than assuming, and never sends internal notes from an off-profile mailbox without consent.

## Requirements

- Claude with plugin support (Cowork or Claude Code).
- Claude in Chrome connected (Diagnostic and some Action paths use the signed-in session).
- MCP connectors for the tools being audited are used when available; the pipeline degrades to user files and signed-in Chrome otherwise. No API keys are stored by the plugin.

## Layout

```
.claude-plugin/
  plugin.json          plugin manifest
  marketplace.json     self-referencing marketplace listing
skills/
  01_stackiq-orchestrator/   SKILL.md + references/state-schema.md
  02_stackiq-intake/
  03_stackiq-diagnostic/     SKILL.md + references/{modes,chrome-coverage}.md
  04_stackiq-insight/
  05_stackiq-action/         SKILL.md + references/action-playbook.md
```

## License

MIT — see [LICENSE](LICENSE).
