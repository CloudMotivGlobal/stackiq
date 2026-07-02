# StackIQ Diagnostic — Chrome full-coverage checklist

Read-only rule from the SKILL still applies: navigate and read, never click destructive controls. The goal here is *completeness* — Chrome runs undercount when they stop at the first screen. A tool sourced via Chrome is only `done` when every applicable section has been visited and every list paginated to its end.

## Table of contents
- Coverage principle
- Sections to visit per tool
- Pagination and completeness
- Per-tool coverage ledger
- Stop conditions and gaps

## Coverage principle
Enumerate the app's own navigation before reading anything: capture the full left-nav / settings menu / admin sidebar, build the list of relevant sections, then visit each one. Do not rely on the default landing view — it usually shows a summary, not the underlying records. Half-coverage (some tabs, first page only) is the failure this checklist exists to prevent.

## Sections to visit per tool
Visit every section that exists for the tool; skip only those genuinely absent. Map each to the metrics it yields:

- **Members / Users / Seats** — provisioned vs active vs invited-pending; per-user last-active. Drives seat dormancy.
- **Billing / Plan / Subscription** — current plan, tier, seat count paid, renewal date, annual vs monthly. Drives spend and tier-mismatch.
- **Usage / Analytics / Activity** — logins, request/API volume, job runs (use the tool's real `activity_type`, not just logins).
- **Integrations / Connected apps / API keys** — active vs idle integrations and keys. Drives consumption waste and ecosystem edges.
- **Teams / Groups / Workspaces** — per-team membership for cohort adoption variance.
- **Admin / Security / SSO / Roles** — orphaned/offboarded accounts, SSO gaps, guest/external users. Drives access findings and shadow-IT discovery.
- **Audit log / Directory** — last-activity truth when the members page lacks it.

Multi-workspace / multi-account tools: repeat the full section pass for **each** workspace, org, or sub-account — do not analyze only the first one.

## Pagination and completeness
- Paginate every list to the last page (click Next / load-more / infinite-scroll until exhausted). Never read only page 1.
- Prefer "show N per page = max" or an export/CSV view when the app offers one — it is the most reliable way to capture the whole set.
- Cross-check counts: if the Billing page says 50 seats but Members shows 27 rows, you have not loaded them all — keep paginating until the row count reconciles (or record the discrepancy as evidence).
- Expand collapsed groups, filters defaulted to "active only," and hidden/archived toggles — deactivated-but-still-billed users hide there.

## Per-tool coverage ledger
For each Chrome-sourced tool, track which sections were fully consumed. Record it in `sources_resolved[]` (e.g. `granularity: "row"` only when full rows were paginated; `aggregate` when only summary counts were reachable). A tool is `done` only when the applicable sections above are each either fully read or logged as a gap. Do not advance to the next tool with sections still unvisited.

## Stop conditions and gaps
- A section that exists but couldn't be fully read (permission wall, page won't paginate, count won't reconcile) → `gaps[]` with `as_finding: true`, naming the section and what's missing. Never drop it silently.
- Hard-stop and surface as a gap on CAPTCHA, rate-limit, or "unusual activity" prompts — do not push through.
- Reaching the access ceiling (a section requires admin the user doesn't have) → gap, not a guess.
