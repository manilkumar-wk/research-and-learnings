# July 2026 Jira Work Report

**Assignee:** kumaresan.perumal  
**Criteria:** Tickets updated or resolved in July 2026, excluding Completed before 2026-07-01, excluding status New, excluding Epics  
**Storypoints Estimated:** Effort Points field  
**Done / Merged / Closed Date:** Resolution date for Closed/Done; Merged uses last updated  
**Total tickets:** 12  
**Removed (completed before July):** 1  
**Removed (status New):** 4  
**Removed (Epic):** 2  
**Added from Q3.1/Q3.2 sprint compare:** 1  

| Jira Id | Summary | Sprint Details | Storypoints Estimated | Status | Done / Merged / Closed Date |
|---|---|---|---|---|---|
| FPLAT-2999 | NNBD - form_config support: sound_null_safety flag | Forms Toolbox 2026 Q3.1 | 1 | Closed | 2026-07-08 |
| INTRISK-102172 | Ingest changes needed for serializable ui deprication | Forms Toolbox 2026 Q3.2 | 3 | Closed | 2026-07-23 |
| INTRISK-102512 | Prod Support:Error when exporting GRC report | Forms Toolbox 2026 Q3.1 | 2 | Closed | 2026-07-20 |
| INTRISK-102809 | Prod Support:formService timeout causing form open & update failures | Forms Toolbox 2026 Q3.1 | 1 | Closed | 2026-07-08 |
| INTRISK-102933 | Production Support: Owner Permission User Unable to Apply Changes in Request Permissions | Forms Toolbox 2026 Q2.6 | 3 | Closed | 2026-07-02 |
| INTRISK-103327 | Prod Support-Test form-Not Able To Export Test Form | Forms Toolbox 2026 Q3.1 | 1 | Closed | 2026-07-08 |
| INTRISK-103545 | Consume graph-form-api 3.181.0 | Forms Toolbox 2026 Q3.1 | 1 | Closed | 2026-07-14 |
| INTRISK-103724 | Prod Support: Controls experience does not load | Forms Toolbox 2026 Q3.2 | 3 | Closed | 2026-07-23 |
| INTRISK-104041 | form_cofig : Upgrade to Dart 3 - QA and Support | Forms Toolbox 2026 Q3.2 | 1 | Closed | 2026-07-28 |
| INTRISK-104052 | NNBD: framework_explorer : Upgrade to Dart 3 | Forms Toolbox 2026 Q3.2 | 1 | Closed | 2026-07-28 |
| INTRISK-104054 | NNBD: grc_universe_client : Upgrade to Dart 3 QA | Forms Toolbox 2026 Q3.2 | 1 | Closed | 2026-07-28 |
| INTRISK-104184 | Review and QA: History panel: Vertex history is not loaded in Dart 3 upgrade | Forms Toolbox 2026 Q3.2 | 1 | Closed | 2026-07-31 |

### Notes

- Removed tickets that were **Done / Merged / Closed** before **2026-07-01**.
- Removed all tickets with status **New**.
- Removed **Epic** issue types.
- No Merged tickets in this report.
- Added **INTRISK-104184** from Q3.2 sprint compare (resolved 2026-07-31; missed by `resolved <= "2026-07-31"` start-of-day boundary).

### Removed tickets (completed before July)

| Jira Id | Status | Done / Merged / Closed Date |
|---|---|---|
| INTRISK-102259 | Closed | 2026-06-16 |

### Removed tickets (status New)

| Jira Id | Summary |
|---|---|
| FPLAT-2240 | Investigation: The archive snapshot view shows create and delete icons |
| INTRISK-102293 | form-service:Rename SLOs alert name without unit info in the name |
| INTRISK-102513 | Duplicative Records in Reporting |
| INTRISK-103349 | Compliance Manager - Unable to install framework in dev |

### Removed tickets (Epic)

| Jira Id | Status | Summary |
|---|---|---|
| FPLAT-2849 | Closed | 2026-Q2 Dependabot/Syncdeps PRs |
| INTRISK-102524 | Open | Serializable UI Deprecation(SUI) |

### JQL

```jql
assignee = kumaresan.perumal
AND issuetype != Epic
AND status not in (New)
AND (
  (
    status in (Closed, Done, Resolved)
    AND resolved >= "2026-07-01"
    AND resolved < "2026-08-01"
  )
  OR (
    status = Merged
    AND updated >= "2026-07-01"
    AND updated < "2026-08-01"
  )
  OR (
    status in (Open, "In Progress", "Ready For Test")
    AND updated >= "2026-07-01"
    AND updated < "2026-08-01"
  )
)
ORDER BY key ASC
```
