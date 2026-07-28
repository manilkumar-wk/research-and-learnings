# graph_app MFE Migration: Jira Tickets & Task Breakdown

> **Estimation scale:** 1 story point = 1 day (8 hours)

## Epic Structure

```
EPIC-1: graph_app Independent Release Pipeline
  |
  +-- EPIC-1.1: Phase 0 - Preparation & Infrastructure
  +-- EPIC-1.2: Phase 1 - graph_app MFE Manifest Migration
  +-- EPIC-1.3: Phase 2 - Dual-Path Validation
  +-- EPIC-1.4: Phase 3 - Release Pipeline Setup
  +-- EPIC-1.5: Phase 4 - wdesk Decoupling
  +-- EPIC-1.6: Phase 5 - Production Cutover & Monitoring
  +-- EPIC-1.7: Phase 6 - Cleanup
```

---

## EPIC-1.1: Phase 0 -- Preparation & Infrastructure

> **Goal:** Set up all infrastructure and validate assumptions before
> writing any migration code. Zero code changes in this phase.

---

### TICKET: GRAPH-001
**Title:** Create `#alert-graph-app` Slack channel for pipeline notifications
**Type:** Task
**Priority:** Medium
**Component:** graph_app
**Labels:** `mfe-migration`, `infrastructure`
**Story Points:** 0.5 (half day)

**Description:**
Create a dedicated Slack channel `#alert-graph-app` for pipeline deployment
notifications and failure alerts. This channel will be referenced in the
`pipeline_template.yaml` for all deployment stage `on_fail` handlers.

**Acceptance Criteria:**
- [ ] Channel `#alert-graph-app` created in Slack
- [ ] Graph app team members added
- [ ] GRC flowmasters team (`@grc-flowmasters-team`) invited (needed for
      prod/APAC/EU failure escalation)
- [ ] Channel purpose documented: "Automated pipeline alerts for graph_app
      independent releases"

---

### TICKET: GRAPH-002
**Title:** Create Signals test plans for graph_app per-environment verification
**Type:** Story
**Priority:** High
**Component:** graph_app
**Labels:** `mfe-migration`, `testing`
**Story Points:** 5 (1 week -- 7 plans, ~0.5-1 day each)

**Description:**
Create Signals (Skynet) test plans for each deployment environment. These
plans will be used as promotion gates in the `pipeline_template.yaml`.
Base the test scenarios on graph_app's existing functional test suite
(`test/ci/` directory).

**Sub-tasks:**

| Sub-task | Description |
|---|---|
| GRAPH-002a | Create Signals plan for **staging** environment |
| GRAPH-002b | Create Signals plan for **pentest** environment |
| GRAPH-002c | Create Signals plan for **sandbox** environment |
| GRAPH-002d | Create Signals plan for **demo** environment |
| GRAPH-002e | Create Signals plan for **APAC (prod-apac)** environment |
| GRAPH-002f | Create Signals plan for **EU (prod-eu)** environment |
| GRAPH-002g | Create Signals plan for **prod** environment |

**Acceptance Criteria:**
- [ ] Each plan covers critical graph_app smoke tests (report load,
      dashboard render, data form navigation, evidence testing)
- [ ] Plans validated by running manually against staging
- [ ] Plan IDs documented for use in pipeline_template.yaml

**Dependencies:** None

---

### TICKET: GRAPH-003
**Title:** Validate FEWS manifest support for all graph_app contribution types
**Type:** Spike
**Priority:** Critical
**Component:** graph_app
**Labels:** `mfe-migration`, `spike`, `blocking`
**Story Points:** 3 (3 days -- POC + validation + documentation)

**Description:**
graph_app requires the following MFE contribution types in its manifest.
Before writing any migration code, we must confirm FEWS supports all of
them.

**Contribution types to validate:**
1. `core.drawer_experiences` -- Confirmed (form_config uses this)
2. `core.rich_experiences` -- Confirmed (form_config uses this)
3. `core.routing` -- Confirmed (form_config uses this)
4. `core.navigation_sidebar` -- Confirmed (form_config uses this)
5. `core.embeddable_experiences` -- **UNCONFIRMED** -- graph_app needs
   `createSampleSelectionExperienceConfig()` and
   `createTestFormSpreadsheetExperienceConfig()`
6. `core.landing_page_widgets` -- **UNCONFIRMED** -- graph_app has 10
   landing page widgets with database-persisted keys
7. Sidebar header contributions -- **UNCONFIRMED** -- graph_app uses
   `graph_archive_header` and `draft_edits_header`

**Acceptance Criteria:**
- [ ] Each contribution type tested with a minimal POC manifest
- [ ] Unsupported types documented with workaround proposals
- [ ] Decision recorded: which types need FEWS platform work vs
      workarounds in graph_app
- [ ] Findings shared with wdesk_sdk / FEWS platform team

**Dependencies:** None
**Blocks:** GRAPH-010, GRAPH-011, GRAPH-012

---

### TICKET: GRAPH-004
**Title:** Resolve GraphRoutePaths constants to literal strings for manifest
**Type:** Task
**Priority:** High
**Component:** graph_app
**Labels:** `mfe-migration`, `documentation`
**Story Points:** 0.5 (half day -- Claude greps constants and builds the table, human verifies)

**Description:**
graph_app's experience configs reference route segments via constants in
`graph_ui/lib/src/utils/graph_route_paths.dart` (e.g.,
`GraphRoutePaths.reports`, `GraphRoutePaths.dashboards`). The MFE
manifest requires literal strings. Produce a complete mapping table.

**Acceptance Criteria:**
- [ ] Every `GraphRoutePaths.*` and `SoxRoutePaths.*` constant resolved
      to its literal string value
- [ ] Mapping verified against the experience config source files
- [ ] Document available for Phase 1 manifest authoring
- [ ] Mapping table:

| Constant | Literal Value |
|---|---|
| `GraphRoutePaths.reports` | `reports` |
| `GraphRoutePaths.dashboards` | `dashboards` |
| `GraphRoutePaths.focus` | ? |
| `GraphRoutePaths.focusNew` | ? |
| `GraphRoutePaths.focusList` | ? |
| `GraphRoutePaths.dataModel` | ? |
| `GraphRoutePaths.support` | ? |
| `GraphRoutePaths.textualQuery` | ? |
| `GraphRoutePaths.textualQueryEdit` | ? |
| `GraphRoutePaths.projectFiles` | ? |
| `GraphRoutePaths.resourceManagementList` | ? |
| `GraphRoutePaths.report` | ? |
| `GraphRoutePaths.reportBuilder` | ? |
| `GraphRoutePaths.dashboard` | ? |
| `GraphRoutePaths.plan` | ? |
| `GraphRoutePaths.rawGraph` | ? |
| `GraphRoutePaths.snapshots` | ? |
| `GraphRoutePaths.dataHistory` | ? |
| `SoxRoutePaths.evidence_testing` | `evidence_testing` |
| `SoxRoutePaths.test_form` | `test_form` |
| `SoxRoutePaths.exportList` | `export_list` |
| `SoxRoutePaths.bulkTestFormImport` | `bulk_test_form_import` |
| `SoxRoutePaths.sampling` | `select_samples` |
| `SoxRoutePaths.graphAttachmentViewer` | ? |
| `SoxRoutePaths.graphMarkupViewer` | ? |

**Dependencies:** Access to graph_ui repo

---

### TICKET: GRAPH-005
**Title:** Determine pipeline ownership and register in rmconsole
**Type:** Task
**Priority:** High
**Component:** graph_app
**Labels:** `mfe-migration`, `infrastructure`
**Story Points:** 1 (1 day -- coordination + registration)

**Description:**
Coordinate with release-eng to determine:
1. Who owns the graph_app pipeline registration in rmconsole -- the
   graph_app team or release-eng?
2. Register the pipeline entity in rmconsole (initially empty; stages
   added in Phase 3)
3. Confirm the `VerifySmithySucceeded` step works for GHA-built repos,
   or identify the equivalent step (`VerifyGHASucceeded`?)

**Acceptance Criteria:**
- [ ] Pipeline entity registered in rmconsole
- [ ] Ownership documented
- [ ] GHA verification step confirmed or alternative identified
- [ ] Deployment window confirmed (same as form_config Mon-Thu 8:50 PM
      CST, or different?)

**Dependencies:** None

---

### TICKET: GRAPH-006
**Title:** Design archive mode and draft session mode strategy for MFE
**Type:** Spike
**Priority:** High
**Component:** graph_app
**Labels:** `mfe-migration`, `spike`, `architecture`
**Story Points:** 3 (3 days -- evaluate options, build POC, document decision)

**Description:**
Today wdesk uses three different `ExperienceRegistry` classes based on
URL mode:
- Normal: `WdeskExperienceRegistry` (full experience set)
- Archive: `IrArchiveModeExperienceRegistry` (reduced set + archive header)
- Draft session: `IRDraftSessionExperienceRegistry` (different reduced
  set + draft edits header)

Post-decoupling, graph_app's MFE must handle these modes internally.
Evaluate and decide between:

**Option A (Recommended):** graph_app MFE detects mode from URL params
and conditionally registers contributions in `WsoxExtension`.

**Option B:** Multiple MFE manifests (`w_sox`, `w_sox_archive`,
`w_sox_draft`).

**Option C:** FEWS manifest conditional contributions (if supported).

Also address sidebar headers:
- `graph_archive_header` and `draft_edits_header` must move into
  graph_app's MFE bundle or be contributed via manifest.

**Acceptance Criteria:**
- [ ] Option selected with rationale documented
- [ ] POC demonstrates the chosen approach works
- [ ] Sidebar header contribution mechanism confirmed
- [ ] Edge cases documented (e.g., `DataDrawerExperienceConfig(isDraftSession: true)`
      constructor parameter passing)

**Dependencies:** GRAPH-003 (FEWS manifest capabilities)

---

## EPIC-1.2: Phase 1 -- graph_app MFE Manifest Migration

> **Goal:** Transform graph_app from a legacy app-style deployment to a
> full MFE with manifest-declared contributions. graph_app continues to
> work via the compile-time path during this phase.

---

### TICKET: GRAPH-010
**Title:** Create MFE entry point using `createMfe()` pattern
**Type:** Story
**Priority:** Critical
**Component:** graph_app
**Labels:** `mfe-migration`, `core`
**Story Points:** 1 (1 day -- Claude generates entry point from form_config/assessments_client pattern, human reviews and tests locally)

**Description:**
Create a new MFE entry point at `app/web/w_sox_app.mfe.dart` that uses
the `createMfe()` pattern (matching form_config and assessments_client)
instead of the legacy `createApp()` pattern in `app/web/main.dart`.

**Tasks:**
- [ ] Create `app/web/w_sox_app.mfe.dart` with `createMfe()` call
- [ ] Set `intlName: 'w_sox'` and `mfeName: 'w_sox'`
- [ ] Call `writeMfeBuildMetadataToWindow()` instead of
      `writeAppBuildMetadataToWindow()`
- [ ] Initialize LaunchDarkly as needed
- [ ] Handle archive mode / draft session mode detection (per GRAPH-006
      decision)

**Reference implementations:**
- `form_config/form_config/web/form_config_example.mfe.dart`
- `assessments_client/web/assessments.mfe.dart`

**Acceptance Criteria:**
- [ ] MFE entry point compiles successfully
- [ ] `createMfe()` initializes correctly in local dev
- [ ] Build metadata written to window
- [ ] Asset loader configured for both local and deployed environments

**Dependencies:** GRAPH-006

---

### TICKET: GRAPH-011
**Title:** Create `WsoxExtension` class with all experience contributions
**Type:** Story
**Priority:** Critical
**Component:** graph_app
**Labels:** `mfe-migration`, `core`
**Story Points:** 2 (2 days -- Claude scaffolds all 27 contributions + 10 widgets from experience_configs.dart, human reviews and wires up service registration)

**Description:**
Create a `WsoxExtension` class extending `WdeskExtension` that registers
all 27+ experience contributions via `onRegisterContributions()`. This
replaces the compile-time registration in wdesk's
`experience_registry.dart`.

**Contributions to register:**

*Drawer experiences (9):*
- ResourceManagementExperienceConfig
- OverviewExperienceConfig
- ReportListExperienceConfig
- DashboardListExperienceConfig
- DataDrawerExperienceConfig
- DataModelExperienceConfig
- SupportExperienceConfig
- TextualQueryExperienceConfig
- ProjectFilesExperienceConfig

*Rich experiences (16):*
- FocusExperienceConfig, FocusNewExperienceConfig
- ExportListExperienceConfig, EvidenceTestingExperienceConfig
- ReportExperienceConfig, ReportBuilderExperienceConfig
- DashboardRichExperienceConfig, TestFormRichExperienceConfig
- SamplingRichExperienceConfig, TextualQueryEditExperienceConfig
- BulkTestFormImportExperienceConfig
- GraphAttachmentViewerExperienceConfig, GraphMarkupViewerExperienceConfig
- RawGraphExperienceConfig, SuggestedPermissionsExperienceConfig
- ResourcePlanExperienceConfig

*Embeddable experiences (2):*
- createSampleSelectionExperienceConfig()
- createTestFormSpreadsheetExperienceConfig()

*Landing page widgets (10):*
- ChartWidgetConfig, DataTypesWidgetConfig
- TestFormRecencyWidgetConfig, AuditFormRecencyWidgetConfig
- ProcedureFormRecencyWidgetConfig, IssueFormRecencyWidgetConfig
- ActionPlanFormRecencyWidgetConfig, ReportRecencyWidgetConfig
- AssignedRequestsWidgetConfig, KeyResourcesWidgetConfig

**Reference implementations:**
- `form_config/form_config/web/extensions/form_config_example/extension.dart`
- `assessments_client/lib/src/mfe/extension.dart`

**Acceptance Criteria:**
- [ ] All 27+ experience contributions registered
- [ ] All 10 landing page widgets registered (if FEWS supports it;
      otherwise create workaround per GRAPH-003 findings)
- [ ] Archive mode and draft session mode handled per GRAPH-006 decision
- [ ] Static asset loading works for all experiences
- [ ] Service registrations (graph client, session, messaging) included
      in `onRegisterServices()`

**Dependencies:** GRAPH-003, GRAPH-006, GRAPH-010

---

### TICKET: GRAPH-012
**Title:** Migrate `manifest.yaml` from legacy app format to full MFE
**Type:** Story
**Priority:** Critical
**Component:** graph_app
**Labels:** `mfe-migration`, `core`
**Story Points:** 1 (1 day -- Claude generates full manifest from route mapping + experience configs, human validates access control expressions and ordering)

**Description:**
Replace the current minimal `app/web/manifest.yaml`:
```yaml
version: 1
app:
  name: w_sox_app
```

With a full MFE manifest declaring all contributions, routing, and
navigation sidebar entries. Use literal route segment strings from
GRAPH-004.

**Sections to include:**
- `microfrontends.w_sox.apps: [wdesk]`
- `extensions.w_sox.entrypoint: w_sox_app.mfe.dart.js`
- `contributions.core.drawer_experiences` (9 entries)
- `contributions.core.rich_experiences` (16 entries)
- `contributions.core.routing` (25+ entries)
- `contributions.core.navigation_sidebar` (9 entries with icons, text,
  location/ordering)
- `contributions.core.embeddable_experiences` (if supported)
- `contributions.core.landing_page_widgets` (if supported)

**Acceptance Criteria:**
- [ ] Manifest passes FEWS validation
- [ ] All `can_user_access` expressions match current licensing checks
- [ ] Route segments match current `GraphRoutePaths` / `SoxRoutePaths`
      literal values exactly
- [ ] Sidebar ordering matches current `DrawerExperienceOrdering` values
- [ ] Sidebar icons match current `navItemIcon` values
- [ ] Display names match current i18n strings

**Dependencies:** GRAPH-003, GRAPH-004

---

### TICKET: GRAPH-013
**Title:** Update `app/build.yaml` for MFE builder configuration
**Type:** Task
**Priority:** High
**Component:** graph_app
**Labels:** `mfe-migration`
**Story Points:** 0.5 (half day -- Claude generates config, human verifies build output)

**Description:**
Update `app/build.yaml` to enable `wdesk_sdk_builders|mfe` for the new
MFE entry point. Ensure the build produces the correct compiled JS output
(`w_sox_app.mfe.dart.js`).

**Acceptance Criteria:**
- [ ] `wdesk_sdk_builders|mfe` builder enabled for new entry point
- [ ] `build_meta.g.dart` generated correctly for MFE context
- [ ] `cdn_assets.tar.gz` includes the MFE compiled output
- [ ] Existing `main.dart` entry point still builds (dual-path support
      during transition)

**Dependencies:** GRAPH-010

---

### TICKET: GRAPH-014
**Title:** Local validation of graph_app MFE in wdesk via FEWS override
**Type:** Story
**Priority:** High
**Component:** graph_app
**Labels:** `mfe-migration`, `testing`
**Story Points:** 2 (2 days -- human deploys and clicks through 27 experiences, Claude fixes issues found)

**Description:**
Deploy graph_app MFE to wk-dev and verify it loads correctly in wdesk
using the FEWS override URL pattern:
```
https://wk-dev.wdesk.org/fews/v1/serve/wdesk+cdn-dev:graph_app@<BUILD_ID>
```

**Acceptance Criteria:**
- [ ] All 9 drawer experiences appear in left navigation
- [ ] All 16 rich experiences render when navigated to
- [ ] Routing works for all route segments (deep links)
- [ ] Sidebar icons and ordering match current production
- [ ] Static assets load correctly (CSS, JS, CodeMirror, highcharts)
- [ ] Archive mode works (if applicable per GRAPH-006)
- [ ] Draft session mode works (if applicable per GRAPH-006)
- [ ] Landing page widgets render (if applicable per GRAPH-003)
- [ ] No console errors related to MFE loading

**Dependencies:** GRAPH-010, GRAPH-011, GRAPH-012, GRAPH-013

---

## EPIC-1.3: Phase 2 -- Dual-Path Validation

> **Goal:** Confirm that both the compile-time path (current) and the MFE
> manifest path (new) work simultaneously with feature parity.

---

### TICKET: GRAPH-020
**Title:** Deploy graph_app MFE to wk-dev and staging for dual-path testing
**Type:** Story
**Priority:** High
**Component:** graph_app
**Labels:** `mfe-migration`, `testing`
**Story Points:** 2 (2 days -- deploy both envs, verify no conflicts)

**Description:**
Deploy graph_app as an MFE to wk-dev and staging environments while
wdesk still bundles w_sox at compile time. Verify:
1. wdesk with compile-time w_sox works (no regression)
2. wdesk with MFE-contributed graph_app works (new path)
3. No conflicts between compile-time and MFE experiences (the
   `RichExperienceRegistryShim` should handle dedup)

**Acceptance Criteria:**
- [ ] wk-dev: both paths verified
- [ ] staging: both paths verified
- [ ] No duplicate experiences in navigation
- [ ] No routing conflicts
- [ ] Functional test suite passes against both paths

**Dependencies:** GRAPH-014

---

### TICKET: GRAPH-021
**Title:** Run full functional test suite against MFE-contributed graph_app
**Type:** Story
**Priority:** High
**Component:** graph_app
**Labels:** `mfe-migration`, `testing`
**Story Points:** 3 (3 days -- run all suites, compare results, triage diffs)

**Description:**
Run the complete graph_app functional test suite (controls testing,
evidence testing, export, planning, reports, dashboards, comments
integration) against the MFE-contributed path. Compare results with
the compile-time path to confirm feature parity.

**Test suites to run:**
- Controls testing
- Evidence testing experience
- Export
- Planning
- Reports and dashboards
- Comments integration
- LaunchDarkly flag testing
- Data experience

**Acceptance Criteria:**
- [ ] All functional test suites pass via MFE path
- [ ] Test results match compile-time path results
- [ ] Any differences documented and triaged
- [ ] Performance comparison documented (MFE load time vs compile-time)

**Dependencies:** GRAPH-020

---

## EPIC-1.4: Phase 3 -- Release Pipeline Setup

> **Goal:** Establish graph_app's independent release pipeline in rmconsole
> with promotion gates and rollback automation.

---

### TICKET: GRAPH-030
**Title:** Add `pipeline_template.yaml` to graph_app repo
**Type:** Story
**Priority:** Critical
**Component:** graph_app
**Labels:** `mfe-migration`, `pipeline`
**Story Points:** 1 (1 day -- Claude already generated the template, human populates plan IDs and registers in rmconsole)

**Description:**
Create `pipeline_template.yaml` in graph_app repo root with the following
stages (modeled on form_config's pipeline):

1. Publish Build Artifacts (gate: tagged, CI passed, QA approved, CDN
   assets available)
2. Internal deploys (wk-dev + staging, parallel, immediate)
3. Staging Testing (Signals plans from GRAPH-002)
4. Deploy to Pentest (immediate, with rollback on failure)
5. Deploy to Sandbox (immediate, with rollback on failure)
6. Deploy to Demo (Mon-Thu 8:50 PM CST window, with rollback)
7. Deploy to APAC (StandardEU window, with rollback)
8. Deploy to EU (StandardEU window, with rollback)
9. Deploy to Prod (Mon-Thu 8:50 PM CST window, with rollback)
10. Close Tickets

Each deployment stage includes:
- `DeployToFews` with `apps: wdesk`
- `on_fail`: Slack notification to `#alert-graph-app` -> Stop (600s) ->
  `RollbackFewsDeploy` -> post-rollback Signals verification
- Post-deploy `StartSignalsRun` with per-environment plan ID

**Acceptance Criteria:**
- [ ] Pipeline template validates in rmconsole
- [ ] All Signals plan IDs from GRAPH-002 populated
- [ ] Slack channel `#alert-graph-app` referenced in all `on_fail` handlers
- [ ] `@grc-flowmasters-team` mentioned for prod/APAC/EU failures
- [ ] Deployment windows match team agreement from GRAPH-005

**Dependencies:** GRAPH-002, GRAPH-005

---

### TICKET: GRAPH-031
**Title:** Dry-run pipeline promotion through wk-dev -> staging -> pentest
**Type:** Story
**Priority:** High
**Component:** graph_app
**Labels:** `mfe-migration`, `pipeline`, `testing`
**Story Points:** 2 (2 days -- promote through 3 stages, verify each gate)

**Description:**
Perform a dry-run promotion of graph_app through its pipeline to validate
the stage gates, Signals integration, and Slack alerting.

**Acceptance Criteria:**
- [ ] Build artifacts published successfully
- [ ] wk-dev deployment succeeds
- [ ] staging deployment succeeds
- [ ] Staging Signals tests run and pass
- [ ] pentest deployment succeeds
- [ ] Slack alerts delivered for test failures (intentionally trigger one)
- [ ] Pipeline can be manually paused and resumed

**Dependencies:** GRAPH-030

---

### TICKET: GRAPH-032
**Title:** Validate rollback automation with intentional bad deploy
**Type:** Story
**Priority:** High
**Component:** graph_app
**Labels:** `mfe-migration`, `pipeline`, `testing`
**Story Points:** 1 (1 day -- deploy bad build, observe rollback, document)

**Description:**
Intentionally deploy a known-bad build to pentest to validate:
1. Signals tests detect the failure
2. Slack notification fires
3. Pipeline pauses for 600s
4. `RollbackFewsDeploy` reverts to previous version
5. Post-rollback Signals run confirms the rollback is healthy

**Acceptance Criteria:**
- [ ] Rollback completes automatically
- [ ] Previous version restored and verified
- [ ] Full rollback cycle documented in runbook
- [ ] Rollback time measured (target: < 15 minutes total)

**Dependencies:** GRAPH-031

---

## EPIC-1.5: Phase 4 -- wdesk Decoupling

> **Goal:** Remove all compile-time w_sox references from wdesk. This is
> the big-bang PR -- all changes must land together.

---

### TICKET: GRAPH-040
**Title:** Remove `w_sox` dependency and all references from wdesk
**Type:** Story
**Priority:** Critical
**Component:** wdesk
**Labels:** `mfe-migration`, `decoupling`, `breaking-change`
**Story Points:** 2 (2 days -- Claude generates the complete PR diff for all 9 files, human reviews and runs CI)

**Description:**
Single PR that removes all compile-time coupling between wdesk and
w_sox (graph_app). This PR must only be merged AFTER Phase 2 confirms
the MFE path works with feature parity.

**Files to modify (9 total, ~103 references):**

| File | Changes |
|---|---|
| `pubspec.yaml` | Remove `w_sox: ^10.4.29` dependency |
| `lib/src/experience_registry.dart` | Remove 2 imports, 25 experience configs, 10 landing page widgets |
| `lib/src/ir_archive_mode_experience_registry.dart` | Remove import + 15 w_sox references |
| `lib/src/ir_draft_session_experience_registry.dart` | Remove import + 16 w_sox references |
| `web/main.dart` | Remove 4 deferred imports, sidebar header setup, simplify `getExperienceRegistry()` |
| `web/headless.dart` | No changes (keep `isGraphTestEnvironment`) |
| `lib/src/utils.dart` | No changes (keep `isGraphTestEnvironment`) |
| `test/unit/experience_registry_test.dart` | Remove ir_widgets import + 10 assertions |
| `.github/workflows/frontend-integration.yaml` | Update consumer test for MFE path |

**Sub-tasks:**

| Sub-task | Description |
|---|---|
| GRAPH-040a | Remove `w_sox` from `pubspec.yaml` and run `dart pub get` |
| GRAPH-040b | Clean `experience_registry.dart` (25 configs + 10 widgets) |
| GRAPH-040c | Clean `ir_archive_mode_experience_registry.dart` (15 refs) |
| GRAPH-040d | Clean `ir_draft_session_experience_registry.dart` (16 refs) |
| GRAPH-040e | Clean `web/main.dart` (deferred imports, headers, registry selection) |
| GRAPH-040f | Update `experience_registry_test.dart` (assertions + counts) |
| GRAPH-040g | Update `frontend-integration.yaml` CI workflow |
| GRAPH-040h | Run `dart analyze` and fix any remaining references |

**Acceptance Criteria:**
- [ ] `dart pub get` succeeds without w_sox
- [ ] `dart analyze` passes with zero errors
- [ ] `dart run dart_dev test` passes
- [ ] wdesk builds and deploys to wk-dev successfully
- [ ] All graph experiences load via MFE path in wk-dev
- [ ] All non-graph experiences (audit, markup, viewer, etc.) unaffected
- [ ] CI consumer integration test updated and passing
- [ ] No compile-time references to `w_sox` or `ir_widgets` remain

**Dependencies:** GRAPH-020, GRAPH-021 (Phase 2 validation complete)
**Blocks:** GRAPH-050 (production cutover)

**CRITICAL WARNING:** This PR MUST NOT be merged until:
1. graph_app MFE is deployed and verified in wk-dev and staging
2. All functional tests pass via the MFE path
3. Landing page widget migration confirmed (or workaround in place)
4. Archive mode and draft session mode working via MFE

---

### TICKET: GRAPH-041
**Title:** Handle landing page widget database key migration
**Type:** Story
**Priority:** High
**Component:** wdesk
**Labels:** `mfe-migration`, `data-migration`
**Story Points:** 1 (1 day -- Claude generates widget migration code, human validates against real dashboard data)

**Description:**
The 10 IR landing page widgets use string keys persisted in the
view-settings database:
```
ir_charts, ir_primary_data_types, ir_recent_test_forms,
ir_recent_audit_forms, ir_recent_procedure_forms,
ir_recent_issue_forms, ir_recent_action_plan_forms,
ir_recent_reports, ir_assigned_requests, ir_key_resources
```

Removing these from wdesk without providing them via MFE manifest
will break existing user landing page dashboards.

**Options:**
1. If FEWS manifest supports `core.landing_page_widgets`: graph_app's
   manifest declares widgets with the **same string keys**
2. If not supported: create a lightweight Dart package with just the
   widget configs that wdesk can depend on (not the full w_sox package)
3. Graceful degradation: wdesk shows a placeholder for unknown widget
   keys with a "Widget no longer available" message

**Acceptance Criteria:**
- [ ] Existing user dashboards continue to render IR widgets
- [ ] Widget keys preserved exactly (database compatibility)
- [ ] No data migration required on the database side
- [ ] Approach validated in wk-dev with real user dashboard data

**Dependencies:** GRAPH-003 (FEWS manifest capabilities)

---

## EPIC-1.6: Phase 5 -- Production Cutover & Monitoring

> **Goal:** Promote both repos through production and monitor for issues.

---

### TICKET: GRAPH-050
**Title:** Production cutover: promote graph_app MFE to all environments
**Type:** Story
**Priority:** Critical
**Component:** graph_app
**Labels:** `mfe-migration`, `production`
**Story Points:** 2 (2 days -- human triggers and monitors pipeline through 8 environments)

**Description:**
Promote graph_app through its independent pipeline to all production
environments:
wk-dev -> staging -> pentest -> sandbox -> demo -> APAC -> EU -> prod

**Acceptance Criteria:**
- [ ] All pipeline stages pass
- [ ] All Signals tests pass in every environment
- [ ] No rollbacks triggered
- [ ] graph_app serving independently in production

**Dependencies:** GRAPH-032 (rollback validated)

---

### TICKET: GRAPH-051
**Title:** Merge wdesk decoupling PR and promote to production
**Type:** Story
**Priority:** Critical
**Component:** wdesk
**Labels:** `mfe-migration`, `production`
**Story Points:** 2 (2 days -- merge, promote through wdesk pipeline, verify)

**Description:**
After graph_app is independently deployed to production (GRAPH-050),
merge the wdesk decoupling PR (GRAPH-040) and promote wdesk through
its pipeline.

**Sequencing:**
1. graph_app MFE must be live in ALL environments first
2. Merge wdesk decoupling PR
3. wdesk CI builds without w_sox
4. Promote wdesk through its own pipeline
5. Each wdesk environment now loads graph_app via MFE manifest

**Acceptance Criteria:**
- [ ] wdesk builds and deploys without w_sox dependency
- [ ] All graph experiences load via MFE in every environment
- [ ] No 404s on graph deep links
- [ ] No broken landing page dashboards
- [ ] wdesk build size decreased (w_sox no longer compiled in)

**Dependencies:** GRAPH-040, GRAPH-050

---

### TICKET: GRAPH-052
**Title:** Post-cutover monitoring (1 week bake period)
**Type:** Task
**Priority:** High
**Component:** graph_app
**Labels:** `mfe-migration`, `monitoring`
**Story Points:** 5 (5 days -- daily monitoring checks for 1 week)

**Description:**
Monitor production for 1 week after cutover:

**Metrics to track:**
- [ ] Error rates in graph_app experiences (Grafana / observability)
- [ ] MFE load time vs previous compile-time load time
- [ ] Signals test pass rates across all environments
- [ ] User-reported issues via support channels
- [ ] Landing page widget rendering success rate
- [ ] Deep link resolution success rate

**Acceptance Criteria:**
- [ ] No increase in error rates vs pre-migration baseline
- [ ] MFE load time within acceptable threshold (< 2s)
- [ ] Zero P1/P2 incidents attributed to migration
- [ ] Week-long bake period completed with green signals

**Dependencies:** GRAPH-051

---

## EPIC-1.7: Phase 6 -- Cleanup

> **Goal:** Remove transitional code and finalize the migration.

---

### TICKET: GRAPH-060
**Title:** Remove legacy `createApp()` entry point from graph_app
**Type:** Task
**Priority:** Medium
**Component:** graph_app
**Labels:** `mfe-migration`, `cleanup`
**Story Points:** 0.5 (half day -- Claude generates cleanup PR, human reviews)

**Description:**
After the MFE path is the sole production path, remove:
- `app/web/main.dart` (legacy `createApp()` entry point)
- Any build config that references the legacy entry point
- The `app: {name: w_sox_app}` section from manifest.yaml if still present

Keep the MFE entry point (`app/web/w_sox_app.mfe.dart`) as the sole
entry point.

**Acceptance Criteria:**
- [ ] Only MFE entry point remains
- [ ] Build produces only MFE output
- [ ] All tests pass
- [ ] No dead code referencing `createApp()`

**Dependencies:** GRAPH-052 (bake period complete)

---

### TICKET: GRAPH-061
**Title:** Evaluate Docker image build necessity post-MFE migration
**Type:** Spike
**Priority:** Low
**Component:** graph_app
**Labels:** `mfe-migration`, `cleanup`
**Story Points:** 0.5 (half day -- Claude analyzes Docker usage across CI, human decides)

**Description:**
graph_app currently builds a Docker image via `Dockerfile-wdeskapp`
(based on `wdesk_sdk_app_server:3` / nginx). Post-MFE migration, assets
are served via FEWS/CDN. Determine if the Docker image is still needed
for functional testing or can be removed.

**Acceptance Criteria:**
- [ ] Document which processes depend on the Docker image
- [ ] If only functional tests: explore switching tests to FEWS-served
      assets
- [ ] Decision documented: keep, modify, or remove Docker build step

**Dependencies:** GRAPH-052

---

### TICKET: GRAPH-062
**Title:** Update graph_app documentation and runbooks for independent release
**Type:** Task
**Priority:** Medium
**Component:** graph_app
**Labels:** `mfe-migration`, `documentation`
**Story Points:** 1 (1 day -- Claude drafts all docs, human reviews and publishes)

**Description:**
Update:
- [ ] README.md with new deployment model
- [ ] Runbook for independent release process
- [ ] On-call playbook for graph_app pipeline failures
- [ ] Developer onboarding docs (how to test MFE locally, FEWS override
      URLs, etc.)
- [ ] Architecture decision record (ADR) documenting the migration

**Dependencies:** GRAPH-052

---

## Summary: Ticket Counts and Estimates

> **Scale:** 1 point = 1 day (8 hours)
>
> **Estimates assume Claude-assisted development** -- code generation,
> manifest authoring, config scaffolding, and PR prep are accelerated.
> Human time is primarily review, coordination, and deployment operations.

| Phase | Tickets | Points (days) | What Claude does | What humans do |
|---|---|---|---|---|
| Phase 0: Preparation | 6 | 5 | Route mapping, documentation | Slack channel, Signals plans, rmconsole registration, FEWS spike |
| Phase 1: MFE Manifest | 5 | 6 | Generate entry point, extension class, manifest.yaml, build config | Review, test locally, iterate |
| Phase 2: Dual-Path Validation | 2 | 2 | Diff analysis, test result comparison | Deploy to envs, run functional tests |
| Phase 3: Pipeline Setup | 3 | 3 | Generate pipeline_template.yaml | Register in rmconsole, run promotions, validate rollback |
| Phase 4: wdesk Decoupling | 2 | 3 | Generate full PR diff (9 files, 103 refs), widget migration code | Review PR, coordinate merge timing |
| Phase 5: Production Cutover | 3 | 7 | Monitoring dashboards, runbook | Promote pipelines, 5-day bake period (passive) |
| Phase 6: Cleanup | 3 | 2 | Dead code removal PR, ADR draft | Review, merge |
| **Total** | **24 tickets** | **28 days** | | |

### Calendar projection

| Scenario | Wall-clock time |
|---|---|
| 1 engineer + Claude | ~6-7 weeks |
| 2 engineers + Claude | ~4-5 weeks |

**What Claude accelerates vs what it can't:**

| Claude handles (fast) | Human-gated (irreducible) |
|---|---|
| Manifest.yaml with all 27 contributions | FEWS capability spike (need platform team answers) |
| WsoxExtension class scaffolding | Signals plan creation (manual in Skynet UI) |
| pipeline_template.yaml generation | rmconsole pipeline registration (manual) |
| wdesk decoupling PR (all 9 files) | Deployment promotion through environments |
| Route mapping table | 5-day production bake period |
| ADR and documentation drafts | Cross-team coordination (release-eng, wdesk_sdk) |
| Test comparison analysis | Rollback validation (intentional bad deploy) |

**Bottom line:** ~60% of the raw coding/authoring work can be Claude-
generated in hours, not days. The calendar is dominated by deployment
operations, environment promotions, Signals test creation, and the
mandatory bake period -- things that require human hands on production
systems.

## Critical Path

```
GRAPH-003 (FEWS capabilities)
    |
    +---> GRAPH-006 (mode strategy)
    |       |
    |       +---> GRAPH-010 (MFE entry point)
    |               |
    |               +---> GRAPH-011 (WsoxExtension)
    |               +---> GRAPH-013 (build.yaml)
    |
    +---> GRAPH-012 (manifest.yaml)
            |
            +---> GRAPH-014 (local validation)
                    |
                    +---> GRAPH-020 (dual-path deploy)
                            |
                            +---> GRAPH-021 (full test suite)
                                    |
                                    +---> GRAPH-040 (wdesk decoupling)
                                            |
                                            +---> GRAPH-050 (prod cutover)
                                                    |
                                                    +---> GRAPH-051 (wdesk prod)
```

**The critical path runs through:** FEWS validation -> mode strategy ->
MFE entry point -> manifest -> validation -> wdesk decoupling -> production.

**Parallel work:** GRAPH-001 (Slack), GRAPH-002 (Signals plans),
GRAPH-004 (route mapping), GRAPH-005 (rmconsole) can all run in parallel
with the critical path during Phase 0.
