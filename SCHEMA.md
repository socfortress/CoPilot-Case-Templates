# YAML Schema

Each `.yaml` file in this repo represents one case template. The schema below maps 1:1 onto CoPilot's `CaseTemplateCreate` Pydantic model (see `backend/app/incidents/schema/case_templates.py`).

## Top-level fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `key` | string | yes | Unique stable identifier for collision detection. Kebab-case. Example: `sysmon-event-1-process-creation`. |
| `name` | string ≤255 | yes | Becomes `CaseTemplate.name`. Shown in the CoPilot UI. |
| `description` | string | recommended | Becomes `CaseTemplate.description`. One-paragraph summary of when to use this playbook. |
| `source` | string ≤50 | recommended | Becomes `CaseTemplate.source`. Convention: `sysmon_<N>` for Sysmon event playbooks, freeform otherwise. Used for future alert-driven auto-selection. |
| `tags` | object | optional | Library-only metadata; **not** persisted into CoPilot. Drives Library-card display + filtering. |
| `tasks` | array | yes | Ordered list of `Task` objects. May be empty (rare). |

## `tags` sub-fields

| Field | Type | Notes |
|---|---|---|
| `sysmon_event_id` | int | The Sysmon EID this playbook covers. Drives card display. |
| `mitre_tactics` | string[] | MITRE tactic short names (e.g. `execution`, `defense-evasion`). Display only. |

## `Task` fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `title` | string ≤500 | yes | Shown as the task headline. |
| `description` | string | optional | One-sentence summary. |
| `guidelines` | string | optional | Multi-line markdown. The analyst's step-by-step walkthrough. |
| `mandatory` | bool | optional, default `false` | If true, NOT_NECESSARY status is rejected and closing the case with this task incomplete triggers a soft warning. |
| `order_index` | int ≥0 | yes | Render order. Start at 0; gaps are fine. |

## Validation rules

- File must be valid YAML 1.2.
- Unknown top-level fields are ignored by the importer (forward-compat).
- `key` must be unique across the entire repo.
- `tasks` array preserves order, but `order_index` is the canonical sort key on import.
