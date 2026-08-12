# Semgrep secrets-inherit PRs (Round 2)

**Date:** 2026-08-12  
**Scope:** Open InfoSec Semgrep tickets with `secrets: inherit` across various repos.  
**Fix pattern:** Applies the same shape as [ts-grc#10517](https://github.com/Workiva/ts-grc/pull/10517) to every PR — declare the child workflow's secrets where possible + pass an explicit least-privilege secrets map from the caller. Where the child references zero secrets, the "explicit map" is empty (i.e. no `secrets:` block) — same effect as removal.  
**Branch pattern:** `{JIRA}` (bare Jira key)

## Ticket triage

- **Total tickets in batch:** 32
- **PRs created:** 31
- **Skipped:** 1 — [GRC-8594](https://jira.atl.workiva.net/browse/GRC-8594) (`react-dangerouslysetinnerhtml`) — user handling manually
- **Team-scope duplicates found (INTRISK/GRC/SAT/A9S/Forms Platform/Services Tools):** 0
- **PRs superseded/closed:** 1 — [ts-grc#10518](https://github.com/Workiva/ts-grc/pull/10518) folded into #10517

## Fix-strategy classification (per reusable-workflow analysis)

| Bucket | Count | Child reference | Fix applied |
|---|---|---|---|
| **A. Child uses zero secrets** — plain removal is correct | 19 | Local reusable workflow with no `${{ secrets.* }}` refs (or external `gha-semver-audit` which uses none) | Remove `secrets: inherit` only |
| **B. External `gha-automated-pr-check`** | 4 | Child uses `SLACK_BOT_TOKEN`; older tags (v0.1.2–v0.1.7) don't declare it in `on.workflow_call.secrets:` | Bump to `v0.1.8` + explicit `SLACK_BOT_TOKEN` map |
| **C. External `gha-ai-code-review`** | 6 | Child uses 8 declared-optional secrets (Bedrock/Slack/Jira integrations) | Explicit map of all 8 |
| **D. Local `integration-workiva-shell.yaml`** | 2 (→ 1 after supersede) | Child uses `JOBRUNR_PRO_LICENSE` + `SLACK_BOT_TOKEN`; didn't declare them | Declare in child + explicit map for **all 21** `secrets: inherit` in file |

## PRs (final fix pattern applied)

| Jira | Repo | File / Alert | PR | Bucket | Fix content |
|---|---|---|---|---|---|
| [INTRISK-103228](https://jira.atl.workiva.net/browse/INTRISK-103228) | audit-service | `ci.yaml` (test) | [#3790](https://github.com/Workiva/audit-service/pull/3790) | A | remove |
| [INTRISK-103227](https://jira.atl.workiva.net/browse/INTRISK-103227) | audit-service | `ai-review-scheduled.yml` | [#3791](https://github.com/Workiva/audit-service/pull/3791) | C | explicit 8-secret map |
| [INTRISK-103224](https://jira.atl.workiva.net/browse/INTRISK-103224) | audit-service | `ai-review.yml` | [#3792](https://github.com/Workiva/audit-service/pull/3792) | C | explicit 8-secret map |
| [INTRISK-103223](https://jira.atl.workiva.net/browse/INTRISK-103223) | audit-service | `ci.yaml` (signals-image) | [#3793](https://github.com/Workiva/audit-service/pull/3793) | A | remove |
| [INTRISK-103225](https://jira.atl.workiva.net/browse/INTRISK-103225) | audit-service | `test.yaml` (form-service) | [#3794](https://github.com/Workiva/audit-service/pull/3794) | A | remove |
| [INTRISK-103233](https://jira.atl.workiva.net/browse/INTRISK-103233) | assessments_client | `ci.yaml` | [#2432](https://github.com/Workiva/assessments_client/pull/2432) | A | remove |
| [INTRISK-103307](https://jira.atl.workiva.net/browse/INTRISK-103307) | attachment_packager_frugal | `check-prs.yaml` | [#374](https://github.com/Workiva/attachment_packager_frugal/pull/374) | B | bump v0.1.5 → v0.1.8 + `SLACK_BOT_TOKEN` |
| [INTRISK-103222](https://jira.atl.workiva.net/browse/INTRISK-103222) | audit | `ai-review-scheduled.yml` | [#40083](https://github.com/Workiva/audit/pull/40083) | C | explicit 8-secret map |
| [INTRISK-103208](https://jira.atl.workiva.net/browse/INTRISK-103208) | audit | `ai-review.yml` | [#40084](https://github.com/Workiva/audit/pull/40084) | C | explicit 8-secret map |
| [INTRISK-103201](https://jira.atl.workiva.net/browse/INTRISK-103201) | audit | `dart_ci.yml` | [#40085](https://github.com/Workiva/audit/pull/40085) | A | remove |
| [INTRISK-103221](https://jira.atl.workiva.net/browse/INTRISK-103221) | audit | `test_and_analysis.yml` | [#40086](https://github.com/Workiva/audit/pull/40086) | A | remove |
| [INTRISK-103202](https://jira.atl.workiva.net/browse/INTRISK-103202) | graph-app-structure | `check-prs.yaml` | [#682](https://github.com/Workiva/graph-app-structure/pull/682) | B | bump v0.1.7 → v0.1.8 + `SLACK_BOT_TOKEN` |
| [INTRISK-103305](https://jira.atl.workiva.net/browse/INTRISK-103305) | graph-template-creation-service | `.github/check-prs.yaml` | [#1056](https://github.com/Workiva/graph-template-creation-service/pull/1056) | B | bump v0.1.2 → v0.1.8 + `SLACK_BOT_TOKEN` |
| [INTRISK-103303](https://jira.atl.workiva.net/browse/INTRISK-103303) | graph-template-creation-service | `build.yaml` | [#1057](https://github.com/Workiva/graph-template-creation-service/pull/1057) | A | remove |
| [INTRISK-103218](https://jira.atl.workiva.net/browse/INTRISK-103218) | graph_app_js | `check-prs.yaml` | [#68](https://github.com/Workiva/graph_app_js/pull/68) | B | bump v0.1.3 → v0.1.8 + `SLACK_BOT_TOKEN` |
| [INTRISK-103297](https://jira.atl.workiva.net/browse/INTRISK-103297) | graph_ui | `ai-review-scheduled.yml` | [#10205](https://github.com/Workiva/graph_ui/pull/10205) | C | explicit 8-secret map |
| [INTRISK-103299](https://jira.atl.workiva.net/browse/INTRISK-103299) | graph_ui | `ai-review.yml` | [#10206](https://github.com/Workiva/graph_ui/pull/10206) | C | explicit 8-secret map |
| [GRC-9849](https://jira.atl.workiva.net/browse/GRC-9849) | grc-evergreen | `build.yml` (assurance-catalog-db-check) | [#13077](https://github.com/Workiva/grc-evergreen/pull/13077) | A | remove |
| [INTRISK-103294](https://jira.atl.workiva.net/browse/INTRISK-103294) | grc-services | `ci.yaml` | [#5444](https://github.com/Workiva/grc-services/pull/5444) | A | remove |
| [INTRISK-103293](https://jira.atl.workiva.net/browse/INTRISK-103293) | grc-services | `ci.yaml` | [#5445](https://github.com/Workiva/grc-services/pull/5445) | A | remove |
| [INTRISK-103214](https://jira.atl.workiva.net/browse/INTRISK-103214) | markup_client | `dart_ci.yml` | [#5703](https://github.com/Workiva/markup_client/pull/5703) | A | remove |
| [INTRISK-103216](https://jira.atl.workiva.net/browse/INTRISK-103216) | markup_client | `dart_ci.yml` | [#5704](https://github.com/Workiva/markup_client/pull/5704) | A | remove |
| [INTRISK-103206](https://jira.atl.workiva.net/browse/INTRISK-103206) | markup_client | `dart_ci.yml` | [#5705](https://github.com/Workiva/markup_client/pull/5705) | A | remove |
| [INTRISK-103215](https://jira.atl.workiva.net/browse/INTRISK-103215) | markup_client | `dart_ci.yml` | [#5706](https://github.com/Workiva/markup_client/pull/5706) | A | remove |
| [INTRISK-103290](https://jira.atl.workiva.net/browse/INTRISK-103290) | request_portal | `ci.yaml` | [#13502](https://github.com/Workiva/request_portal/pull/13502) | A | remove |
| [INTRISK-103289](https://jira.atl.workiva.net/browse/INTRISK-103289) | request_portal | `ci.yaml` | [#13503](https://github.com/Workiva/request_portal/pull/13503) | A | remove |
| [INTRISK-103210](https://jira.atl.workiva.net/browse/INTRISK-103210) | requests_client | `ci.yaml` | [#27818](https://github.com/Workiva/requests_client/pull/27818) | A | remove |
| [INTRISK-103209](https://jira.atl.workiva.net/browse/INTRISK-103209) | requests_client | `ci.yaml` | [#27819](https://github.com/Workiva/requests_client/pull/27819) | A | remove |
| [INTRISK-103285](https://jira.atl.workiva.net/browse/INTRISK-103285) | sa-tools-vc-gen | `main.yml` | [#306](https://github.com/Workiva/sa-tools-vc-gen/pull/306) | A | remove |
| [GRC-10060](https://jira.atl.workiva.net/browse/GRC-10060) | ts-grc | `run-functional-tests.yaml` (all 21 jobs) | [#10517](https://github.com/Workiva/ts-grc/pull/10517) | D | declare secrets in child + explicit map on all 21 callers |
| [GRC-10062](https://jira.atl.workiva.net/browse/GRC-10062) | ts-grc | `run-functional-tests.yaml` (line 163) | ~[#10518](https://github.com/Workiva/ts-grc/pull/10518)~ **superseded by #10517** | D | (see #10517) |

## Rationale

Following the review on [ts-grc#10517](https://github.com/Workiva/ts-grc/pull/10517), simply removing `secrets: inherit` was found to break functionality when the reusable workflow references secrets. The corrected pattern per bucket:

- **Bucket A** — Child references zero secrets. Removal is semantically equivalent to `secrets: {}` and remains the fix.
- **Bucket B** — Older `gha-automated-pr-check` tags don't declare `secrets:` in their `workflow_call:`. Explicit passing fails schema validation on those tags. Bumping to `v0.1.8` (the first tag that declares `SLACK_BOT_TOKEN` as optional) enables the least-privilege explicit-map pattern without any behavioral drift for callers.
- **Bucket C** — `gha-ai-code-review` declares 8 optional secrets. Passing them explicitly preserves AI-review Bedrock/Slack/Jira integrations that would silently degrade if we just removed the line.
- **Bucket D** — Local child needed to declare its 2 secrets (`JOBRUNR_PRO_LICENSE`, `SLACK_BOT_TOKEN`) so callers can pass them explicitly. Because all 21 caller jobs in `run-functional-tests.yaml` share the same shape, the fix was expanded to cover all of them in one PR.

## Next actions

- Wait for CI on all 31 PRs (some will retrigger from the amendments).
- Once each PR is green, comment `@Workiva/release-management-p` to trigger Rosie automerge (Round 1 pattern).
- [GRC-8594](https://jira.atl.workiva.net/browse/GRC-8594) (XSS) is handled separately by the user.
