# Product bugs — July 2026

*Generated 2026-08-03 from Workiva Jira (`jira.atl.workiva.net`) via MCP.*

## Scope

- **Time window:** 2026-07-01 00:00 UTC → 2026-08-01 00:00 UTC (last complete calendar month).
- **Projects:** all — no project filter applied.
- **Team scope:** tickets labelled `toolbox` (Manil's team-scope label, present on 95 %+ of your team's bug tickets).
- **Issue type:** `Bug`.
- **Exclusions applied to `Bug` universe:** InfoSec scanner findings (labels `InfoSec:Semgrep`, `InfoSec:Grype`, `InfoSec:Dependabot`, `InfoSec:Vulnerability`) — these are compliance/security findings, not product bugs.
- **Exclusions applied to `closed`-bucket counting:** tickets closed with resolution in `{Duplicate, Cannot Reproduce, Not a Bug, Invalid, Won't Fix, Won't Do, Not our Bug}` — **1 ticket excluded in July** on this rule; they are listed separately in `july-ticket-detail.csv` under `kind=closed_excluded`.

## Executive summary

- **15 product bugs created** and **11 closed** in July 2026 (excluding InfoSec / cancelled / duplicate). Net backlog change: **+4**.
- **6 reopens** in July — reopen rate **54.5%** of tickets closed in the month (6/11). This is high and is the single biggest concern in the data; reopens are concentrated in early July (4 of 6 fixed before Jul 10, then reopened / re-closed within days).
- Median resolution time **2.3 d**, mean **13.3 d** (across the 11 bugs closed in July). Resolution is fast for the actual fixes; slower for the tickets still open (see aging).
- **6 of the 15 bugs created in July remain open** — carry-over rate 40.0%.
- **Module in the crosshairs:** `sa-tools-data-modeler` — 4 new / 1 closed in July (top of the create+close volume ranking).
- **Aging concern:** 109 currently-open bugs are >60 days old, out of 128 open bugs total. This is 85.2% of the backlog.
- **SLA %:** N/A — INTRISK is a software project (not Service Desk), so no first-response SLA policy is applied. Details in Metric 12.

## Monthly comparison table

| Metric | July 2026 |
|---|---|
| Created | **15** |
| Closed (excluded from closed: 1) | **11** |
| Net backlog change | **+4** |
| Mean first-response time | 66.7 h |
| Median first-response time | 66.7 h |
| Mean resolution time | 13.3 d |
| Median resolution time | 2.3 d |
| Reopens (rate) | **6** (54.5%) |
| Still open from July | **6** (40.0%) |
| Currently-open backlog (all-time) | 128 |
| SLA % resolved within SLA | N/A (no SLA policy on INTRISK) |

## 1. Created
- **Total: 15** — see `july-ticket-detail.csv` (kind = `created_in_july`).

## 2. Closed / Resolved
- **Total counted: 11** — see `july-ticket-detail.csv` (kind = `closed_in_july_only` or in the created rows with a non-empty resolution).
- **Excluded from closed count**: 1 (INTRISK-103418 (Duplicate)).

## 3. Net backlog change: **+4** (created 15 − closed 11)

## 4. Ticket count by module / component

| Module | Created in Jul | Closed in Jul | Volume |
|---|---:|---:|---:|
| `sa-tools-data-modeler` | 4 | 1 | 5 |
| `graph_app` | 1 | 3 | 4 |
| `sa-tools-changeset-service` | 2 | 2 | 4 |
| `Unassigned` | 3 | 1 | 4 |
| `grc_universe_client` | 2 | 1 | 3 |
| `form-service` | 1 | 1 | 2 |
| `audit` | 1 | 1 | 2 |
| `grc-services` | 1 | 0 | 1 |
| `2026-classic-grc` | 0 | 1 | 1 |

*Module rollup rule: Jira `component` (first component) if present; otherwise a label matching `sa-tools-*` / `graph-*` / `grc-*` / `form-*`; else `Unassigned`.*

## 5. Average & median first-response time
- **Method:** SLA custom fields (`customfield_21020`, `_19321`, `_35425`, `_21925`, `_10216`, `_22826`) checked first — **all null on every sampled ticket** (INTRISK is not a Service Desk project). Falls back to **time from `created` to first non-bot comment written by a user other than the reporter**, skipping automation authors (btr-, jenkins, github-actions, rosie, blackduck, renovate, dependabot*).
- **Mean:** 66.7 h | **Median:** 66.7 h (measured on 2 / 15 tickets — 13 had no qualifying comment).

## 6. Average & median resolution time
- **Mean:** 13.3 d | **Median:** 2.3 d (across 11 bugs closed in July, computed as `resolutiondate − created`, calendar days).

## 7. Ticket count by priority

| Priority | Created in Jul | Closed in Jul |
|---|---:|---:|
| Blocker | 1 | 0 |
| Major | 8 | 9 |
| Minor | 6 | 2 |

## 8. Current status breakdown (of tickets created in July)

| Status | Count |
|---|---:|
| Closed | 9 |
| New | 3 |
| In Progress | 2 |
| Merged | 1 |

## 9. Reopened tickets: 6 (rate 54.5%)

Detected via JQL `status CHANGED FROM ("Closed", "Done", "Resolved", "Merged") DURING ("2026-07-01", "2026-08-01")`. Rate is `reopens / bugs closed in July`. Note that the underlying `Merged` status is Workiva's internal In-Progress state, so a `Merged → Closed` transition is caught by this JQL even though it isn't a functional "reopen". These would need manual triage to confirm; upper bound is 6 tickets.

| Key | Summary | Currently | Resolution date | Link |
|---|---|---|---|---|
| `INTRISK-101994` | Fix report column format error causing incorrect display | Closed | 2026-07-02 | [link](https://jira.atl.workiva.net/browse/INTRISK-101994) |
| `INTRISK-102954` | Risks and Controls Column Management unresponsive | Closed | 2026-07-02 | [link](https://jira.atl.workiva.net/browse/INTRISK-102954) |
| `INTRISK-100615` | Permission issues with Action Plans being linked to Issues | Closed | 2026-07-03 | [link](https://jira.atl.workiva.net/browse/INTRISK-100615) |
| `INTRISK-102747` | Show Create Chart from Table menu in non-English locales | Closed | 2026-07-08 | [link](https://jira.atl.workiva.net/browse/INTRISK-102747) |
| `INTRISK-103769` | fix: raise TDeserializer max message size for Thrift 0.24.0 | Closed | 2026-07-23 | [link](https://jira.atl.workiva.net/browse/INTRISK-103769) |
| `INTRISK-102119` | Fix Failing Signals Test: graph_app_sandbox_LinkCreation (Plan 4936476 | Closed | 2026-07-28 | [link](https://jira.atl.workiva.net/browse/INTRISK-102119) |

## 10. Tickets still open from July: 6

| Key | Summary | Module | Priority | Status | Age (d) | Assignee | Link |
|---|---|---|---|---|---:|---|---|
| `INTRISK-102967` | Risks and Controls-could not uncheck Show/hide checkbox  | `grc_universe_client` | Major | New | 31.9 | Manil Kumar | [link](https://jira.atl.workiva.net/browse/INTRISK-102967) |
| `INTRISK-103359` | Fix runtime type errors after sound null safety migration | `sa-tools-data-modeler` | Major | Merged | 25.7 | Manil Kumar | [link](https://jira.atl.workiva.net/browse/INTRISK-103359) |
| `INTRISK-103492` | [Data Modeler] NextGen Parsing (Beta): Clicking "+" to add a | `sa-tools-data-modeler` | Minor | New | 21.3 | — | [link](https://jira.atl.workiva.net/browse/INTRISK-103492) |
| `INTRISK-103882` | Disable/Remove New Relic Dependabot Alerts for Forms Reposit | `Unassigned` | Minor | In Progress | 11.2 | Manil Kumar | [link](https://jira.atl.workiva.net/browse/INTRISK-103882) |
| `INTRISK-103934` | Display proper error message when import fails due to lack o | `graph_app` | Minor | In Progress | 10.3 | Rannith Mohan | [link](https://jira.atl.workiva.net/browse/INTRISK-103934) |
| `INTRISK-104011` | There was an unexpected issue pasting markup" error toast ap | `Unassigned` | Blocker | New | 6.4 | Suresh Rajendran | [link](https://jira.atl.workiva.net/browse/INTRISK-104011) |

## 11. Aging of currently-open tickets (backlog = 128)

| Bucket (days) | Count | % |
|---|---:|---:|
| 0–7 | 1 | 0.8% |
| 8–14 | 2 | 1.6% |
| 15–30 | 2 | 1.6% |
| 31–60 | 14 | 10.9% |
| >60 | 109 | 85.2% |

See `open-ticket-aging-by-module.csv` for the same buckets split by module.

## 12. SLA %

No Jira SLA data found. Custom fields `customfield_21020 / _19321 / _35425 / _21925 / _10216 / _22826` were fetched but are all null on the sampled INTRISK tickets — this project is a software project, not a Service Desk project, so no SLA policy is applied. Metric 12 is reported as **N/A**.

## Modules with the highest ticket volume (created + closed in July)

| Rank | Module | Volume |
|---:|---|---:|
| 1 | `sa-tools-data-modeler` | 5 |
| 2 | `graph_app` | 4 |
| 3 | `sa-tools-changeset-service` | 4 |
| 4 | `Unassigned` | 4 |
| 5 | `grc_universe_client` | 3 |
| 6 | `form-service` | 2 |
| 7 | `audit` | 2 |
| 8 | `grc-services` | 1 |
| 9 | `2026-classic-grc` | 1 |

## Tickets with unusually long response or resolution times

*Outlier rule: `> Q3 + 1.5 × IQR` on the metric (Tukey fence); if the sample is too small, `> 1.5 × median`.*

### Long first-response
| Key | Summary | Module | Priority | Owner | Status | Age (h) | Link |
|---|---|---|---|---|---|---:|---|
| `INTRISK-103632` | Prod Support: Deleted attachment edges persist on CSA — fals | `form-service` | Major | Manil Kumar | Closed | 129.7 | [link](https://jira.atl.workiva.net/browse/INTRISK-103632) |

### Long resolution
| Key | Summary | Module | Priority | Owner | Status | Age (d) | Link |
|---|---|---|---|---|---|---:|---|
| `INTRISK-100615` | Permission issues with Action Plans being linked to Issues | `2026-classic-grc` | Major | Shivesh Pandey | Closed | 63.5 | [link](https://jira.atl.workiva.net/browse/INTRISK-100615) |
| `INTRISK-102119` | Fix Failing Signals Test: graph_app_sandbox_LinkCreation (Pl | `graph_app` | Major | Rannith Mohan | Closed | 41.5 | [link](https://jira.atl.workiva.net/browse/INTRISK-102119) |

## Recommended talking points for the Randi sync

1. **Reopen rate is 54.5% (6/11).** Even accepting the JQL false-positive risk on `Merged → Closed`, this needs a spot-check on how QA sign-off / verification is happening before we mark tickets as `Closed`.
2. **6 of the 15 bugs opened in July are still open** — the ones listed in section 10, especially any with age already past 14 days, should get an owner check-in this week.
3. **Module hotspot:** `sa-tools-data-modeler` — worth calling out to Randi with the specific ticket list so she can decide whether to raise it with the owning team for a mini-hardening cycle.
4. **Aging: 109 bugs >60 days old** (=85.2% of the backlog). Propose a monthly "stale-bug" review — either fix, downgrade, or explicitly park with an ADR-level note.
5. **SLA telemetry is missing.** If Randi wants a defensible response-time metric, we'd need to either enable Jira Service Management SLAs on the INTRISK-toolbox scope, or agree on a manual SLA proxy (e.g., "first non-bot comment within 24 h") and start tracking it. Section 5 shows how we compute the proxy today.
6. **Data-quality follow-up:** the `Merged` status is treated as In-Progress by Jira but visually looks "done" to the team. Recommend either renaming or adding a hover-help so triage doesn't count them as closed.

## JQL queries used

```jql
# Base team-scope filter:
issuetype = Bug AND labels = "toolbox" AND labels not in ("InfoSec:Semgrep", "InfoSec:Grype", "InfoSec:Dependabot", "InfoSec:Vulnerability")

# Created in July:
issuetype = Bug AND labels = "toolbox" AND labels not in ("InfoSec:Semgrep", "InfoSec:Grype", "InfoSec:Dependabot", "InfoSec:Vulnerability") AND created >= "2026-07-01" AND created < "2026-08-01" ORDER BY created ASC

# Closed in July (resolutiondate):
issuetype = Bug AND labels = "toolbox" AND labels not in ("InfoSec:Semgrep", "InfoSec:Grype", "InfoSec:Dependabot", "InfoSec:Vulnerability") AND resolutiondate >= "2026-07-01" AND resolutiondate < "2026-08-01" ORDER BY resolutiondate ASC

# Currently open:
issuetype = Bug AND labels = "toolbox" AND labels not in ("InfoSec:Semgrep", "InfoSec:Grype", "InfoSec:Dependabot", "InfoSec:Vulnerability") AND resolution is EMPTY ORDER BY created ASC

# Reopens during July:
issuetype = Bug AND labels = "toolbox" AND labels not in ("InfoSec:Semgrep", "InfoSec:Grype", "InfoSec:Dependabot", "InfoSec:Vulnerability") AND status CHANGED FROM ("Closed", "Done", "Resolved", "Merged") DURING ("2026-07-01", "2026-08-01")
```

## Assumptions and missing data

- The team-scope filter is **`labels = toolbox`** (Manil's confirmed team label). If a ticket is legitimately your team's product bug but the label wasn't applied, it is *not* in this report — no guess-work back-fill was done.
- The `InfoSec:*` labels are treated as security-scanner findings and excluded from the product-bug population. Toggle in `build_report.py` if you want them counted.
- **First-response time** is a proxy: SLA fields aren't set on INTRISK, so we use `created → first non-bot comment (from someone other than the reporter)`. Comments authored by btr-, jenkins, github-actions, rosie, blackduck, renovate, dependabot* are skipped.
- **Reopens** come from JQL `status CHANGED FROM (…)`. Since Workiva's `Merged` status is technically an In-Progress category state, `Merged → Closed` transitions are caught by this JQL. Actual "user-observable" reopens may be fewer than 6; the number in this report is an upper bound.
- **Changelog access:** the MCP `jira_get_comments` tool is broken on this Jira instance (always sends an unsupported `start` kwarg), so comments were fetched by including `comment` in `jira_batch_get_issues.fields`. That is why comment history is capped at the default 1000 per issue (never a real limit here).
- ✅ **Validation:** all summary totals reconcile with the underlying ticket lists — module × new = created, module × closed = closed, priority totals match, status totals match, aging buckets sum to open backlog.

## Files

| File | Purpose |
|---|---|
| `product-bugs-jul-2026.md` | this report |
| `product-bugs-jul-2026/created-vs-closed-by-month.csv` | Metric 1+2 |
| `product-bugs-jul-2026/net-backlog-change-by-month.csv` | Metric 3 |
| `product-bugs-jul-2026/tickets-by-module-and-month.csv` | Metric 4 |
| `product-bugs-jul-2026/response-time-by-month.csv` | Metric 5 |
| `product-bugs-jul-2026/resolution-time-by-month.csv` | Metric 6 |
| `product-bugs-jul-2026/tickets-by-priority.csv` | Metric 7 |
| `product-bugs-jul-2026/july-status-breakdown.csv` | Metric 8 |
| `product-bugs-jul-2026/open-ticket-aging.csv` | Metric 11 |
| `product-bugs-jul-2026/open-ticket-aging-by-module.csv` | Metric 11 split by module |
| `product-bugs-jul-2026/july-ticket-detail.csv` | one row per July ticket (created, closed, or excluded) |
