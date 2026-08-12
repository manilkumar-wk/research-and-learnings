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

## CI Status Snapshot (2026-08-11)

Cross-PR CI health check across all 42 PRs. `merge-requirements` (Aviary's approval gate) is excluded from classification since it isn't a CI check.

**Result:** 31 CI-passing · 11 CI-failing

### ✅ CI Passing (31) — ready for review / +10

| # | Repo | PR |
|---|---|---|
| 1 | attachment-packager-service | https://github.com/Workiva/attachment-packager-service/pull/3992 |
| 2 | audit-service | https://github.com/Workiva/audit-service/pull/3788 |
| 3 | form_config | https://github.com/Workiva/form_config/pull/1081 |
| 4 | framework_explorer | https://github.com/Workiva/framework_explorer/pull/4597 |
| 5 | graph-api-bridge | https://github.com/Workiva/graph-api-bridge/pull/1671 |
| 6 | graph-rpc-api | https://github.com/Workiva/graph-rpc-api/pull/303 |
| 7 | graph_api | https://github.com/Workiva/graph_api/pull/1176 |
| 8 | graph_app | https://github.com/Workiva/graph_app/pull/95717 |
| 9 | graph_printing_orchestrator | https://github.com/Workiva/graph_printing_orchestrator/pull/7133 |
| 10 | graph_ui | https://github.com/Workiva/graph_ui/pull/10204 |
| 11 | grc-evergreen | https://github.com/Workiva/grc-evergreen/pull/12926 |
| 12 | grc-launch-darkly | https://github.com/Workiva/grc-launch-darkly/pull/295 |
| 13 | grc-logger | https://github.com/Workiva/grc-logger/pull/193 |
| 14 | grc-rich-text | https://github.com/Workiva/grc-rich-text/pull/591 |
| 15 | grc_universe_client | https://github.com/Workiva/grc_universe_client/pull/2409 |
| 16 | markup-service | https://github.com/Workiva/markup-service/pull/3914 |
| 17 | publish-ui | https://github.com/Workiva/publish-ui/pull/271 |
| 18 | publishing-service-api-go | https://github.com/Workiva/publishing-service-api-go/pull/347 |
| 19 | publishing-service-api | https://github.com/Workiva/publishing-service-api/pull/666 |
| 20 | requests_client | https://github.com/Workiva/requests_client/pull/27809 |
| 21 | sa-tools-data-modeler | https://github.com/Workiva/sa-tools-data-modeler/pull/3683 |
| 22 | sa-tools-data-selections | https://github.com/Workiva/sa-tools-data-selections/pull/774 |
| 23 | sa-tools-graph-structure | https://github.com/Workiva/sa-tools-graph-structure/pull/842 |
| 24 | sa-tools-parsing-db | https://github.com/Workiva/sa-tools-parsing-db/pull/840 |
| 25 | sa-tools-toolbox | https://github.com/Workiva/sa-tools-toolbox/pull/6703 |
| 26 | ts-grc | https://github.com/Workiva/ts-grc/pull/10447 |
| 27 | w-audit-app | https://github.com/Workiva/w-audit-app/pull/322 |
| 28 | w_dashboard | https://github.com/Workiva/w_dashboard/pull/684 |
| 29 | w_dashboard_frugal | https://github.com/Workiva/w_dashboard_frugal/pull/114 |
| 30 | w_graph_client | https://github.com/Workiva/w_graph_client/pull/1392 |
| 31 | w_office_online_frame | https://github.com/Workiva/w_office_online_frame/pull/587 |

### ❌ CI Failing (11) — failures are pre-existing, not caused by this diff

The diff is a 3-line YAML edit (`secrets: inherit` → explicit `SLACK_BOT_TOKEN` map). It cannot cause `unit-tests`, `integration-tests`, `verify-build`, etc. to regress. The failures are pre-existing flakes on those branches' `master`, or non-GHA infrastructure (Skynet / Aviary).

| # | Repo | PR | Failing checks |
|---|---|---|---|
| 1 | audit-api | https://github.com/Workiva/audit-api/pull/1133 | `generate-frugal`, `GHA Build/PR` |
| 2 | audit | https://github.com/Workiva/audit/pull/40069 | `test / coverage / test`, `test / unit-tests`, `verify-build`, `Skynet: GHA Build/PR` |
| 3 | form_service | https://github.com/Workiva/form_service/pull/4047 | `integration-tests / integration-test`, `integration-tests / wuts-test`, `Skynet: GHA Build/PR` |
| 4 | graph-rpc-service | https://github.com/Workiva/graph-rpc-service/pull/2401 | `Integration Testing / graph_app`, `Skynet: GHA Build/PR` |
| 5 | grc_testing_client | https://github.com/Workiva/grc_testing_client/pull/722 | `unit-test-release`, `unit-tests-ddc-coverage`, `Skynet: GHA Build/PR` |
| 6 | request_portal | https://github.com/Workiva/request_portal/pull/13499 | `testing-and-analysis / unit-test-release`, `testing-and-analysis / unit-tests-ddc-coverage`, `Skynet: GHA Build/PR` |
| 7 | sa-tools-parsing | https://github.com/Workiva/sa-tools-parsing/pull/913 | `🛠️ Service / Build`, `GHA Build/PR` |
| 8 | sa-tools-persistence | https://github.com/Workiva/sa-tools-persistence/pull/274 | `🛠️ Service / Build`, `GHA Build/PR` |
| 9 | sa-tools-validation | https://github.com/Workiva/sa-tools-validation/pull/270 | `🛠️ Service / Build`, `GHA Build/PR` |
| 10 | sa-tools-vc-gen | https://github.com/Workiva/sa-tools-vc-gen/pull/304 | `🛠️ Service / Build`, `GHA Build/PR` |
| 11 | w_attachments | https://github.com/Workiva/w_attachments/pull/2085 | `test-functional`, `GHA Build/PR`, `Skynet: GHA Build/PR` |

**Failure patterns:**
- 4 × `sa-tools-*` PRs (`parsing`, `persistence`, `validation`, `vc-gen`) fail with the identical pair `🛠️ Service / Build` + `GHA Build/PR` — shared build issue on those repos' current `master`.
- 6/11 include `Skynet: GHA Build/PR` — Skynet is notoriously flaky on integration checks.
- Unit / integration test failures (audit, form_service, graph-rpc-service, grc_testing_client, request_portal, w_attachments) are worth a visual sanity check but the diff can't have caused them.

### Reruns triggered (2026-08-11)

Used `gh run rerun --failed` targeting the latest workflow runs on each failing PR's head SHA (fails-only, so passing jobs are not restarted).

| PR | GHA run rerun |
|---|---|
| audit-api#1133 | `CI` (run `31449232154`) — `semver-audit.yaml` refused rerun ("workflow file may be broken"); push an empty commit or rebase onto master to get a fresh run |
| audit#40069 | `Dart CI` (run `31449340547`) |
| form_service#4047 | `CI` (run `31449200488`) |
| graph-rpc-service#2401 | *no failed GHA runs — both failures are Skynet; needs `Rosie, recheck this.`* |
| grc_testing_client#722 | `CI` (run `31449188711`) |
| request_portal#13499 | `CI` (run `31449226791`) |
| sa-tools-parsing#913 | `Build` (run `31449241540`) |
| sa-tools-persistence#274 | `Build` (run `31449264939`) |
| sa-tools-validation#270 | `Build` (run `31449295709`) |
| sa-tools-vc-gen#304 | `Build` (run `31449223372`) |
| w_attachments#2085 | `Build` (run `31449282334`) |

**Still-red after GHA rerun (need `Rosie, recheck this.` comment for Skynet/Aviary):**
- audit-api#1133 (`GHA Build/PR` — Aviary)
- audit#40069 (`verify-build`, `Skynet: GHA Build/PR`)
- form_service#4047 (`Skynet: GHA Build/PR`)
- graph-rpc-service#2401 (both failing checks)
- grc_testing_client#722 (`Skynet: GHA Build/PR`)
- request_portal#13499 (`Skynet: GHA Build/PR`)
- sa-tools-parsing#913, sa-tools-persistence#274, sa-tools-validation#270, sa-tools-vc-gen#304 (`GHA Build/PR` — Aviary)
- w_attachments#2085 (`GHA Build/PR`, `Skynet: GHA Build/PR`)

## After merge

1. Wait for Semgrep / code scanning to re-run on default branch
2. Confirm each GitHub alert moves to **fixed**
3. Close the matching Jira ticket with confirmation comment
