---
name: stackiq-orchestrator
description: >
  Master entry point for StackIQ, a weekly AI/SaaS utilization diagnostic pipeline (5 skills,
  linear 1 to 5). ALWAYS invoke this first; sub-skills (intake, diagnostic, insight, action) are
  routed internally, never called directly by the user. Trigger WHENEVER the user states a problem
  about AI or SaaS adoption, spend, seats, licenses, tool utilization, wasted subscriptions, or tool
  sprawl, or says "run StackIQ", "audit my stack", "are my tools being used", "find wasted spend",
  "check my SaaS costs", "weekly stack review". Pure routing, zero analysis. Boot sequence runs an
  env self-check and a Chrome connection check (both silent, hard-stop on fail), greets, captures the
  problem statement into state/session.json, then auto-advances the stages. Holds the problem
  statement for the whole session via the state file. Halt-and-report on any downstream failure.
---

# StackIQ Orchestrator (Skill 1 of 5)

The single entry point for a StackIQ run. It routes; it never researches, analyzes, or writes findings. All substantive work happens in Skills 2 to 5. Its only job is to boot the environment, capture the problem statement, own the shared state file, and advance the pipeline stage by stage.

## Canonical state contract

StackIQ shares all data through two files under a `state/` directory in the working folder:

- `state/session.json` — the live run. Every skill reads it on entry and appends its own block on exit. No skill re-injects prompts into another.
- `state/history/run_<n>.json` — one durable snapshot per weekly run, the source for week-over-week diffing.

The full field-by-field schema for both files lives in `references/state-schema.md`. Read it before creating or writing state. The orchestrator owns the `session_id`, `created_at`, `run_number`, `problem_statement`, `stage`, and `env` blocks; each sub-skill owns exactly one output block (`intake`, `diagnostic`, `insight`, `action`).

## Boot sequence (run in order, stop on first failure)

1. **Env self-check (silent).** Confirm the working directory is writable and that `state/session.json` can be created. If not, hard stop: report "StackIQ can't write to its working directory" and end. Do not proceed.
2. **Chrome check (silent).** Confirm Claude in Chrome is connected (Diagnostic and Action may need signed-in read/click access). If not connected, hard stop and tell the user to connect the Chrome extension before rerunning. Do not proceed.
3. **Welcome message (mandatory, always print to the user).** This is the first thing the user sees — never skip it, never fold it silently into a tool call. After the silent checks pass, print a short welcome that names StackIQ, says in one line what it does, and asks for the problem statement. Then stop and wait for the user's reply. Keep it to ~2–3 short lines. Use this template (adapt wording, keep the structure):

   > **StackIQ — AI/SaaS utilization diagnostic.**
   > I audit your tools for wasted spend, idle seats, overlap, and access risk, then hand you a sized report and approve-before-anything-happens fixes.
   > What tool, spend, or adoption problem should I dig into?

   If this is run 2+ (prior snapshots exist), add a line noting it's a follow-up run and will compare against last week.
4. **Capture the problem statement.** Write the user's free-text answer verbatim to `session.json → problem_statement`. Determine `run_number`: if `state/history/` already has snapshots, this run is `max(existing)+1`; otherwise `1`.
5. **Initialize state.** Create `session.json` with a fresh `session_id`, `created_at` (ISO 8601), `run_number`, the problem statement, `env` results, and `stage.status` all set to `pending` except `orchestrator: active`.

## Routing (linear, auto-advance)

Advance the stages 1 → 2 → 3 → 4 → 5 without asking the user to re-confirm between stages. For each stage: set its `stage.status` to `active`, invoke the corresponding skill, and on its return set the status to `done` and move on.

| Stage | Skill | Owns block |
|-------|-------|------------|
| intake | 02_stackiq-intake | `intake` |
| diagnostic | 03_stackiq-diagnostic | `diagnostic` + writes `history/run_<n>.json` |
| insight | 04_stackiq-insight | `insight` |
| action | 05_stackiq-action | `action` + appends to `history/run_<n>.json` |

Set `stage.current` to the stage in flight throughout, so a resumed run knows where it stopped.

## Guardrails

- **Never analyze.** The orchestrator does no research, sizing, sourcing, or drafting. If tempted to "just check one number," stop — that belongs to Diagnostic.
- **Hard stop** on env or Chrome check failure (steps 1 to 2). No partial runs.
- **Halt-and-report** on any downstream skill failure: set `stage.status.<skill>` to `halted`, write `stage.halt_reason` with the cause, tell the user plainly what failed and at which stage, and stop. Do not silently skip a stage or fabricate its output.
- **One writer per block.** The orchestrator never writes another skill's block; sub-skills never write the orchestrator's stage/env fields.

## Handoff

Control passes down the chain purely through `session.json`. On completion of Skill 5, the run is closed: `stage.current` = action, all `stage.status` = done, and `history/run_<n>.json` holds the snapshot next week's Diagnostic will diff against.
