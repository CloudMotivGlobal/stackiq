---
name: stackiq-intake
description: >
  StackIQ intake (Skill 2 of 5). Auto-invoked by the orchestrator after the problem statement is
  captured; not normally triggered directly. Turns the free-text problem statement into a
  structured intake block covering scope (tools, disambiguated, named and/or auto-discovered),
  access ceiling, priority, seat scale, and lookback window. Parses the statement to pre-fill
  fields, detects whole-stack language ("all my tools", "my whole stack") to set discovery_mode,
  then asks only missing core questions as one tappable batch (scope multi-select with an optional
  "find everything else" toggle, access, priority, scale, optional notes, lookback if priority is
  mixed), echoes a one-line summary, and requires explicit Confirm before writing state. Question
  count scales 0 to 6 by completeness. Never asks about connectors — Diagnostic owns that. Writes
  intake to state/session.json and hands off to Diagnostic.
---

# StackIQ Intake (Skill 2 of 5)

Converts the orchestrator's free-text `problem_statement` into a structured `intake` block. Ask as little as possible: infer first, ask only what's missing, always confirm.

## On entry

Read `state/session.json → problem_statement`. Parse it to pre-fill any field it clearly implies (a named tool → scope; "we're overpaying" → priority `cut_cost` → lookback `12mo`; "nobody uses it" → priority `lift_adoption` → lookback `90d`; a team size → scale).

Also parse for **discovery intent** — language asking StackIQ to find the tools itself rather than (or in addition to) naming them: "all my tools," "my whole stack," "everything we're using," "what tools do we even have," "full audit," "shadow IT," "find every tool." Any of these → pre-fill `scope.discovery_mode = auto_discover`. Absent that language, default `scope.discovery_mode = named_only` — naming tools is still the common case and nothing changes for it. Discovery and naming are additive, not a fork: a statement like "audit HubSpot and find anything else we're paying for" pre-fills both a named tool *and* `auto_discover`.

## Fields to produce

Write these into `intake` (see the orchestrator's `references/state-schema.md` for the exact shape and allowed values):

- `scope`: `tools[]` (each `{name, vendor, category}`), `breadth` (`single_tool | category | ecosystem`), and `discovery_mode` (`named_only | auto_discover`). `tools[]` needs at least 1 entry under `named_only`; it may be empty under `auto_discover` (Diagnostic fills the rest in).
- `access`: `admin_full | limited | read_only | unsure` — the permission ceiling.
- `priority[]`: any of `cut_cost | lift_adoption | consolidate | de_risk` — flat, at least 1.
- `scale`: `<10 | 10-50 | 50-200 | 200+` (SMB seat bands).
- `lookback`: `90d | 12mo` — inferred from priority (cost → 12mo, adoption → 90d).
- `notes`: optional free text.
- `connectivity_hint`: if the user claims they have access/a connector, record the claim here. It is NOT source of truth — Diagnostic verifies.

## Completeness check, then ask

Evaluate the four core fields — Scope, Access, Priority, Scale. `discovery_mode` is resolved alongside Scope, not counted as a fifth blocking field — it's either inferred from the problem statement or defaulted to `named_only`, and only surfaces as a question when Scope itself is too vague to infer (see Q1). Then:

- **All four inferable** → skip straight to the confirm step.
- **Some missing** → render only the missing questions.
- **Too vague to infer Scope or Priority** → offer a generic fallback option set for those.

Present missing questions as one tappable batch, single step, with inferred values pre-selected:

- **Q1 Scope** — multi-select, adaptive. Include disambiguation: if the user typed an ambiguous name (e.g. "Monday"), confirm which entity — the product (Monday.com), a vendor, or a whole category. Always include one more option alongside the named tools, pre-checked when `discovery_mode` was already inferred as `auto_discover`: **"Also find every other tool I'm using"** — checking it (or leaving it checked) sets `auto_discover`; leaving it unchecked with ≥1 tool selected keeps `named_only`. If the user selects zero named tools, that option must default to checked (there is otherwise nothing to audit) and the copy adapts to "I don't know all my tools — find them for me."
- **Q2 Access** — single-select.
- **Q3 Priority** — multi-select, ≥1.
- **Q4 Scale** (seat band) — single-select.
- **Q5 Additional info** — free text, optional.
- **Q6 Lookback** — inferred from priority; shown only when priority is mixed or ambiguous (e.g. both cost and adoption selected).

If the user answers off-script in free text, parse it into the fields and proceed rather than re-asking.

## Confirm gate (mandatory)

Echo a one-line summary of the resolved intake — when `discovery_mode = auto_discover`, say so plainly, e.g. "Scope: HubSpot, Calendly, plus your whole stack — I'll find the rest." Then present `[Confirm]` / `[Adjust]`.

- **Confirm** → write the `intake` block to `session.json`, set `stage.status.intake = done`, auto-advance to Diagnostic.
- **Adjust** → keep prior selections sticky, let the user edit only the wrong field, then re-confirm. Never re-ask fields that were already correct.

## Guardrails

- Question count scales 0 → 6 with completeness; the confirm step is always shown, even at zero questions.
- Minimum one selection on Priority, always. Minimum one selection on Scope (a named tool or the "find everything else" toggle) — the toggle satisfies this on its own when no tools are named.
- Priority is flat — no ranking, no weighting.
- Never ask about connectors, MCPs, or data sources — Diagnostic owns source resolution, including which discovery sources (MCP connectors, SSO catalog, expense data) it can actually reach. A user's access/connector claim goes into `connectivity_hint` only.
- Scope is always structured (`tools[]`), never stored as prose. `discovery_mode` is always resolved to exactly one of `named_only` / `auto_discover` before the confirm gate — never left implicit.

## Handoff

The `intake` block feeds Diagnostic, which reads `access` (permission ceiling), `connectivity_hint` (trust-but-verify), `lookback` (metric window), `scope.tools` (named targets + categories), and `scope.discovery_mode` (whether to run the stack-discovery phase before the per-tool pipeline).
