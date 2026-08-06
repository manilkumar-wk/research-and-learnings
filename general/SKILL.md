---
name: monthly-jira-work-report
description: >-
  Build a monthly Jira work report for a person (self or teammate) from Workiva
  Jira: tickets updated/resolved in a month, with sprint, Effort Points, status,
  and Done/Merged/Closed dates; write markdown + CSV and generic JQL. Use when
  the user asks for a monthly Jira report for themselves or a team member by
  name/username, July/August (etc.) tickets worked, story points report, or
  monthly Jira JQL.
---

# Monthly Jira Work Report

Generate a monthly work report from Workiva Jira (`jira.atl.workiva.net`) via the Atlassian MCP.

## Inputs

| Input | Required | Default / notes |
|---|---|---|
| Month + year | Yes | e.g. July 2026 |
| Person | No | Default `manil.kumar`. Accept **Jira username** (`jane.doe`) **or display name** (`Jane Doe`) |
| Output formats | No | Markdown + CSV |
| Extra filters | No | Same defaults as below unless user overrides |

Month window: `start = YYYY-MM-01`, `end = last day of month`.

### Resolve teammate identity

If the user gives a display name (not `first.last`):

1. Call Atlassian MCP `jira_search_users` with that name.
2. Pick the matching active Workiva user; confirm username if ambiguous.
3. Use the returned **username** (`name` field) in all JQL (`assignee = <username>`).

If they give a username already, use it directly.

Output paths (include username so team reports don’t collide):

- `<workspace>/<yyyy>-<mm>-jira-work-report-<username>.md`
- `<workspace>/<yyyy>-<mm>-jira-work-report-<username>.csv`

For the default user `manil.kumar`, also allow short names `july-2026-jira-work-report.md` / `.csv` if the user prefers.

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
| Merged | `updated` (Merged has no resolutiondate) |
| New, Open, In Progress, Ready For Test, etc. | `—` |

## Workflow

### 1. Fetch candidates

Search with Atlassian MCP `jira_search_issues`. Paginate if `total > returned`.

Union of:

```jql
assignee = <username> AND updated >= "<start>" AND updated <= "<end>" ORDER BY key ASC
```

```jql
assignee = <username> AND resolved >= "<start>" AND resolved <= "<end>" ORDER BY key ASC
```

Fields: `summary`, `status`, `resolutiondate`, `updated`, `customfield_10214`, `customfield_12020`.

Do **not** use `assignee was ... DURING` alone — it returns hundreds of stale assigned tickets.

### 2. Apply default filters

1. Drop tickets **Closed / Done / Merged** with completion date **before** month start.
2. Drop status **New**.
3. Drop **UpNext** sprint + status New (redundant if New is dropped).
4. Keep Open / In Progress / Ready For Test unless the user asks to remove them.

### 3. Write markdown + CSV

Markdown columns:

`Jira Id | Summary | Sprint Details | Storypoints Estimated | Status | Done / Merged / Closed Date`

CSV: same columns, empty string instead of `—`.

Header must show **Assignee:** `<username>` (and display name if known).

### 4. Provide generic JQL

```jql
assignee = <username>
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
    status in (Open, "In Progress", "Ready For Test")
    AND updated >= "<start>"
    AND updated <= "<end>"
  )
)
ORDER BY key ASC
```

Drop Open / In Progress / Ready For Test branch if the user asked to exclude incomplete work.

## Example prompts

- "July 2026 Jira work report"
- "August 2026 Jira report for jane.doe"
- "Monthly Jira report for July 2026 for Jane Doe"
- "Same report for September for my teammate Alex Smith"

## Constraints

- Prefer Effort Points over Original story points.
- Prefer markdown + CSV in the workspace; do not commit unless asked.
- Do not invent completion dates; document Merged / missing-resolution fallbacks in Notes.
- One person per report run (or run sequentially if the user lists multiple teammates).
