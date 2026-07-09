# StackIQ state schema (canonical)

Two files, both under `state/` in the working directory. `session.json` is the live run; `history/run_<n>.json` is the durable per-run snapshot used for week-over-week diffing. Comments below are documentation only — real JSON must not contain them.

## Table of contents
- session.json — full shape
- Enumerated values (single source of truth)
- history/run_<n>.json — snapshot shape
- Ownership map (which skill writes which block)

## session.json

```json
{
  "session_id": "",
  "created_at": "",
  "run_number": 1,

  "problem_statement": "",

  "stage": {
    "current": "",
    "status": {
      "orchestrator": "",
      "intake": "",
      "diagnostic": "",
      "insight": "",
      "action": ""
    },
    "halt_reason": ""
  },

  "env": {
    "workdir_writable": false,
    "chrome_connected": false
  },

  "intake": {
    "scope": {
      "tools": [ { "name": "", "vendor": "", "category": "" } ],
      "breadth": "",
      "discovery_mode": ""
    },
    "access": "",
    "priority": [],
    "scale": "",
    "lookback": "",
    "notes": "",
    "connectivity_hint": ""
  },

  "diagnostic": {
    "sources_resolved": [ { "tool": "", "source": "", "granularity": "" } ],
    "stack_summary": {
      "discovery_ran": false,
      "discovery_sources_used": [],
      "discovery_sources_unavailable": [],
      "tools_named": 0,
      "tools_discovered": 0,
      "tools_triaged_clean": 0,
      "tools_deep_dived": 0
    },
    "findings": [
      {
        "id": "",
        "category": [],
        "mode": "",
        "title": "",
        "affected_tools": [],
        "severity": "",
        "confidence": "",
        "size": { "unit": "", "value": 0 },
        "comparator": "",
        "value": "",
        "evidence": [],
        "recommendation_hook": ""
      }
    ],
    "gaps": [ { "type": "", "detail": "", "as_finding": true } ],
    "discovered_tools": [
      {
        "name": "", "vendor": "", "category": "",
        "discovery_source": [], "confidence": "",
        "triage_result": "", "merged_into_scope": false
      }
    ],
    "clean_bill": false
  },

  "insight": {
    "excel_path": "",
    "ppt_path": "",
    "ordered_finding_ids": []
  },

  "action": {
    "executed":              [ { "finding_id": "", "action_taken": "", "channel": "", "date": "" } ],
    "awaiting_confirmation": [ { "finding_id": "", "action_taken": "", "channel": "", "recipient": "", "sent_date": "" } ],
    "pending_approval":      [ { "finding_id": "", "action_taken": "", "date": "" } ],
    "blocked":               [ { "finding_id": "", "action_taken": "", "blocked_reason": "", "date": "" } ]
  }
}
```

## Enumerated values (single source of truth)

- `stage.current` / `stage.status.*`: status is one of `pending | active | done | halted`; `stage.current` is one of `orchestrator | intake | diagnostic | insight | action`.
- `intake.scope.breadth`: `single_tool | category | ecosystem`.
- `intake.scope.discovery_mode`: `named_only | auto_discover`. `named_only` (default) requires ≥1 tool in `scope.tools[]`, exactly today's behavior. `auto_discover` means the user asked to find their whole stack rather than (or in addition to) naming tools; `scope.tools[]` may be empty, and Diagnostic's stack-discovery phase populates the rest. Named tools and discovered tools are never mutually exclusive — a user can name some and still ask StackIQ to find everything else.
- `intake.access`: `admin_full | limited | read_only | unsure` (permission ceiling).
- `intake.priority[]`: any of `cut_cost | lift_adoption | consolidate | de_risk` (flat, at least 1).
- `intake.scale`: `<10 | 10-50 | 50-200 | 200+`.
- `intake.lookback`: `90d | 12mo` (cost priority infers 12mo, adoption infers 90d).
- `finding.category[]`: any of `Dx | Rx | Triage | Dx+` (client-facing; drives PPT structure).
- `finding.mode`: `node | ecosystem | portfolio | cohort | spend | access` (internal; drives Action logic).
- `finding.severity`: priority-weighted, `high | medium | low`.
- `finding.confidence`: `confirmed | probable`.
- `finding.size.unit`: `$ | seats | accounts`.
- `finding.comparator`: `baseline | week_over_week`.
- `diagnostic.stack_summary.discovery_sources_used[]` / `discovery_sources_unavailable[]`: any of `mcp_connector | sso_catalog | expense_scan`.
- `discovered_tools[].discovery_source[]`: any of `mcp_connector | sso_catalog | expense_scan` — an array because a tool corroborated by more than one leg lists all of them.
- `discovered_tools[].confidence`: `confirmed | probable` — `confirmed` when surfaced by a live MCP connector or corroborated by ≥2 discovery legs; `probable` when surfaced by a single leg (e.g. SSO-only, no matching spend).
- `discovered_tools[].triage_result`: `not_triaged | clean | flagged`. `not_triaged` is a transient in-progress value only; every discovered tool must resolve to `clean` or `flagged` before the `diagnostic` block is written.
- `discovered_tools[].merged_into_scope`: boolean. `true` once a discovered tool has been folded into the same per-tool analysis loop as named tools (the `auto_discover` case). Stays `false` only when the tool was found opportunistically under `named_only` mode (incidental shadow-IT, surfaced but not analyzed — see the Diagnostic SKILL for the mode split) or when a gap prevented any further verification.
- `action` statuses: `executed | awaiting_confirmation | pending_approval | blocked` (distinct). `pending_approval` = not yet approved by the user; `awaiting_confirmation` = approved and the notify-by-email action was sent, now waiting for the recipient to confirm (or a later run to verify); `blocked` = not resolvable by approval alone; `executed` = done/confirmed.

## history/run_<n>.json

Absolute metrics captured this run — the diff source next week reads. Diagnostic writes the snapshot; Action appends the `actions` array on completion.

```json
{
  "run_number": 0,
  "timestamp": "",
  "snapshot": {
    "tools": [
      {
        "name": "",
        "billing_model": "",
        "activity_type": "",
        "origin": "",
        "metrics": {}
      }
    ]
  },
  "finding_ids": [],
  "actions": [
    { "finding_id": "", "action_taken": "", "status": "", "channel": "", "recipient": "", "approved_by_user": false, "sent_date": "", "confirmed_date": "", "date": "" }
  ]
}
```

- `billing_model`: `seat | consumption | flat | tiered`.
- `activity_type`: `login | request_volume | job_runs`.
- `origin`: `named | mcp_connector | sso_catalog | expense_scan` — how this tool entered scope this run; carried into the snapshot so next week's Diagnostic can tell a newly-discovered tool from one the user has always named.
- `metrics`: shape varies by billing model — e.g. provisioned/active/dormant seats, spend, request volume, tier. Store whatever the source exposes.

## Ownership map

| Block / file | Written by | Read by |
|---|---|---|
| `session_id`, `created_at`, `run_number`, `problem_statement`, `stage`, `env` | Orchestrator | all |
| `intake` | Intake (Skill 2) | Diagnostic, Action |
| `diagnostic`, `history/run_<n>.json` snapshot | Diagnostic (Skill 3) | Insight, Action, next run |
| `insight` | Insight (Skill 4) | Action |
| `action`, `history/run_<n>.json` actions | Action (Skill 5) | Orchestrator, next run |

Append-only discipline: no skill mutates another skill's block. Action never edits Diagnostic's finding rows — it keys its outcomes to `finding_id`. Discovery does not create an exception: Diagnostic still owns `discovered_tools[]` and `stack_summary` outright — it never reaches back to append to Intake's `scope.tools[]`, even in `auto_discover` mode. "Merging" discovered tools into scope happens functionally (they're treated identically in analysis, sizing, and the `findings[]` they generate), not by rewriting Intake's block.
