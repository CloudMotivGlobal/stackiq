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
      "breadth": ""
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
    "discovered_tools": [],
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
- `metrics`: shape varies by billing model — e.g. provisioned/active/dormant seats, spend, request volume, tier. Store whatever the source exposes.

## Ownership map

| Block / file | Written by | Read by |
|---|---|---|
| `session_id`, `created_at`, `run_number`, `problem_statement`, `stage`, `env` | Orchestrator | all |
| `intake` | Intake (Skill 2) | Diagnostic, Action |
| `diagnostic`, `history/run_<n>.json` snapshot | Diagnostic (Skill 3) | Insight, Action, next run |
| `insight` | Insight (Skill 4) | Action |
| `action`, `history/run_<n>.json` actions | Action (Skill 5) | Orchestrator, next run |

Append-only discipline: no skill mutates another skill's block. Action never edits Diagnostic's finding rows — it keys its outcomes to `finding_id`.
