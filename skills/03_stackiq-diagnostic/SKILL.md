---
name: stackiq-diagnostic
description: >
  StackIQ diagnostic (Skill 3 of 5). Auto-invoked after intake confirms; not normally triggered
  directly. Read-only analysis engine. When discovery_mode is auto_discover, first runs stack
  discovery — reconciling MCP connectors, the SSO app catalog (via Chrome), and expense/billing
  data into one tool list merged with named tools — then triages every tool with a cheap pass
  before running the full six-mode analysis only on flagged tools. Per tool: verifies
  connectivity, resolves the best source (MCP, files, Chrome; no web search), profiles billing
  model and activity type, runs applicable modes (node, ecosystem, portfolio, cohort, spend,
  access). Sizes findings in dollars/seats/accounts, diffs week-over-week, dedupes, classifies,
  and always emits ONE consolidated output — zero findings is a clean bill. Surfaces gaps and
  shadow-IT. STRICTLY READ-ONLY. Writes the diagnostic block plus a run snapshot.
---

# StackIQ Diagnostic (Skill 3 of 5)

The analysis core. It reads, navigates, sizes, and classifies — and writes nothing but findings and a snapshot. It never remediates; Skill 5 is the sole writer.

## On entry

Read from `state/session.json`: `problem_statement`, the full `intake` block (including `scope.discovery_mode`), and `run_number`. Read `state/history/` for any prior run snapshots (the diff source). The state field shapes and allowed values are in the orchestrator's `references/state-schema.md`.

## Pipeline (run in order)

1. **Stack discovery — conditional.** Runs only when `intake.scope.discovery_mode == auto_discover`. Skip straight to step 3 under `named_only` (nothing here changes for a named-tool run). Full ladder, reconciliation rules, and confidence scoring are in `references/discovery.md` — read it before running this step. In short: enumerate connected MCP connectors (cheapest, always first — a live connector is definitionally a tool in use), crawl the org's SSO/identity-provider app catalog via signed-in Chrome, and scan expense/billing data for recurring SaaS/AI charges; reconcile all three legs into one deduped list in `discovered_tools[]`, corroborated entries marked `confidence: confirmed`, single-leg entries `probable`. A leg that can't be reached (no IdP access, no expense source, user declines) is a `gap` naming which leg was skipped — never imply full-stack coverage you didn't actually get. From here, "in-scope tools" means `intake.scope.tools[]` **union** `discovered_tools[]`, deduped by name/vendor — discovered tools are folded into the same per-tool loop below, not held to a separate track.
2. **Triage — all in-scope tools, before deep analysis.** For every tool (named or discovered), pull only the cheapest available signal — plan tier and seat/user count if already visible from discovery metadata, or one lightweight billing/MCP read otherwise. Set `triage_result: flagged` when spend or seats look non-trivial against `intake.scale`, a dormancy/idle signal is already visible at this cheap pass, or the tool has zero visibility yet (unknown defaults to flagged, never to clean). Otherwise `triage_result: clean`. This is a coarse pre-filter, not the six analysis modes — its only job is to stop a large stack from forcing a full Chrome crawl on every single tool. A tool that triages `clean` still gets a minimal entry in the output (reviewed, nothing found) — it is never silently dropped.
3. **Connector validation.** For each tool proceeding to deep-dive (i.e. `triage_result: flagged`, or any named tool when discovery didn't run), verify whether a live MCP / connector is actually reachable. Treat `intake.connectivity_hint` as a claim to verify, never as truth.
4. **Source resolution (per tool, independent).** Walk the ladder and land on the highest available rung for each tool: live MCP → user-provided files (uploads) → signed-in Chrome (read-only navigation). No web search. If a tool has no reachable source, log it as a gap-finding and continue — do not halt. Record each resolution in `sources_resolved[]` with `granularity` (`row | aggregate | summary`).
   - **When the source is Chrome, crawl the whole app — not a sample.** Visit every relevant admin section/tab and paginate to the end of every list. Do not stop at the first screen or the default view. Follow the full coverage checklist in `references/chrome-coverage.md`, and only mark a tool `done` once every applicable section has been visited and every record page consumed. A section that exists but wasn't reached is a `gap`, not a silent omission.
5. **Per-tool profiling.** Detect the `billing_model` (`seat | consumption | flat | tiered`) — this selects the metric set. Detect the `activity_type` (`login | request_volume | job_runs`); do not assume logins are the only signal.
6. **Mode resolution.** From `problem_statement` + `intake`, classify the run into one or more of the six modes below. Modes compose. Skip any mode that can't run (e.g. `consolidate`/portfolio needs ≥2 tools; `cohort` needs team-level data) and record why as a gap.
7. **Analysis per mode.** Run each selected mode. Full mode definitions, signals, and confidence rules are in `references/modes.md` — read it before running analysis.
8. **Run-1 vs run-n framing.** Run 1 = absolute T0 snapshot (licenses vs used, dormancy, tier mismatch — dollar-heavy, self-evident); set each finding's `comparator` to `baseline`. Run 2+ = week-over-week deltas vs the prior snapshot; set `comparator` to `week_over_week` and phrase `value` as a delta (e.g. "+1 idle seat").
9. **Sizing.** Size every finding in `$`, `seats`, or `accounts` against the `intake.scale` anchor. If a finding can't be sized, flag it qualitative rather than inventing a number.
10. **Cross-mode dedup.** If the same underlying issue surfaces in multiple modes, collapse it to one finding framed as dormancy/waste.
11. **Classify + severity.** Assign each finding one or more client-facing categories (`Dx | Rx | Triage | Dx+`) and a priority-weighted `severity`. Keep the internal `mode` on the finding for Skill 5.
12. **Always emit — once, for the whole run.** Process every in-scope tool to completion (all triage, all flagged deep-dives) before writing anything. Write the `diagnostic` block regardless of finding count, covering every tool in a single pass — never emit a partial or per-tool interim block; even mid-run status updates to the user should stay conversational, not written to state. Zero findings → set `clean_bill: true` (a success state, not a failure).

## Outputs

- Append the `diagnostic` block to `session.json`: `sources_resolved[]`, `stack_summary`, `findings[]`, `gaps[]`, `discovered_tools[]`, `clean_bill`.
- `stack_summary` (only meaningful when discovery ran, but always present): `discovery_ran`, `discovery_sources_used[]`, `discovery_sources_unavailable[]`, `tools_named`, `tools_discovered`, `tools_triaged_clean`, `tools_deep_dived`. This is what lets Insight answer "did you actually check my whole stack."
- `discovered_tools[]` entries: `{name, vendor, category, discovery_source[], confidence, triage_result, merged_into_scope}`. See `references/discovery.md` for how each field is set.
- Write `state/history/run_<n>.json` with the absolute `snapshot.tools[]` metrics (including each tool's `origin`: `named | mcp_connector | sso_catalog | expense_scan`) and the `finding_ids[]` raised this run. This is what next week diffs against — capture it even on a clean bill.

Each finding carries: `id`, `category[]`, `mode`, `title`, `affected_tools[]`, `severity`, `confidence` (`confirmed | probable`), `size`, `comparator`, `value`, `evidence[]` (raw counts/diffs/source refs), and `recommendation_hook` (a seed for Skill 5).

## Guardrails

- **Strictly read-only.** Read and navigate only. Never click destructive or state-changing controls — cancel, delete, downgrade, revoke all belong to Skill 5.
- **Discovery is opt-in, never ambient.** Run the stack-discovery phase only when `intake.scope.discovery_mode == auto_discover`. Do not crawl an SSO admin console or expense platform just because a run has few named tools — those are sensitive admin surfaces beyond what was asked, and touching them uninvited is a scope violation, not a favor.
- **Triage before deep-dive, always, under discovery.** A tool never goes straight to the full six-mode + Chrome-crawl treatment without first resolving `triage_result`. This is what keeps a 40-tool stack from forcing 40 full crawls.
- **One consolidated emission.** The whole stack — discovery, triage, every deep-dive — resolves before the `diagnostic` block is written, exactly once. No partial, per-tool interim state writes.
- **Full Chrome coverage, never partial.** In Chrome, exhaust every relevant tab/section and paginate every list to completion before a tool is `done` (see `references/chrome-coverage.md`). Never analyze from a single default screen, a first page of results, or half the app — partial coverage silently undercounts seats, spend, and accounts. Any section reached-but-incomplete or blocked becomes an explicit `gap`.
- **No web search.** Sources are MCP, files, and signed-in Chrome only — discovery legs included.
- **Gaps are findings, not silent errors.** Unreachable tools, un-runnable modes, and unreachable discovery legs go into `gaps[]` (with `as_finding: true`), never swallowed.
- **Probable stays probable.** Flag inferred findings (`confidence: probable`) for verification; never assert them as hard fact.
- **Shadow-IT vs. merged discovery — the mode split.** `discovered_tools[]` means two different things depending on `discovery_mode`. Under `named_only`, a tool stumbled on incidentally (e.g. an unexpected app spotted during a Chrome pass) is opportunistic shadow-IT: log it in `discovered_tools[]` with `merged_into_scope: false`, surface it as a discovery finding, but do not run it through profiling/mode analysis — the user didn't ask for it. Under `auto_discover`, discovery is the point: every tool the ladder finds gets `merged_into_scope: true` and the full triage → (maybe) deep-dive treatment like any named tool. Never apply the `named_only` (flag-only) treatment silently during an `auto_discover` run, or vice versa.
- **Output regardless of count** — clean bill is a first-class result.

## Handoff

The `diagnostic` block feeds Insight (Skill 4), which orders findings by dollar impact, renders the Excel + PPT using `category` and `confidence`, and renders a coverage section from `stack_summary`. `history/run_<n>.json` feeds next week's Diagnostic — including tool `origin`, so a tool discovered this week is recognized as already-known next week rather than re-surfacing as new. `mode` + `evidence` + `recommendation_hook` feed Skill 5's action logic.
