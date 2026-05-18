# YAML Schema

Each `.yaml` file in this repo represents one case template. The schema below maps 1:1 onto CoPilot's `CaseTemplateCreate` Pydantic model (see `backend/app/incidents/schema/case_templates.py`).

## Top-level fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `key` | string | yes | Unique stable identifier for collision detection. Kebab-case. Example: `sysmon-event-1-process-creation`. |
| `name` | string ≤255 | yes | Becomes `CaseTemplate.name`. Shown in the CoPilot UI. |
| `description` | string | recommended | Becomes `CaseTemplate.description`. One-paragraph summary of when to use this playbook. |
| `source` | string ≤50 | recommended | Becomes `CaseTemplate.source` — the **SIEM source** the template scopes to (e.g. `wazuh`, `velociraptor`). Match exactly what arrives on `incident_management_alert.source`. Use `match` (below) to narrow further by event type. |
| `match` | object | optional | Conditional auto-apply. When present, both child keys are required. See [`match` sub-fields](#match-sub-fields). |
| `tags` | object | optional | Library-only metadata; **not** persisted into CoPilot. Drives Library-card display + filtering. |
| `tasks` | array | yes | Ordered list of `Task` objects. May be empty (rare). |

## `tags` sub-fields

| Field | Type | Notes |
|---|---|---|
| `sysmon_event_id` | int | The Sysmon EID this playbook covers. Drives card display. |
| `mitre_tactics` | string[] | MITRE tactic short names (e.g. `execution`, `defense-evasion`). Display only. |

## `match` sub-fields

When a template carries a `match` block, CoPilot fetches the raw Wazuh document for the originating alert (via the alert's asset's `index_name` + `index_id`) at auto-apply time and only applies the template if `document[field] == value`.

| Field | Type | Required (when `match` is present) | Notes |
|---|---|---|---|
| `field` | string ≤255 | yes | Name of a **flat top-level key** on the Wazuh document, e.g. `data_win_system_eventID`. Do **not** use the nested dotted path `data.win.system.eventID` — the importer matches against the flattened indexed fields. |
| `value` | string \| number \| bool | yes | Equality target. Wazuh field values come back from OpenSearch as strings (numeric IDs arrive as `"1"`, not `1`), and the parser coerces YAML scalars to strings on import — so `value: 1` and `value: "1"` behave identically. Prefer quoting for clarity. |

Half-set blocks (only `field` or only `value`) are ignored by the importer and logged as warnings — a partial condition would silently never trigger, so the parser drops the block rather than persist a misleading template.

## Selection precedence

When a case is created or an alert is linked, CoPilot runs a two-stage picker:

1. **Field-match stage.** Every template scoped to the alert's `(customer_code, source)` that has a `match` block is evaluated against the raw Wazuh document. **All matches apply** — they layer additively. A "Sysmon EID 1" template and a "high-severity Sysmon" template can both fire on the same alert and produce two task groups.
2. **Fallback stage.** If zero field-match templates matched — or the raw event couldn't be fetched (deleted index, network blip) — CoPilot falls back to the legacy tier picker (`customer+source > customer > source > global default`) and applies one template.

Templates without a `match` block participate only in step 2. This split is deliberate: a generic "wazuh global" template shouldn't drown a specific "sysmon event 1" one.

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
- `match` is all-or-nothing: half-set blocks (only `field` or only `value`) are dropped with a warning on the CoPilot side. Set both, or omit the block.
