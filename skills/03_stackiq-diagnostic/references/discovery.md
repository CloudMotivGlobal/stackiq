# StackIQ Diagnostic — stack discovery ladder

Runs once per Diagnostic pass, only when `intake.scope.discovery_mode == auto_discover` (pipeline step 1). Its job is to answer "what tools does this org actually have" without an API key, using only what the plugin already has: connected MCPs, a signed-in Chrome session, and whatever files/connectors expose spend. Read this before running discovery.

## Table of contents
- The three legs
- Reconciliation and confidence
- Triage heuristic (pipeline step 2)
- Gap handling per leg
- Interaction with named_only mode

## The three legs

Run all three; each is independent and a failure on one never blocks the others.

**Leg 1 — MCP connectors (always first, cheapest).** Enumerate every MCP connector/tool actually connected and live in this session. Each connected connector is a candidate tool: it is definitionally in use (someone authorized it), so it enters `discovered_tools[]` with `discovery_source: ["mcp_connector"]` and `confidence: confirmed` outright — no further corroboration needed. This is the fastest leg and requires no navigation.

**Leg 2 — SSO / identity-provider app catalog (via signed-in Chrome).** Most real SaaS usage in an org is gated behind SSO. Ask the user which identity provider they use if it isn't already obvious (Okta, Google Workspace, Microsoft Entra ID, or other), then navigate to its admin console's application/app-catalog view and enumerate every listed app — this is the single richest source of "what do we actually have." Paginate to completion using the same discipline as `chrome-coverage.md` (full list, not the default/starred view). Each app becomes a candidate with `discovery_source` including `"sso_catalog"`.

**Leg 3 — Expense / billing scan.** Catches tools paid for but not SSO-gated (a single admin's personal login, a free-tier-turned-paid tool, a card swipe nobody provisioned through IT). Use a connected finance/expense MCP if one exists (e.g. an accounting or spend-management connector); otherwise ask the user which expense platform or card statement to check and navigate it via signed-in Chrome, or accept an uploaded statement/export file. Extract recurring line items that match recognizable SaaS/AI vendor naming patterns (subscription cadence, known vendor name fragments). Each becomes a candidate with `discovery_source` including `"expense_scan"`.

## Reconciliation and confidence

After all three legs run (or are logged as gaps), merge candidates by normalized vendor/tool name (case-insensitive, strip Inc./legal suffixes, match common aliases like "Google Workspace" vs "GSuite") into one deduped list:

- Corroborated by **2 or more legs**, or surfaced via a **live MCP connector** → `confidence: confirmed`.
- Surfaced by **exactly one** of the SSO or expense legs alone → `confidence: probable` — flag it for verification same as any other probable finding; do not upgrade it silently later in the pipeline.
- A tool in the SSO catalog with **no matching spend** is still valid (could be free-tier, or paid annually outside the lookback window) — keep it as `probable`, do not drop it.
- A tool in **expense only**, not in SSO and not an MCP connector, is exactly the shadow-IT case this leg exists to catch — keep it, and let it flow into `access`-mode analysis as a candidate finding (unsanctioned tool with real spend).

Every reconciled entry gets `merged_into_scope: true` in `auto_discover` mode (see the SKILL's guardrail on the `named_only` vs `auto_discover` split) and proceeds to triage (pipeline step 2) exactly like a named tool.

## Triage heuristic (pipeline step 2)

Once the master tool list exists (named ∪ discovered), triage every entry before committing to a full deep-dive. Pull only the cheapest signal already on hand or one lightweight read:

- `seat` billing → provisioned/active seat count if already visible from the SSO catalog or a quick members-page glance. Flag if seats are double-digit-plus relative to `intake.scale`, or if any dormancy signal is already visible.
- `consumption` billing → last invoice/usage amount if visible from the expense leg. Flag if spend is non-trivial against scale, or the key/integration looks idle at a glance.
- `flat` billing → is there any usage signal at all. No visibility yet → flag (unknown is never assumed clean).
- `tiered` billing → current tier name if visible. Flag if the tier looks like it's above what a quick glance at usage would justify.

Anything that can't be read even at this cheap level → `flagged` by default (visibility gaps get resolved by the deep-dive, not assumed away). A tool only gets `triage_result: clean` when the cheap pass positively shows nothing worth a full crawl. `clean` tools still get a one-line entry in the output — "reviewed, no issues at triage level" — never a silent drop.

## Gap handling per leg

- No IdP identified, user doesn't have admin access to it, or Chrome hits a CAPTCHA/rate-limit/permission wall mid-crawl → gap naming `sso_catalog` as unavailable, `as_finding: true`. Continue with the other legs.
- No expense/finance connector, no accessible expense platform, and the user has no statement to upload → gap naming `expense_scan` as unavailable. Continue.
- Zero MCP connectors currently connected → not a gap by itself (it's a valid, if sparse, result) unless it's the *only* leg that ran, in which case note in `stack_summary` that coverage rests on SSO/expense alone.
- If **all three legs** fail or are declined, discovery produced nothing: still write `stack_summary.discovery_ran: true` with empty `discovery_sources_used[]`, log a top-level gap ("could not discover any tools beyond those named"), and proceed with whatever was named at intake. Never claim a full-stack audit happened when it didn't.

## Interaction with named_only mode

Discovery (this whole file) never runs under `named_only`. A tool spotted incidentally while working a named tool (e.g. an unexpected integration visible on a connected-apps page) is still worth surfacing — log it in `discovered_tools[]` with `discovery_source: ["mcp_connector"]` or whatever surfaced it, `confidence` as appropriate, and `merged_into_scope: false`. It becomes a shadow-IT discovery finding, not a fully-analyzed tool — the user asked about specific tools, not a full audit, and StackIQ doesn't expand scope on its own.
