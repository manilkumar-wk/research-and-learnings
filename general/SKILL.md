---
name: monthly-jira-work-report
description: >-
  Build a monthly Jira work report for manil.kumar (or a given user) from
  Workiva Jira: tickets updated/resolved in a month, with sprint, Effort
  Points, status, and Done/Merged/Closed dates; write a markdown report and
  generic JQL. Use when the user asks for a monthly Jira report, July/August
  (etc.) tickets worked, story points report, or monthly Jira JQL.
---

# Monthly Jira Work Report

Generate a monthly work report from Workiva Jira (`jira.atl.workiva.net`) via the Atlassian MCP.

## Inputs

Ask only if missing:

| Input | Default |
|---|---|
| Month / year | Required (e.g. July 2026) |
| Jira username | `manil.kumar` |
| Output path | `<workspace>/<yyyy>-<mm>-jira-work-report.md` |

Month window: `start = YYYY-MM-01`, `end = last day of month`.

## Field mapping (Workiva)

| Report column | Jira field |
|---|---|
| Jira Id | `key` |
| Summary | `summary` |
| Sprint Details | `customfield_12020` (Sprint) — use **latest** sprint name |
| Storypoints Estimated | `customfield_10214` (**Effort Points**) |
| Status | `status.name` |
| Done / Merged / Closed Date | see below |

Parse sprint names from Greenhopper strings with `name=...`.

### Completion date

| Status | Date source |
|---|---|
| Closed, Done | `resolutiondate` (fallback: `updated` if null) |
| Merged | `updated` (Merged has no resolutiondate; status category is In Progress) |
| New, Open, In Progress, etc. | `—` |

## Workflow

### 1. Fetch candidates

Search with Atlassian MCP `jira_search_issues`. Paginate if `total > returned`.

Union of:

```jql
assignee = <user> AND updated >= "<start>" AND updated <= "<end>" ORDER BY key ASC
```

```jql
assignee = <user> AND resolved >= "<start>" AND resolved <= "<end>" ORDER BY key ASC
```

Fields: `summary`, `status`, `resolutiondate`, `updated`, `customfield_10214`, `customfield_12020`.

Do **not** use `assignee was ... DURING` alone — it returns hundreds of stale assigned tickets.

Do **not** rely on a saved filter unless the user confirms it matches the month.

### 2. Apply default filters

1. Drop tickets **Closed / Done / Merged** with completion date **before** month start.
2. Drop status **New**.
3. Drop **UpNext** sprint + status New (redundant if New is dropped; keep for clarity).
4. Keep Open / In Progress unless the user asks to remove them.

Confirm remaining counts with the user if they want further cuts.

### 3. Write markdown report

Write `<yyyy>-<mm>-jira-work-report.md`:

```markdown
# <Month YYYY> Jira Work Report

**Assignee:** <user>
**Criteria:** Tickets updated or resolved in <Month YYYY>, excluding Completed before <start>, excluding status New
**Storypoints Estimated:** Effort Points field
**Done / Merged / Closed Date:** Resolution date for Closed/Done; Merged uses last updated
**Total tickets:** <n>

| Jira Id | Summary | Sprint Details | Storypoints Estimated | Status | Done / Merged / Closed Date |
|---|---|---|---|---|---|
| ... | ... | ... | ... | ... | ... |

### Notes
- ...
```

Use `—` for empty sprint / points / date.

### 4. Provide generic JQL

Always return a **generic** (no `key in (...)`) JQL the user can paste into Jira:

```jql
assignee = <user>
AND status not in (New)
AND (
  (
    status in (Closed, Done)
    AND resolved >= "<start>"
    AND resolved <= "<end>"
  )
  OR (
    status = Merged
    AND updated >= "<start>"
    AND updated <= "<end>"
  )
  OR (
    status in (Open, "In Progress")
    AND updated >= "<start>"
    AND updated <= "<end>"
  )
)
ORDER BY key ASC
```

If the user asked to exclude Open / In Progress, drop that OR branch.

Also offer a key-based JQL only if they explicitly ask for keys.

### 5. Optional canvas

If useful, also create a canvas table under the workspace `canvases/` folder. Markdown report is required.

## Example prompts

- "July 2026 Jira work report"
- "Monthly Jira report for August 2026"
- "Same report for September for manil.kumar"

## Constraints

- Prefer Effort Points over Original story points.
- Prefer markdown in the workspace; do not commit unless asked.
- Do not invent completion dates; document Merged / missing-resolution fallbacks in Notes.
