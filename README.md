# StackIQ

An AI/SaaS utilization diagnostic for Claude (Cowork / Claude Code). StackIQ audits how well an organization's AI and SaaS tools are actually used — finding idle seats, tier mismatches, duplicate tools, orphaned accounts, and shadow IT — sizes every finding in dollars, seats, or accounts, ships an Excel workbook and a CEO-level deck, and then executes approved remediations behind a hard human-approval gate. Name the tools you want checked, or ask StackIQ to find your whole stack itself.

<img width="1345" height="402" alt="image" src="https://github.com/user-attachments/assets/7a7ca82b-6d6c-44f7-8a1a-25fd7c328f6f" />


It is a five-stage, read-then-act pipeline. Stages 1–4 are strictly read-only; only stage 5 writes or acts, and never without an explicit human YES.

## Pipeline

| # | Skill | Role |
|---|-------|------|
| 1 | `stackiq-orchestrator` | Entry point. Boot checks, captures the problem statement, owns shared state, routes stages 1→5. Pure routing. |
| 2 | `stackiq-intake` | Turns the problem statement into a structured intake (scope, access ceiling, priority, scale, lookback) and resolves whether the run is against named tools, a full-stack auto-discovery, or both. Infer-first, confirm-gated. |
| 3 | `stackiq-diagnostic` | Read-only analysis engine. When asked for a full-stack audit, first discovers the tool list (MCP connectors + SSO app catalog + expense scan, reconciled) and triages every tool before committing to a full deep-dive. Source ladder (MCP → files → signed-in Chrome), six analysis modes, dollar sizing, week-over-week diffing. Always emits one consolidated output. |
| 4 | `stackiq-insight` | Renders findings money-descending into an Excel workbook (`xlsx`) and a category-structured deck (`pptx`), plus a coverage summary whenever discovery ran. |
| 5 | `stackiq-action` | The only writer. Billing-model-aware remediations + a fully-wired notify-by-email action, all behind a hard approval gate. Records outcomes to three dated sinks. |

## Install

From Claude Code or Cowork:

```
/plugin marketplace add CloudMotivGlobal/stackiq
/plugin install stackiq-plugin@stackiq
```

Then start a run by stating a problem, e.g. "audit HubSpot for wasted spend," "run StackIQ," or "find every tool I'm paying for and tell me what's wasted."

## How it works

- **Shared state.** All stages communicate through `state/session.json` (the live run) and `state/history/run_<n>.json` (per-run snapshots for week-over-week diffing). Each skill reads on entry and appends only its own block. The canonical schema is in `skills/01_stackiq-orchestrator/references/state-schema.md`.
- **Read-only until approved.** Diagnostic navigates but never clicks destructive controls. Action is the sole writer and holds every side-effectful step behind an explicit user confirmation, respecting the access ceiling captured at intake.
- **Full-stack discovery.** Instead of only auditing tools you name, you can ask StackIQ to find your whole stack. It reconciles three sources — connected MCP integrations, your SSO/identity-provider app catalog (Okta, Google Workspace, Microsoft Entra), and expense/billing data — into one deduped tool list, merges it with anything you named, then triages every tool with a cheap pass before running the full six-mode analysis only on the ones that look wasteful or risky. Named and discovered tools are reported together, ranked purely by dollar impact; a coverage summary shows what was checked, what wasn't reachable, and why. See `skills/03_stackiq-diagnostic/references/discovery.md`.
- **Confirmed action: notify-by-email.** Emails the responsible party the required fix, then tracks it to closure (`awaiting_confirmation` → `executed`). Send paths: a send-capable email connector, the Outlook desktop app (computer use), signed-in webmail via Chrome, or a drafted email the user sends. It asks which channel to use rather than assuming, and never sends internal notes from an off-profile mailbox without consent.

## Requirements

- Claude with plugin support (Cowork or Claude Code).
- Claude in Chrome connected (Diagnostic and some Action paths use the signed-in session; full-stack discovery additionally uses it to read the SSO app catalog and, where needed, an expense platform).
- MCP connectors for the tools being audited are used when available; the pipeline degrades to user files and signed-in Chrome otherwise. No API keys are stored by the plugin.

## Layout

```
.claude-plugin/
  plugin.json          plugin manifest
  marketplace.json     self-referencing marketplace listing
skills/
  01_stackiq-orchestrator/   SKILL.md + references/state-schema.md
  02_stackiq-intake/
  03_stackiq-diagnostic/     SKILL.md + references/{modes,chrome-coverage,discovery}.md
  04_stackiq-insight/
  05_stackiq-action/         SKILL.md + references/action-playbook.md
```

## License

MIT — see [LICENSE](LICENSE).
