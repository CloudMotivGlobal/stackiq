# StackIQ Diagnostic — the six analysis modes

Modes are selected at runtime from the problem statement + intake, and they compose (a run can fire several). Each finding records the single `mode` it came from; cross-mode dedup (pipeline step 10) collapses duplicates afterward. Skip any mode whose preconditions aren't met and log the reason as a gap.

Modes only run on tools that reach the deep-dive stage. Under `auto_discover`, every in-scope tool is triaged first (pipeline step 2, detailed in `references/discovery.md`); a mode fires only against tools with `triage_result: flagged` (or any named tool, on a run where discovery didn't run at all). A triage-`clean` tool never enters mode analysis — it gets a one-line "reviewed, nothing found" entry instead.

## Table of contents
- node
- ecosystem
- portfolio
- cohort
- spend
- access
- Billing-model → metric map
- Confidence rules

## node — single-system utilization
Precondition: ≥1 tool with a reachable source.
Analyze utilization and configuration on one system in isolation: provisioned vs active vs dormant seats, feature adoption, config hygiene. The core run-1 workhorse — licenses-owned vs licenses-used dormancy is the classic dollar-heavy baseline finding.

## ecosystem — cross-tool entity reconciliation
Precondition: ≥2 tools that plausibly share entities.
Discover the entity types each tool masters (contacts, deals, users, tickets…), reconcile shared entities across tools, then infer the edges: system-of-record, broken or manual syncs, duplicated data. Sync/SoR inferences are `probable` unless directly observed.

## portfolio — duplicate-capability detection
Precondition: ≥2 tools (else skip, log gap).
Run parallel capability detection across the stack to find tools that do the same job (two CRMs, three schedulers). Feeds `consolidate` priority. Overlap is sized as the cheaper/dormant tool's spend.

## cohort — team × tool adoption variance
Precondition: team- or user-level activity data available (else skip, log gap).
Compare adoption across teams for the same tool to surface pockets of non-use (a seat block a department never logs into). Uses `activity_type`, not just logins.

## spend — billing forensics
Precondition: billing/invoice data reachable.
Billing-only analysis: upcoming renewals, tier mismatch (paying for a tier the usage doesn't need), plan overlap, annual-vs-monthly waste. Findings here are almost always `$`-sized and `confirmed`.

## access — hygiene and risk
Precondition: user/admin list reachable.
Orphaned accounts, offboarded users still provisioned, SSO gaps, and shadow-IT discovery (accounts/tools appearing that aren't in scope → `discovered_tools[]`). Feeds `de_risk` priority; sized in `accounts`.

## Billing-model → metric map
- `seat` → provisioned, active, dormant; cost/seat; dormant-seat dollars.
- `consumption` → request/usage volume vs committed spend; idle keys/integrations.
- `flat` → is the single subscription used at all; cancel/consolidate candidate.
- `tiered` → current tier vs usage-justified tier; right-size-down delta.

## Confidence rules
- `confirmed` — read directly off a source (a billing line, a seat count, an admin list).
- `probable` — inferred (a sync looks broken, a tool looks redundant). Must be flagged for verification in the finding and never upgraded to fact downstream.
