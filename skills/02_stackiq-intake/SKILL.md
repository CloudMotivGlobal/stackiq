---
name: stackiq-intake
description: >
  StackIQ intake (Skill 2 of 5). Auto-invoked by the StackIQ orchestrator after the problem statement
  is captured; not normally triggered directly. Turns the free-text problem statement into a structured
  intake block covering scope (which tools, disambiguated), access ceiling, priority, seat scale, and
  metric lookback window. Parses the statement to pre-fill inferable fields, then asks only the missing
  core questions as a single tappable batch (scope multi-select with vendor/category disambiguation,
  access single-select, priority multi-select, scale single-select, optional notes, lookback only if
  priority is mixed), echoes a one-line summary, and requires an explicit Confirm before writing state.
  Question count scales 0 to 6 by completeness. Never asks about connectors — Diagnostic owns that.
  Writes the intake block to state/session.json and hands off to Diagnostic.
---

# StackIQ Intake (Skill 2 of 5)

Converts the orchestrator's free-text `problem_statement` into a structured `intake` block. Ask as little as possible: infer first, ask only what's missing, always confirm.

## On entry

Read `state/session.json → problem_statement`. Parse it to pre-fill any field it clearly implies (a named tool → scope; "we're overpaying" → priority `cut_cost` → lookback `12mo`; "nobody uses it" → priority `lift_adoption` → lookback `90d`; a team size → scale).

## Fields to produce

Write these into `intake` (see the orchestrator's `references/state-schema.md` for the exact shape and allowed values):

- `scope`: `tools[]` (each `{name, vendor, category}`, at least 1) and `breadth` (`single_tool | category | ecosystem`).
- `access`: `admin_full | limited | read_only | unsure` — the permission ceiling.
- `priority[]`: any of `cut_cost | lift_adoption | consolidate | de_risk` — flat, at least 1.
- `scale`: `<10 | 10-50 | 50-200 | 200+` (SMB seat bands).
- `lookback`: `90d | 12mo` — inferred from priority (cost → 12mo, adoption → 90d).
- `notes`: optional free text.
- `connectivity_hint`: if the user claims they have access/a connector, record the claim here. It is NOT source of truth — Diagnostic verifies.

## Completeness check, then ask

Evaluate the four core fields — Scope, Access, Priority, Scale. Then:

- **All four inferable** → skip straight to the confirm step.
- **Some missing** → render only the missing questions.
- **Too vague to infer Scope or Priority** → offer a generic fallback option set for those.

Present missing questions as one tappable batch, single step, with inferred values pre-selected:

- **Q1 Scope** — multi-select, adaptive, ≥1. Include disambiguation: if the user typed an ambiguous name (e.g. "Monday"), confirm which entity — the product (Monday.com), a vendor, or a whole category.
- **Q2 Access** — single-select.
- **Q3 Priority** — multi-select, ≥1.
- **Q4 Scale** (seat band) — single-select.
- **Q5 Additional info** — free text, optional.
- **Q6 Lookback** — inferred from priority; shown only when priority is mixed or ambiguous (e.g. both cost and adoption selected).

If the user answers off-script in free text, parse it into the fields and proceed rather than re-asking.

## Confirm gate (mandatory)

Echo a one-line summary of the resolved intake, then present `[Confirm]` / `[Adjust]`.

- **Confirm** → write the `intake` block to `session.json`, set `stage.status.intake = done`, auto-advance to Diagnostic.
- **Adjust** → keep prior selections sticky, let the user edit only the wrong field, then re-confirm. Never re-ask fields that were already correct.

## Guardrails

- Question count scales 0 → 6 with completeness; the confirm step is always shown, even at zero questions.
- Minimum one selection on Scope and on Priority.
- Priority is flat — no ranking, no weighting.
- Never ask about connectors, MCPs, or data sources — Diagnostic owns source resolution. A user's access/connector claim goes into `connectivity_hint` only.
- Scope is always structured (`tools[]`), never stored as prose.

## Handoff

The `intake` block feeds Diagnostic, which reads `access` (permission ceiling), `connectivity_hint` (trust-but-verify), `lookback` (metric window), and `scope.tools` (targets + categories).
