---
name: stackiq-diagnostic
description: >
  StackIQ diagnostic (Skill 3 of 5). Auto-invoked by the StackIQ orchestrator after intake confirms;
  not normally triggered directly. The read-only analysis engine: for each in-scope tool it verifies
  connectivity (trust-but-verify), resolves the best available data source down a ladder (live MCP,
  then user files, then signed-in Chrome, no web search), profiles the tool's billing model and activity
  type, then runs one or more of six analysis modes (node, ecosystem, portfolio, cohort, spend, access)
  chosen from the problem statement and intake. Sizes every finding in dollars, seats, or accounts
  against the scale anchor, diffs against last week's snapshot when present, dedupes across modes,
  classifies each finding, and always emits output — no findings is a first-class clean bill of health.
  Surfaces unreachable tools and un-runnable modes as gaps, and flags shadow-IT discoveries. STRICTLY
  READ-ONLY — never clicks destructive controls. Writes the diagnostic block plus a durable run snapshot.
---

# StackIQ Diagnostic (Skill 3 of 5)

The analysis core. It reads, navigates, sizes, and classifies — and writes nothing but findings and a snapshot. It never remediates; Skill 5 is the sole writer.

## On entry

Read from `state/session.json`: `problem_statement`, the full `intake` block, and `run_number`. Read `state/history/` for any prior run snapshots (the diff source). The state field shapes and allowed values are in the orchestrator's `references/state-schema.md`.

## Pipeline (run in order)

1. **Connector validation.** For each `intake.scope.tools[]`, verify whether a live MCP / connector is actually reachable. Treat `intake.connectivity_hint` as a claim to verify, never as truth.
2. **Source resolution (per tool, independent).** Walk the ladder and land on the highest available rung for each tool: live MCP → user-provided files (uploads) → signed-in Chrome (read-only navigation). No web search. If a tool has no reachable source, log it as a gap-finding and continue — do not halt. Record each resolution in `sources_resolved[]` with `granularity` (`row | aggregate | summary`).
   - **When the source is Chrome, crawl the whole app — not a sample.** Visit every relevant admin section/tab and paginate to the end of every list. Do not stop at the first screen or the default view. Follow the full coverage checklist in `references/chrome-coverage.md`, and only mark a tool `done` once every applicable section has been visited and every record page consumed. A section that exists but wasn't reached is a `gap`, not a silent omission.
3. **Per-tool profiling.** Detect the `billing_model` (`seat | consumption | flat | tiered`) — this selects the metric set. Detect the `activity_type` (`login | request_volume | job_runs`); do not assume logins are the only signal.
4. **Mode resolution.** From `problem_statement` + `intake`, classify the run into one or more of the six modes below. Modes compose. Skip any mode that can't run (e.g. `consolidate`/portfolio needs ≥2 tools; `cohort` needs team-level data) and record why as a gap.
5. **Analysis per mode.** Run each selected mode. Full mode definitions, signals, and confidence rules are in `references/modes.md` — read it before running analysis.
6. **Run-1 vs run-n framing.** Run 1 = absolute T0 snapshot (licenses vs used, dormancy, tier mismatch — dollar-heavy, self-evident); set each finding's `comparator` to `baseline`. Run 2+ = week-over-week deltas vs the prior snapshot; set `comparator` to `week_over_week` and phrase `value` as a delta (e.g. "+1 idle seat").
7. **Sizing.** Size every finding in `$`, `seats`, or `accounts` against the `intake.scale` anchor. If a finding can't be sized, flag it qualitative rather than inventing a number.
8. **Cross-mode dedup.** If the same underlying issue surfaces in multiple modes, collapse it to one finding framed as dormancy/waste.
9. **Classify + severity.** Assign each finding one or more client-facing categories (`Dx | Rx | Triage | Dx+`) and a priority-weighted `severity`. Keep the internal `mode` on the finding for Skill 5.
10. **Always emit.** Write the `diagnostic` block regardless of finding count. Zero findings → set `clean_bill: true` (a success state, not a failure).

## Outputs

- Append the `diagnostic` block to `session.json`: `findings[]`, `gaps[]`, `discovered_tools[]`, `sources_resolved[]`, `clean_bill`.
- Write `state/history/run_<n>.json` with the absolute `snapshot.tools[]` metrics and the `finding_ids[]` raised this run. This is what next week diffs against — capture it even on a clean bill.

Each finding carries: `id`, `category[]`, `mode`, `title`, `affected_tools[]`, `severity`, `confidence` (`confirmed | probable`), `size`, `comparator`, `value`, `evidence[]` (raw counts/diffs/source refs), and `recommendation_hook` (a seed for Skill 5).

## Guardrails

- **Strictly read-only.** Read and navigate only. Never click destructive or state-changing controls — cancel, delete, downgrade, revoke all belong to Skill 5.
- **Full Chrome coverage, never partial.** In Chrome, exhaust every relevant tab/section and paginate every list to completion before a tool is `done` (see `references/chrome-coverage.md`). Never analyze from a single default screen, a first page of results, or half the app — partial coverage silently undercounts seats, spend, and accounts. Any section reached-but-incomplete or blocked becomes an explicit `gap`.
- **No web search.** Sources are MCP, files, and signed-in Chrome only.
- **Gaps are findings, not silent errors.** Unreachable tools and un-runnable modes go into `gaps[]` (with `as_finding: true`), never swallowed.
- **Probable stays probable.** Flag inferred findings (`confidence: probable`) for verification; never assert them as hard fact.
- **Shadow-IT is flagged, not adopted.** Tools discovered outside scope go into `discovered_tools[]` as discovery findings — do not auto-add them to scope.
- **Output regardless of count** — clean bill is a first-class result.

## Handoff

The `diagnostic` block feeds Insight (Skill 4), which orders findings by dollar impact and renders the Excel + PPT using `category` and `confidence`. `history/run_<n>.json` feeds next week's Diagnostic. `mode` + `evidence` + `recommendation_hook` feed Skill 5's action logic.
