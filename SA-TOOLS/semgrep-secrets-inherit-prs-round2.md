# Semgrep secrets-inherit PRs (Round 2)

**Date:** 2026-08-12  
**Scope:** Open InfoSec Semgrep tickets with `secrets: inherit` across various repos (superset of just `check-prs.yaml` — includes `ci.yaml`, `ai-review*.yml`, `dart_ci.yml`, `build.yml`, `test.yaml`, `main.yml`, `run-functional-tests.yaml`, etc.)  
**Fix pattern:** Remove `secrets: inherit` line. If any reusable workflow actually needs secrets, CI will flag it and reviewer can add an explicit `secrets:` map with only the required entries.  
**Branch pattern:** `{JIRA}` (bare Jira key)

## Ticket triage

- **Total tickets in batch:** 32
- **PRs created:** 31
- **Skipped:** 1 — [GRC-8594](https://jira.atl.workiva.net/browse/GRC-8594) (`react-dangerouslysetinnerhtml` in `packages/ts-grc-frameworks/src/components/format-markdown.tsx`) — user handling manually
- **Team-scope duplicates found (INTRISK/GRC/SAT/A9S/Forms Platform/Services Tools):** 0

## Duplicate policy applied

Per user direction, only INTRISK/GRC duplicates within team-managed projects are subject to closure. SEC-project tickets are the master alerts and were intentionally not touched. No team-scope duplicates were found in this batch.

## PRs

| Jira | Repo | File / Alert | Branch | PR | CI snapshot (2026-08-12) |
|---|---|---|---|---|---|
| [INTRISK-103228](https://jira.atl.workiva.net/browse/INTRISK-103228) | audit-service | `ci.yaml` (test) | `INTRISK-103228` | [#3790](https://github.com/Workiva/audit-service/pull/3790) | 1 fail (Skynet flake / integration) |
| [INTRISK-103227](https://jira.atl.workiva.net/browse/INTRISK-103227) | audit-service | `ai-review-scheduled.yml` | `INTRISK-103227` | [#3791](https://github.com/Workiva/audit-service/pull/3791) | ✅ green |
| [INTRISK-103224](https://jira.atl.workiva.net/browse/INTRISK-103224) | audit-service | `ai-review.yml` | `INTRISK-103224` | [#3792](https://github.com/Workiva/audit-service/pull/3792) | 2 fail |
| [INTRISK-103223](https://jira.atl.workiva.net/browse/INTRISK-103223) | audit-service | `ci.yaml` (signals-image) | `INTRISK-103223` | [#3793](https://github.com/Workiva/audit-service/pull/3793) | 1 fail |
| [INTRISK-103225](https://jira.atl.workiva.net/browse/INTRISK-103225) | audit-service | `test.yaml` (form-service) | `INTRISK-103225` | [#3794](https://github.com/Workiva/audit-service/pull/3794) | ✅ green |
| [INTRISK-103233](https://jira.atl.workiva.net/browse/INTRISK-103233) | assessments_client | `ci.yaml` | `INTRISK-103233` | [#2432](https://github.com/Workiva/assessments_client/pull/2432) | ✅ green |
| [INTRISK-103307](https://jira.atl.workiva.net/browse/INTRISK-103307) | attachment_packager_frugal | `check-prs.yaml` | `INTRISK-103307` | [#374](https://github.com/Workiva/attachment_packager_frugal/pull/374) | 2 fail (`frugal-audit`, etc.) |
| [INTRISK-103222](https://jira.atl.workiva.net/browse/INTRISK-103222) | audit | `ai-review-scheduled.yml` | `INTRISK-103222` | [#40083](https://github.com/Workiva/audit/pull/40083) | 2 fail (`verify-build`, integration) |
| [INTRISK-103208](https://jira.atl.workiva.net/browse/INTRISK-103208) | audit | `ai-review.yml` | `INTRISK-103208` | [#40084](https://github.com/Workiva/audit/pull/40084) | 2 fail |
| [INTRISK-103201](https://jira.atl.workiva.net/browse/INTRISK-103201) | audit | `dart_ci.yml` | `INTRISK-103201` | [#40085](https://github.com/Workiva/audit/pull/40085) | 1 fail |
| [INTRISK-103221](https://jira.atl.workiva.net/browse/INTRISK-103221) | audit | `test_and_analysis.yml` | `INTRISK-103221` | [#40086](https://github.com/Workiva/audit/pull/40086) | 2 fail |
| [INTRISK-103202](https://jira.atl.workiva.net/browse/INTRISK-103202) | graph-app-structure | `check-prs.yaml` | `INTRISK-103202` | [#682](https://github.com/Workiva/graph-app-structure/pull/682) | ✅ green |
| [INTRISK-103305](https://jira.atl.workiva.net/browse/INTRISK-103305) | graph-template-creation-service | `.github/check-prs.yaml` | `INTRISK-103305` | [#1056](https://github.com/Workiva/graph-template-creation-service/pull/1056) | 1 fail |
| [INTRISK-103303](https://jira.atl.workiva.net/browse/INTRISK-103303) | graph-template-creation-service | `build.yaml` | `INTRISK-103303` | [#1057](https://github.com/Workiva/graph-template-creation-service/pull/1057) | ✅ green |
| [INTRISK-103218](https://jira.atl.workiva.net/browse/INTRISK-103218) | graph_app_js | `check-prs.yaml` | `INTRISK-103218` | [#68](https://github.com/Workiva/graph_app_js/pull/68) | ✅ green |
| [INTRISK-103297](https://jira.atl.workiva.net/browse/INTRISK-103297) | graph_ui | `ai-review-scheduled.yml` | `INTRISK-103297` | [#10205](https://github.com/Workiva/graph_ui/pull/10205) | ✅ green |
| [INTRISK-103299](https://jira.atl.workiva.net/browse/INTRISK-103299) | graph_ui | `ai-review.yml` | `INTRISK-103299` | [#10206](https://github.com/Workiva/graph_ui/pull/10206) | 1 fail |
| [GRC-9849](https://jira.atl.workiva.net/browse/GRC-9849) | grc-evergreen | `build.yml` (assurance-catalog-db-check) | `GRC-9849` | [#13077](https://github.com/Workiva/grc-evergreen/pull/13077) | 1 fail (`codecov/project`) |
| [INTRISK-103294](https://jira.atl.workiva.net/browse/INTRISK-103294) | grc-services | `ci.yaml` | `INTRISK-103294` | [#5444](https://github.com/Workiva/grc-services/pull/5444) | ✅ green |
| [INTRISK-103293](https://jira.atl.workiva.net/browse/INTRISK-103293) | grc-services | `ci.yaml` | `INTRISK-103293` | [#5445](https://github.com/Workiva/grc-services/pull/5445) | ✅ green |
| [INTRISK-103214](https://jira.atl.workiva.net/browse/INTRISK-103214) | markup_client | `dart_ci.yml` | `INTRISK-103214` | [#5703](https://github.com/Workiva/markup_client/pull/5703) | 1 fail |
| [INTRISK-103216](https://jira.atl.workiva.net/browse/INTRISK-103216) | markup_client | `dart_ci.yml` | `INTRISK-103216` | [#5704](https://github.com/Workiva/markup_client/pull/5704) | ✅ green |
| [INTRISK-103206](https://jira.atl.workiva.net/browse/INTRISK-103206) | markup_client | `dart_ci.yml` | `INTRISK-103206` | [#5705](https://github.com/Workiva/markup_client/pull/5705) | ✅ green |
| [INTRISK-103215](https://jira.atl.workiva.net/browse/INTRISK-103215) | markup_client | `dart_ci.yml` | `INTRISK-103215` | [#5706](https://github.com/Workiva/markup_client/pull/5706) | ✅ green |
| [INTRISK-103290](https://jira.atl.workiva.net/browse/INTRISK-103290) | request_portal | `ci.yaml` | `INTRISK-103290` | [#13502](https://github.com/Workiva/request_portal/pull/13502) | ✅ green |
| [INTRISK-103289](https://jira.atl.workiva.net/browse/INTRISK-103289) | request_portal | `ci.yaml` | `INTRISK-103289` | [#13503](https://github.com/Workiva/request_portal/pull/13503) | 1 fail |
| [INTRISK-103210](https://jira.atl.workiva.net/browse/INTRISK-103210) | requests_client | `ci.yaml` | `INTRISK-103210` | [#27818](https://github.com/Workiva/requests_client/pull/27818) | ✅ green |
| [INTRISK-103209](https://jira.atl.workiva.net/browse/INTRISK-103209) | requests_client | `ci.yaml` | `INTRISK-103209` | [#27819](https://github.com/Workiva/requests_client/pull/27819) | ✅ green |
| [INTRISK-103285](https://jira.atl.workiva.net/browse/INTRISK-103285) | sa-tools-vc-gen | `main.yml` | `INTRISK-103285` | [#306](https://github.com/Workiva/sa-tools-vc-gen/pull/306) | ✅ green |
| [GRC-10060](https://jira.atl.workiva.net/browse/GRC-10060) | ts-grc | `run-functional-tests.yaml` | `GRC-10060` | [#10517](https://github.com/Workiva/ts-grc/pull/10517) | ✅ green |
| [GRC-10062](https://jira.atl.workiva.net/browse/GRC-10062) | ts-grc | `run-functional-tests.yaml` | `GRC-10062` | [#10518](https://github.com/Workiva/ts-grc/pull/10518) | 1 cancelled |

## Result

- **Green (no failing checks):** 17
- **Failing checks (needs review):** 13 — a mix of `verify-build`, `codecov/project`, `frugal-audit`, `Integration Testing`, `Skynet` results. Some may be flakes/repo-wide gates unrelated to the tiny YAML change; others may indicate the removed `secrets: inherit` was actually needed and require adding back an explicit `secrets:` map.
- **Cancelled:** 1 (GRC-10062, may need re-run)

## Next actions

1. For each failing PR, inspect the failing job. If the failure is caused by a workflow no longer receiving a required secret, restore that specific secret with an explicit map (e.g. `secrets: { SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }} }`) instead of `secrets: inherit`.
2. Once green, comment `@Workiva/release-management-p` on each PR to trigger Rosie automerge, following the same pattern used in Round 1.
3. GRC-8594 (XSS) handled separately by user.
