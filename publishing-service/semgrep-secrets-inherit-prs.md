# Semgrep secrets-inherit PRs (check-prs.yaml)

**Date:** 2026-08-10  
**Scope:** Open InfoSec Semgrep tickets with `secrets: inherit` in `.github/workflows/check-prs.yaml`  
**Fix:** Replace `secrets: inherit` with explicit `SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}` (least privilege for `Workiva/gha-automated-pr-check`)  
**Branch pattern:** `{JIRA}-semgrep-issue-fix`

**Result:** 42 / 42 PRs created

## PRs

| Jira | Repo | Branch | PR |
|---|---|---|---|
| GRC-10061 | ts-grc | `GRC-10061-semgrep-issue-fix` | https://github.com/Workiva/ts-grc/pull/10447 |
| INTRISK-103681 | markup-service | `INTRISK-103681-semgrep-issue-fix` | https://github.com/Workiva/markup-service/pull/3914 |
| GRC-10106 | grc-rich-text | `GRC-10106-semgrep-issue-fix` | https://github.com/Workiva/grc-rich-text/pull/591 |
| INTRISK-103690 | grc_testing_client | `INTRISK-103690-semgrep-issue-fix` | https://github.com/Workiva/grc_testing_client/pull/722 |
| INTRISK-103759 | requests_client | `INTRISK-103759-semgrep-issue-fix` | https://github.com/Workiva/requests_client/pull/27809 |
| INTRISK-103758 | audit-service | `INTRISK-103758-semgrep-issue-fix` | https://github.com/Workiva/audit-service/pull/3788 |
| INTRISK-103834 | form_service | `INTRISK-103834-semgrep-issue-fix` | https://github.com/Workiva/form_service/pull/4047 |
| INTRISK-103833 | graph-rpc-api | `INTRISK-103833-semgrep-issue-fix` | https://github.com/Workiva/graph-rpc-api/pull/303 |
| INTRISK-103831 | graph_app | `INTRISK-103831-semgrep-issue-fix` | https://github.com/Workiva/graph_app/pull/95717 |
| INTRISK-103830 | graph_ui | `INTRISK-103830-semgrep-issue-fix` | https://github.com/Workiva/graph_ui/pull/10204 |
| INTRISK-103829 | w_office_online_frame | `INTRISK-103829-semgrep-issue-fix` | https://github.com/Workiva/w_office_online_frame/pull/587 |
| INTRISK-103828 | publish-ui | `INTRISK-103828-semgrep-issue-fix` | https://github.com/Workiva/publish-ui/pull/271 |
| INTRISK-103827 | sa-tools-vc-gen | `INTRISK-103827-semgrep-issue-fix` | https://github.com/Workiva/sa-tools-vc-gen/pull/304 |
| INTRISK-103826 | request_portal | `INTRISK-103826-semgrep-issue-fix` | https://github.com/Workiva/request_portal/pull/13499 |
| INTRISK-103868 | audit-api | `INTRISK-103868-semgrep-issue-fix` | https://github.com/Workiva/audit-api/pull/1133 |
| INTRISK-103867 | graph-rpc-service | `INTRISK-103867-semgrep-issue-fix` | https://github.com/Workiva/graph-rpc-service/pull/2401 |
| INTRISK-103866 | sa-tools-parsing | `INTRISK-103866-semgrep-issue-fix` | https://github.com/Workiva/sa-tools-parsing/pull/913 |
| INTRISK-103865 | sa-tools-parsing-db | `INTRISK-103865-semgrep-issue-fix` | https://github.com/Workiva/sa-tools-parsing-db/pull/840 |
| INTRISK-103864 | w-audit-app | `INTRISK-103864-semgrep-issue-fix` | https://github.com/Workiva/w-audit-app/pull/322 |
| GRC-10213 | grc-launch-darkly | `GRC-10213-semgrep-issue-fix` | https://github.com/Workiva/grc-launch-darkly/pull/295 |
| GRC-10212 | grc-logger | `GRC-10212-semgrep-issue-fix` | https://github.com/Workiva/grc-logger/pull/193 |
| INTRISK-103922 | grc_universe_client | `INTRISK-103922-semgrep-issue-fix` | https://github.com/Workiva/grc_universe_client/pull/2409 |
| INTRISK-103921 | sa-tools-persistence | `INTRISK-103921-semgrep-issue-fix` | https://github.com/Workiva/sa-tools-persistence/pull/274 |
| INTRISK-103920 | sa-tools-data-selections | `INTRISK-103920-semgrep-issue-fix` | https://github.com/Workiva/sa-tools-data-selections/pull/774 |
| INTRISK-103919 | sa-tools-toolbox | `INTRISK-103919-semgrep-issue-fix` | https://github.com/Workiva/sa-tools-toolbox/pull/6703 |
| INTRISK-103918 | publishing-service-api | `INTRISK-103918-semgrep-issue-fix` | https://github.com/Workiva/publishing-service-api/pull/666 |
| INTRISK-103917 | w_attachments | `INTRISK-103917-semgrep-issue-fix` | https://github.com/Workiva/w_attachments/pull/2085 |
| INTRISK-103948 | attachment-packager-service | `INTRISK-103948-semgrep-issue-fix` | https://github.com/Workiva/attachment-packager-service/pull/3992 |
| GRC-10282 | grc-evergreen | `GRC-10282-semgrep-issue-fix` | https://github.com/Workiva/grc-evergreen/pull/12926 |
| FPLAT-3050 | w_graph_client | `FPLAT-3050-semgrep-issue-fix` | https://github.com/Workiva/w_graph_client/pull/1392 |
| INTRISK-103950 | sa-tools-validation | `INTRISK-103950-semgrep-issue-fix` | https://github.com/Workiva/sa-tools-validation/pull/270 |
| INTRISK-103949 | publishing-service-api-go | `INTRISK-103949-semgrep-issue-fix` | https://github.com/Workiva/publishing-service-api-go/pull/347 |
| INTRISK-103955 | form_config | `INTRISK-103955-semgrep-issue-fix` | https://github.com/Workiva/form_config/pull/1081 |
| INTRISK-103954 | sa-tools-graph-structure | `INTRISK-103954-semgrep-issue-fix` | https://github.com/Workiva/sa-tools-graph-structure/pull/842 |
| INTRISK-103953 | w_dashboard_frugal | `INTRISK-103953-semgrep-issue-fix` | https://github.com/Workiva/w_dashboard_frugal/pull/114 |
| INTRISK-103952 | w_dashboard | `INTRISK-103952-semgrep-issue-fix` | https://github.com/Workiva/w_dashboard/pull/684 |
| FPLAT-3051 | framework_explorer | `FPLAT-3051-semgrep-issue-fix` | https://github.com/Workiva/framework_explorer/pull/4597 |
| INTRISK-104154 | graph_printing_orchestrator | `INTRISK-104154-semgrep-issue-fix` | https://github.com/Workiva/graph_printing_orchestrator/pull/7133 |
| INTRISK-104153 | sa-tools-data-modeler | `INTRISK-104153-semgrep-issue-fix` | https://github.com/Workiva/sa-tools-data-modeler/pull/3683 |
| INTRISK-104339 | graph_api | `INTRISK-104339-semgrep-issue-fix` | https://github.com/Workiva/graph_api/pull/1176 |
| INTRISK-104394 | graph-api-bridge | `INTRISK-104394-semgrep-issue-fix` | https://github.com/Workiva/graph-api-bridge/pull/1671 |
| INTRISK-104465 | audit | `INTRISK-104465-semgrep-issue-fix` | https://github.com/Workiva/audit/pull/40069 |

## Not included (different workflow files)

- 6 × `syncdeps.yaml` / `syncdeps.yml` `secrets-inherit`
- 3 × `ts-grc` `run-functional-tests.yaml` `secrets-inherit`

## After merge

1. Wait for Semgrep / code scanning to re-run on default branch
2. Confirm each GitHub alert moves to **fixed**
3. Close the matching Jira ticket with confirmation comment
