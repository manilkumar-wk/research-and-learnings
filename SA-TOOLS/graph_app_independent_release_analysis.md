# graph_app Independent Release Pipeline: Full Analysis

## TL;DR

- **graph_app (w_sox) already has a mature CI/CD pipeline** (GHA-based, CDN
  publishing, FEWS deploy, Docker image, functional tests) but **lacks an
  rmconsole pipeline_template.yaml** for independent environment promotion --
  that is the single biggest gap.
- **wdesk consumes graph_app two ways**: (1) `w_sox ^10.4.29` as a Dart pub
  dependency for experience configs, and (2) `gha-integration-testing` in CI
  that pulls `release:latest` of graph_app -- both must be addressed.
- **graph_app uses a legacy `app/web/manifest.yaml`** (app-style:
  `{version: 1, app: {name: w_sox_app}}`) instead of the modern MFE manifest
  pattern used by form_config and assessments_client -- this must be migrated
  to declare contributions, routing, and sidebar entries.
- **The wdesk experience_registry.dart hardcodes ~25 w_sox experience configs**
  -- post-decoupling, these will be contributed dynamically via the MFE
  manifest (the form_config pattern), eliminating compile-time coupling.
- **Sequencing is critical**: manifest migration and pipeline_template.yaml
  must land before the wdesk decoupling PR, with a dual-release overlap
  window where both paths work.

---

## Part A: graph_app Current State Analysis

### A.1 Identity and Package Structure

| Property | Value |
|---|---|
| **Package name** | `w_sox` |
| **Version** | `10.4.40` |
| **Pub registry** | `https://pub.workiva.org` |
| **Dart SDK** | `>=2.19.0 <3.0.0` |
| **Repository** | `https://github.com/Workiva/graph_app` |
| **Description** | The client-side portion of the Workiva SOX solution |
| **Null safety** | Disabled (build hack: `sound_null_safety: false`) |

The repo contains two packages:
- **Root (`w_sox`)**: Library package exporting experience configs, feature
  modules, and domain logic. Published to pub.workiva.org.
- **App (`w_sox_app`)**: Application shell in `app/` with web entry point.
  `publish_to: none`. Produces the compiled JS/Docker artifacts.

### A.2 Existing CI/CD Pipeline (GitHub Actions)

graph_app **already has a comprehensive CI/CD pipeline** in
`.github/workflows/ci.yml`. This is a strength -- not starting from zero.

**Pipeline stages today:**

```
pre-build (BUILD_ID, pub cache)
    |
    +---> test (unit tests, analysis, linting, semver audit)
    +---> intl-check
    +---> main-pub-package (publish to pub.workiva.org)
    +---> generate-lockfiles
    +---> generate-assets (dart2js compile -> cdn_assets.tar.gz)
              |
              +---> publish-to-cdn (gha-deploy-component -> CDN)
              +---> build-docker (Dockerfile-wdeskapp -> drydock registry)
              +---> deploy (gha-dart/deploy-to-fews -> FEWS)
                        |
                        +---> mark-build-done
                        +---> verify-build
              +---> functional-tests (Docker-based integration suite)
```

**What exists:**
- CDN asset publishing: `https://cdn-prod.wdesk.com/graph_app/<tag>/`
- Docker image: based on `wdesk_sdk_app_server:3` (nginx)
- FEWS deployment: via `app/web/manifest.yaml`
- Pub package: `w_sox` published on each build
- Functional tests: extensive suite (controls, evidence, export, planning,
  reports, dashboards, comments integration)
- Consumer integration branches for upstream deps (highcharts, wdesk_sdk,
  w_table, etc.)
- Automated version bumping via Rosie

### A.3 What's MISSING for Independent Release

| Gap | Severity | Detail |
|---|---|---|
| **No `pipeline_template.yaml`** | [BREAKING] | Cannot be registered in rmconsole without this. form_config has one with 9 stages. |
| **Legacy manifest format** | [BREAKING] | `app/web/manifest.yaml` is `{version: 1, app: {name: w_sox_app}}` -- a bare app declaration. No MFE contributions, routing, or sidebar entries declared. form_config and assessments_client use the full MFE manifest with contribution declarations. |
| **No Signals test plans** | [HIGH] | form_config has 8 Signals plan IDs for per-environment post-deploy testing. graph_app has functional tests but they run in CI only, not as rmconsole promotion gates. |
| **No per-environment deploy stages** | [HIGH] | No deployment to wk-dev -> staging -> pentest -> sandbox -> demo -> APAC -> EU -> prod. Currently rides wdesk's pipeline. |
| **No rollback automation** | [HIGH] | form_config has `RollbackFewsDeploy` with automatic Signals re-verification. graph_app has nothing. |
| **No Slack alerting channel** | [MEDIUM] | form_config alerts `alert-form-config`. graph_app needs its own (e.g., `alert-graph-app`). |
| **No CHANGELOG** | [LOW] | Relies on git history. Not blocking but important for release notes. |

### A.4 Dependency Profile

**Total Workiva dependencies:** 70+ packages from `pub.workiva.org`.

**Critical SDK coupling:**
- `wdesk_sdk: ^9.25.2` -- app infrastructure, experience framework
- `wdesk_browser_environment: ^1.17.52` -- browser detection
- `wdesk_sdk_builders: ^2.1.87` -- MFE code generation (dev dep)

**Graph ecosystem (owned):**
- `graph_api: ^16.202.2`
- `graph_form_api: ^3.181.4`
- `graph_rpc_api: ^5.1.15`
- `graph_ui: ^38.3.5`
- `w_graph_client: ^10.2.10`

**This coupling is architectural, not accidental.** graph_app is designed as
an MFE within the wdesk experience framework. Independent release does NOT
mean removing the wdesk_sdk dependency -- it means graph_app controls its own
deploy cadence rather than being gated on wdesk's release cycle.

### A.5 Experience Config Surface (API Contract)

`lib/experience_configs.dart` exports **25 experience configs** that wdesk
consumes. This is the contract surface:

**Drawer experiences (navigation):**
- ResourceManagementExperienceConfig, OverviewExperienceConfig
- ReportListExperienceConfig, DashboardListExperienceConfig
- DataDrawerExperienceConfig, DataModelExperienceConfig
- SupportExperienceConfig, TextualQueryExperienceConfig
- ProjectFilesExperienceConfig

**Rich experiences (full-page):**
- ExportListExperienceConfig, FocusExperienceConfig,
  FocusNewExperienceConfig
- SamplingRichExperienceConfig, ReportExperienceConfig,
  DashboardRichExperienceConfig
- TestFormRichExperienceConfig, TextualQueryEditExperienceConfig
- EvidenceTestingExperienceConfig, GraphAttachmentViewerExperienceConfig
- GraphMarkupViewerExperienceConfig, ReportBuilderExperienceConfig
- ResourcePlanExperienceConfig, RawGraphExperienceConfig
- SuggestedPermissionsExperienceConfig, BulkTestFormImportExperienceConfig

**Embeddable experiences:**
- createSampleSelectionExperienceConfig()
- createTestFormSpreadsheetExperienceConfig()

### A.6 Readiness Scorecard

| Dimension | Status | Score |
|---|---|---|
| Build automation | GHA with multi-stage pipeline | 9/10 |
| Test coverage | Unit + functional + consumer integration | 8/10 |
| Asset hosting | CDN + FEWS + Docker | 9/10 |
| Versioning | Semver with Rosie automation | 8/10 |
| Pipeline promotion | **Missing entirely** | 0/10 |
| MFE manifest | **Legacy app-style, not MFE contribution** | 2/10 |
| Rollback capability | **None** | 0/10 |
| Post-deploy verification | **No Signals plans** | 0/10 |
| **Overall readiness** | | **4.5/10** |

The build/test/publish side is mature. The promotion/deploy/verify side is
entirely absent -- that's the work.

---

## Part B: wdesk Consumption Pattern and Blast Radius

### B.1 How wdesk Consumes graph_app Today

There are **three coupling vectors**:

#### Vector 1: Dart Pub Dependency (Compile-Time)

**File:** `wdesk/pubspec.yaml` line 118-122
```yaml
w_sox:
  hosted:
    name: w_sox
    url: https://pub.workiva.org
  version: ^10.4.29
```

wdesk depends on `w_sox` (graph_app's package name) as a regular Dart
dependency. This pulls in all experience configs at compile time. wdesk's
`experience_registry.dart` then **hardcodes** references to 25+ `w_sox.*`
configs:

```dart
import 'package:w_sox/experience_configs.dart' as w_sox;
// ...
w_sox.ReportListExperienceConfig(),
w_sox.DashboardListExperienceConfig(),
w_sox.DataDrawerExperienceConfig(),
// ... 20+ more
```

**Impact:** Every graph_app version bump requires a wdesk dependency update.
Any breaking change in experience configs breaks the wdesk build.

#### Vector 2: CI Consumer Integration Testing

**File:** `wdesk/.github/workflows/frontend-integration.yaml`
```yaml
integration-testing:
  name: graph_app
  steps:
    - uses: Workiva/gha-integration-testing@v4.0.37
      with:
        consumer-repository: Workiva/graph_app
        use-commit-from: release:latest
```

wdesk's CI tests against `release:latest` of graph_app. This is a **required
CI gate** -- wdesk cannot merge if its changes break graph_app.

**Impact:** This is actually healthy coupling for correctness. Post-
decoupling, this test should remain but should test against the independently
deployed graph_app rather than a compile-time bundle.

#### Vector 3: Runtime Test Environment Detection

**File:** `wdesk/lib/src/utils.dart`
```dart
bool get isGraphTestEnvironment =>
    Environment.current.testEnvironment == 'graph_app';
```

**Files:** `wdesk/web/main.dart`, `wdesk/web/headless.dart`

When `isGraphTestEnvironment == true`, wdesk installs mock session credentials
configured for graph_app functional testing (sox.admin@workiva.com).

**Impact:** [LOW] Test-only code. No production impact.

### B.2 Blast Radius Matrix

If graph_app is decoupled today with no migration work, here's what breaks:

| What breaks | Risk | Impact | Mitigation |
|---|---|---|---|
| **wdesk build fails** -- `w_sox` import in experience_registry.dart no longer resolved | [BREAKING] | wdesk cannot build or deploy | Migrate to MFE manifest contributions (form_config pattern) |
| **25 experiences disappear from wdesk** -- hardcoded configs removed | [BREAKING] | All SOX/GRC features vanish from wdesk UI | MFE manifest must declare all contributions before removing compile-time refs |
| **Navigation sidebar loses graph entries** -- no sidebar registration | [BREAKING] | Users can't navigate to any graph feature | Manifest must include `core.navigation_sidebar` entries |
| **Routing breaks** -- route segments not registered | [BREAKING] | Deep links to graph features return 404 | Manifest must include `core.routing` entries |
| **Landing page widgets disappear** -- `ir_widgets.*` from w_sox/landing_page_widgets.dart | [HIGH] | Dashboard widgets (charts, IR-specific) gone from landing page | Requires landing page widget contribution in manifest |
| **Embeddable experiences break** -- `createSampleSelectionExperienceConfig()` etc. | [HIGH] | Features embedding graph views in other contexts fail | Must be declared as embeddable contributions |
| **CI consumer test gate fails** -- integration test against graph_app may need reconfiguration | [MEDIUM] | wdesk PRs blocked until test is updated | Update integration test to test against deployed MFE |
| **Version drift** -- graph_app deploys independently, wdesk may reference stale API | [MEDIUM] | Runtime errors if API contract changes | Semver enforcement + contract testing |
| **Mock session credentials** -- test env detection still works | [LOW] | No impact on production | Can remain as-is |

### B.3 wdesk Experience Registry: Full w_sox Dependency Map

From `wdesk/lib/src/experience_registry.dart`:

**Drawer experiences (8 references):**
```
w_sox.ResourceManagementExperienceConfig()     line 62
w_sox.OverviewExperienceConfig()               line 63
w_sox.ReportListExperienceConfig()             line 75
w_sox.DashboardListExperienceConfig()          line 76
w_sox.DataDrawerExperienceConfig()             line 77
w_sox.DataModelExperienceConfig()              line 80
w_sox.SupportExperienceConfig()                line 81
w_sox.TextualQueryExperienceConfig()           line 85
w_sox.ProjectFilesExperienceConfig()           line 86
```

**Rich experiences (16 references):**
```
w_sox.ExportListExperienceConfig()             line 118
w_sox.FocusExperienceConfig()                  line 119
w_sox.FocusNewExperienceConfig()               line 120
w_sox.BulkTestFormImportExperienceConfig()     line 121
w_sox.SamplingRichExperienceConfig.v2()        line 125
w_sox.ReportExperienceConfig()                 line 126
w_sox.DashboardRichExperienceConfig()          line 127
w_sox.TestFormRichExperienceConfig()            line 128
w_sox.TextualQueryEditExperienceConfig()       line 129
w_sox.EvidenceTestingExperienceConfig()        line 130
w_sox.GraphAttachmentViewerExperienceConfig()  line 131
w_sox.GraphMarkupViewerExperienceConfig()      line 132
w_sox.ReportBuilderExperienceConfig()          line 133
w_sox.ResourcePlanExperienceConfig()           line 134
w_sox.RawGraphExperienceConfig()               line 135
w_sox.SuggestedPermissionsExperienceConfig()   line 136
```

**Embeddable experiences (2 references):**
```
w_sox.createSampleSelectionExperienceConfig()  line 169
w_sox.createTestFormSpreadsheetExperienceConfig() line 170
```

**Landing page widgets (from `w_sox/landing_page_widgets.dart`):**
```
ir_widgets.ChartWidgetConfig()                  line 194
```

**Total: 27 compile-time references that must be replaced by manifest-driven
dynamic contributions.**

---

## Part C: New Pipeline Design

### C.1 Reference Pattern Comparison

| Dimension | form_config | assessments_client | graph_app (current) | graph_app (target) |
|---|---|---|---|---|
| **Manifest style** | Full MFE with contributions | Full MFE with contributions | Legacy app-only | Full MFE with contributions |
| **pipeline_template.yaml** | Yes (9 stages) | No (GHA-only) | No | Yes (modeled on form_config) |
| **Signals plans** | 8 plans (per-environment) | None | None | 8+ plans needed |
| **Rollback automation** | Yes (RollbackFewsDeploy) | No | No | Yes |
| **FEWS deploy** | Yes (wdesk app target) | Yes (wdesk app target) | Yes (wdesk app target) | Yes (wdesk app target) |
| **CDN publishing** | Via build | Via build | Yes (gha-deploy-component) | Keep as-is |
| **Docker image** | No | No | Yes (wdesk_sdk_app_server:3) | Keep as-is |
| **Entry pattern** | `createMfe()` + `WdeskExtension` | `createMfe()` + `WdeskExtension` | `create_app.dart` (legacy) | Migrate to `createMfe()` |
| **Slack alerts** | `alert-form-config` | None | None | `alert-graph-app` |

### C.2 Manifest Migration (Critical Path)

graph_app's current manifest is a bare app declaration:

```yaml
# CURRENT: app/web/manifest.yaml
version: 1
app:
  name: w_sox_app
```

It must become a full MFE manifest. Here's the skeleton modeled on
form_config, covering all 27 experience contributions:

```yaml
# TARGET: app/web/manifest.yaml
version: 1

microfrontends:
  w_sox:
    apps: [ wdesk ]
    extensions:
      w_sox:
        entrypoint: w_sox_app.mfe.dart.js  # compiled MFE entry
        contributions:
          # === Drawer Experiences (left nav) ===
          core.drawer_experiences:
            - name: resource_management
              details:
                display_name: Resource Management
                # Adjust access expression per your licensing model
                can_user_access: "ability.MODULE_GRAPH"
            - name: overview
              details:
                display_name: Overview
                can_user_access: "ability.MODULE_GRAPH"
            - name: report_list
              details:
                display_name: Reports
                can_user_access: "ability.MODULE_GRAPH"
            - name: dashboard_list
              details:
                display_name: Dashboards
                can_user_access: "ability.MODULE_GRAPH"
            - name: data_drawer
              details:
                display_name: Data
                can_user_access: "ability.MODULE_GRAPH"
            - name: data_model
              details:
                display_name: Data Model
                can_user_access: "ability.MODULE_GRAPH"
            - name: support
              details:
                display_name: Support
                can_user_access: "ability.MODULE_GRAPH"
            - name: textual_query
              details:
                display_name: Textual Query
                can_user_access: "ability.MODULE_GRAPH"
            - name: project_files
              details:
                display_name: Project Files
                can_user_access: "ability.MODULE_GRAPH"

          # === Rich Experiences (full-page) ===
          core.rich_experiences:
            - name: export_list
              details:
                display_name: Export List
            - name: focus
              details:
                display_name: Focus
            - name: focus_new
              details:
                display_name: Focus New
            - name: bulk_test_form_import
              details:
                display_name: Bulk Test Form Import
            - name: sampling
              details:
                display_name: Sampling
            - name: report
              details:
                display_name: Report
            - name: dashboard_rich
              details:
                display_name: Dashboard
            - name: test_form
              details:
                display_name: Test Form
            - name: textual_query_edit
              details:
                display_name: Textual Query Edit
            - name: evidence_testing
              details:
                display_name: Evidence Testing
            - name: graph_attachment_viewer
              details:
                display_name: Attachment Viewer
            - name: graph_markup_viewer
              details:
                display_name: Markup Viewer
            - name: report_builder
              details:
                display_name: Report Builder
            - name: resource_plan
              details:
                display_name: Resource Plan
            - name: raw_graph
              details:
                display_name: Raw Graph
            - name: suggested_permissions
              details:
                display_name: Suggested Permissions

          # === Routing ===
          core.routing:
            - name: resource_management_route
              details:
                route_segment: resource-management
                experience_name: w_sox.core.drawer_experiences.resource_management
            - name: overview_route
              details:
                route_segment: overview
                experience_name: w_sox.core.drawer_experiences.overview
            - name: report_list_route
              details:
                route_segment: reports
                experience_name: w_sox.core.drawer_experiences.report_list
            - name: dashboard_list_route
              details:
                route_segment: dashboards
                experience_name: w_sox.core.drawer_experiences.dashboard_list
            # ... additional routes for each experience

          # === Navigation Sidebar ===
          core.navigation_sidebar:
            - name: resource_management
              details:
                icon: unify.accountTree
                text: Resource Management
                url: resource-management
                location: default@10
            - name: reports
              details:
                icon: unify.assessment
                text: Reports
                url: reports
                location: default@11
            - name: dashboards
              details:
                icon: unify.dashboard
                text: Dashboards
                url: dashboards
                location: default@12
            # ... additional sidebar entries
```

**Key notes:**
- `can_user_access` expressions must match the current licensing/ability
  checks in the experience config classes
- Route segments must match what wdesk currently resolves (check
  `w_router` config)
- Sidebar `location` values control ordering -- must match current positions
- The entrypoint file name changes from the app-style `main.dart` to
  an MFE-style `w_sox_app.mfe.dart`

### C.3 Entry Point Migration

**Current** (`app/web/main.dart`):
```dart
import 'package:wdesk_sdk/create_app.dart';  // <-- legacy app creation
void main() async {
  writeBuildMetadataToWindow();
  // ... session/archive detection
  createApp(experienceRegistry, ...);
}
```

**Target** (modeled on form_config/assessments_client):
```dart
import 'package:wdesk_sdk/create_mfe.dart';
import 'package:w_sox_app/build_meta.g.dart';

void main() async {
  writeMfeBuildMetadataToWindow();
  await createMfe(
    assetLoader: assetLoader,
    intlName: 'w_sox',
    mfeName: 'w_sox',
  );
  // Register extension with all experience contributions
  WsoxExtension(assetLoader);
}
```

A new `WsoxExtension` class extending `WdeskExtension` must be created,
registering all contributions via `onRegisterContributions()`.

### C.4 pipeline_template.yaml for graph_app

Modeled directly on form_config's proven pattern:

```yaml
# graph_app/pipeline_template.yaml
stages:
  - name: Publish Build Artifacts
    enabled: true
    steps:
      - name: VerifyReleaseTagged
      - name: VerifySmithySucceeded
        # Note: graph_app uses GHA, not Smithy. This step may need
        # to be replaced with a GHA-specific verification step.
        # Check with release-eng if VerifyGHASucceeded exists.
      - name: RosieReleaseApprovalScan
      - name: PublishBuildArtifacts
      - name: VerifyLatestSkynetResultSuccessful
        # Requires Signals plan IDs to be created first
      - name: VerifyQATestingCompleted
      - name: VerifyCDNAssetsAvailable

  - name: Internal deploys
    enabled: true
    groups:
      - name: Deploy to wk-dev
        steps:
          - name: DeployToFews
            options:
              apps: wdesk
              cluster_name: wk-dev
              delay_days: 0
              deployable_days: []
              deployable_time_in_cst: ''
              max_attempts: 20
              wait_type: Immediately
            on_fail:
              - name: SendNotification
                options:
                  color: red
                  message: "wk-dev - Deployment Failed"
                  slack_channels:
                    - alert-graph-app
      - name: Deploy to staging
        steps:
          - name: DeployToFews
            options:
              apps: wdesk
              cluster_name: staging
              delay_days: 0
              deployable_days: []
              deployable_time_in_cst: ''
              max_attempts: 20
              wait_type: Immediately
            on_fail:
              - name: SendNotification
                options:
                  color: red
                  message: "Staging - Deployment Failed"
                  slack_channels:
                    - alert-graph-app

  - name: Staging Testing
    enabled: true
    groups:
      - name: Functional testing
        steps:
          - name: StartSignalsRun
            options:
              max_attempts: 20
              plan_id: 0000000000  # TODO: Create Signals plan
      # Consider splitting admin/nonadmin like form_config
      # if graph_app has role-differentiated testing

  - name: Deploy to Pentest
    enabled: true
    steps:
      - name: DeployToFews
        options:
          apps: wdesk
          cluster_name: pentest
          delay_days: 0
          deployable_days: []
          deployable_time_in_cst: ''
          max_attempts: 20
          wait_type: Immediately
        on_fail:
          - name: SendNotification
            options:
              color: red
              message: "Pentest - Deployment Failed - commencing rollback"
              slack_channels:
                - alert-graph-app
          - name: Stop
            options:
              automatically_continue_after: 600
          - name: RollbackFewsDeploy
            options:
              apps: wdesk
              cluster_name: pentest
              delay_days: 0
              deployable_days: []
              deployable_time_in_cst: ''
              max_attempts: 20
          - name: StartSignalsRun
            options:
              max_attempts: 20
              plan_id: 0000000000  # TODO: pentest Signals plan
      - name: StartSignalsRun
        options:
          max_attempts: 20
          plan_id: 0000000000  # TODO: pentest Signals plan
        on_fail:
          - name: SendNotification
            options:
              color: red
              message: "Pentest - Signals Tests Failed - commencing rollback"
              slack_channels:
                - alert-graph-app
          - name: Stop
            options:
              automatically_continue_after: 600
          - name: RollbackFewsDeploy
            options:
              apps: wdesk
              cluster_name: pentest
              delay_days: 0
              deployable_days: []
              deployable_time_in_cst: ''
              max_attempts: 20
          - name: StartSignalsRun
            options:
              max_attempts: 20
              plan_id: 0000000000

  - name: Deploy to sandbox
    enabled: true
    steps:
      - name: DeployToFews
        options:
          apps: wdesk
          cluster_name: sandbox
          delay_days: 0
          deployable_days: []
          deployable_time_in_cst: ''
          max_attempts: 20
          wait_type: Immediately
        on_fail:
          - name: SendNotification
            options:
              color: red
              message: "Sandbox - Deployment Failed - commencing rollback"
              slack_channels:
                - alert-graph-app
          - name: Stop
            options:
              automatically_continue_after: 600
          - name: RollbackFewsDeploy
            options:
              apps: wdesk
              cluster_name: sandbox
              max_attempts: 20
          - name: StartSignalsRun
            options:
              max_attempts: 20
              plan_id: 0000000000
      - name: StartSignalsRun
        options:
          max_attempts: 20
          plan_id: 0000000000
        on_fail:
          - name: SendNotification
            options:
              color: red
              message: "Sandbox - Signals Tests Failed - commencing rollback"
              slack_channels:
                - alert-graph-app
          - name: Stop
            options:
              automatically_continue_after: 600
          - name: RollbackFewsDeploy
            options:
              apps: wdesk
              cluster_name: sandbox
              max_attempts: 20
          - name: StartSignalsRun
            options:
              max_attempts: 20
              plan_id: 0000000000

  - name: Deploy to Demo
    enabled: true
    steps:
      - name: DeployToFews
        options:
          apps: wdesk
          cluster_name: demo
          delay_days: 0
          deployable_days: []
          deployable_time_in_cst: ''
          max_attempts: 20
          wait_type: NextDayEightFiftyPmCentralMondayThroughThursday
        on_fail:
          - name: SendNotification
            options:
              color: red
              message: "Demo - Deployment Failed - commencing rollback"
              slack_channels:
                - alert-graph-app
          - name: Stop
            options:
              automatically_continue_after: 600
          - name: RollbackFewsDeploy
            options:
              apps: wdesk
              cluster_name: demo
              max_attempts: 20
          - name: StartSignalsRun
            options:
              max_attempts: 20
              plan_id: 0000000000
      - name: StartSignalsRun
        options:
          max_attempts: 20
          plan_id: 0000000000
        on_fail:
          - name: SendNotification
            options:
              color: red
              message: "Demo - Signals Tests Failed - commencing rollback"
              slack_channels:
                - alert-graph-app
          - name: Stop
            options:
              automatically_continue_after: 600
          - name: RollbackFewsDeploy
            options:
              apps: wdesk
              cluster_name: demo
              max_attempts: 20
          - name: StartSignalsRun
            options:
              max_attempts: 20
              plan_id: 0000000000

  - name: Deploy to APAC
    enabled: true
    steps:
      - name: DeployToFews
        options:
          apps: wdesk
          cluster_name: prod-apac
          delay_days: 0
          deployable_days: []
          deployable_time_in_cst: ''
          max_attempts: 20
          wait_type: StandardEU
        on_fail:
          - name: SendNotification
            options:
              color: red
              message: "APAC - Deployment Failed - commencing rollback"
              names_to_mention:
                - '@grc-flowmasters-team'
              slack_channels:
                - alert-graph-app
          - name: Stop
            options:
              automatically_continue_after: 600
          - name: RollbackFewsDeploy
            options:
              apps: wdesk
              cluster_name: prod-apac
              max_attempts: 20
          - name: StartSignalsRun
            options:
              max_attempts: 20
              plan_id: 0000000000
      - name: StartSignalsRun
        options:
          max_attempts: 20
          plan_id: 0000000000
        on_fail:
          - name: SendNotification
            options:
              color: red
              message: "APAC - Signals Tests Failed - commencing rollback"
              names_to_mention:
                - '@grc-flowmasters-team'
              slack_channels:
                - alert-graph-app
          - name: Stop
            options:
              automatically_continue_after: 600
          - name: RollbackFewsDeploy
            options:
              apps: wdesk
              cluster_name: prod-apac
              max_attempts: 20
          - name: StartSignalsRun
            options:
              max_attempts: 20
              plan_id: 0000000000

  - name: Deploy to EU
    enabled: true
    steps:
      - name: DeployToFews
        options:
          apps: wdesk
          cluster_name: prod-eu
          delay_days: 0
          deployable_days: []
          deployable_time_in_cst: ''
          max_attempts: 20
          wait_type: StandardEU
        on_fail:
          - name: SendNotification
            options:
              color: red
              message: "EU - Deployment Failed - commencing rollback"
              names_to_mention:
                - '@grc-flowmasters-team'
              slack_channels:
                - alert-graph-app
          - name: Stop
            options:
              automatically_continue_after: 600
          - name: RollbackFewsDeploy
            options:
              apps: wdesk
              cluster_name: prod-eu
              max_attempts: 20
          - name: StartSignalsRun
            options:
              max_attempts: 20
              plan_id: 0000000000
      - name: StartSignalsRun
        options:
          max_attempts: 20
          plan_id: 0000000000
        on_fail:
          - name: SendNotification
            options:
              color: red
              message: "EU - Signals Tests Failed - commencing rollback"
              names_to_mention:
                - '@grc-flowmasters-team'
              slack_channels:
                - alert-graph-app
          - name: Stop
            options:
              automatically_continue_after: 600
          - name: RollbackFewsDeploy
            options:
              apps: wdesk
              cluster_name: prod-eu
              max_attempts: 20
          - name: StartSignalsRun
            options:
              max_attempts: 20
              plan_id: 0000000000

  - name: Deploy to prod
    enabled: true
    steps:
      - name: DeployToFews
        options:
          apps: wdesk
          cluster_name: prod
          delay_days: 0
          deployable_days: []
          deployable_time_in_cst: ''
          max_attempts: 20
          wait_type: NextDayEightFiftyPmCentralMondayThroughThursday
        on_fail:
          - name: SendNotification
            options:
              color: red
              message: "Prod - Deployment Failed - commencing rollback"
              names_to_mention:
                - '@grc-flowmasters-team'
              slack_channels:
                - alert-graph-app
          - name: Stop
            options:
              automatically_continue_after: 600
          - name: RollbackFewsDeploy
            options:
              apps: wdesk
              cluster_name: prod
              max_attempts: 20
          - name: StartSignalsRun
            options:
              max_attempts: 20
              plan_id: 0000000000
      - name: StartSignalsRun
        options:
          max_attempts: 20
          plan_id: 0000000000
        on_fail:
          - name: SendNotification
            options:
              color: red
              message: "Prod - Signals Tests Failed - commencing rollback"
              names_to_mention:
                - '@grc-flowmasters-team'
              slack_channels:
                - alert-graph-app
          - name: Stop
            options:
              automatically_continue_after: 600
          - name: RollbackFewsDeploy
            options:
              apps: wdesk
              cluster_name: prod
              max_attempts: 20
          - name: StartSignalsRun
            options:
              max_attempts: 20
              plan_id: 0000000000

  - name: Close Tickets
    enabled: true
    steps:
      - name: TransitionTicketsToClosed
```

### C.5 Versioning and Channel Strategy

**Current:** `10.4.40` (semver, Rosie-bumped on each merge)

**Recommendation:** Keep the existing semver strategy. The version is already
well-managed. What changes is how consumers reference it:

| Consumer | Current | Target |
|---|---|---|
| wdesk (compile-time) | `w_sox: ^10.4.29` in pubspec.yaml | Remove dependency; consume via FEWS MFE |
| wdesk (CI) | `gha-integration-testing` with `release:latest` | Keep, but test against deployed MFE |
| CDN | `cdn-prod.wdesk.com/graph_app/<tag>` | No change |
| FEWS | `deploy-to-fews` with manifest | No change (manifest content changes) |

### C.6 Rollback Playbook

```
ROLLBACK PROCEDURE: graph_app Independent Release
==================================================

1. DETECT
   - Signals tests fail post-deploy in any environment
   - pipeline_template.yaml on_fail handler triggers automatically
   - Slack alert sent to #alert-graph-app

2. AUTOMATIC ROLLBACK (pipeline handles this)
   - Pipeline pauses for 600s (operator override window)
   - RollbackFewsDeploy executes, reverting to previous FEWS version
   - Post-rollback Signals run verifies the rollback succeeded

3. MANUAL ROLLBACK (if automatic fails)
   a. Identify the last-known-good BUILD_ID from FEWS deploy history
   b. Deploy manually:
      FEWS: Override via URL pattern:
      https://<cluster>.wdesk.org/fews/v1/serve/wdesk+cdn-dev:graph_app@<GOOD_BUILD_ID>
   c. Verify via Signals or manual smoke test
   d. Investigate root cause before re-deploying

4. CDN ROLLBACK
   - CDN assets are versioned by tag; previous tags remain available
   - If CDN asset is the issue, pin consumers to previous tag

5. PUB PACKAGE ROLLBACK
   - If a broken pub package was published, publish a patch version
     with the fix (pub.workiva.org does not support unpublishing)
   - Downstream consumers (if any remain) update their version pin

6. WDESK IMPACT
   - Post-decoupling: wdesk does NOT need to rollback for graph_app issues
   - FEWS dynamically serves the correct graph_app version
   - wdesk only needs action if the MFE manifest contract changes
```

---

## Migration Checklist (Sequenced by Dependency)

### Phase 0: Preparation (No Code Changes)

- [ ] **Create Slack channel** `#alert-graph-app` for pipeline notifications
- [ ] **Create Signals test plans** for each environment (staging, pentest,
      sandbox, demo, APAC, EU, prod) -- reuse existing functional test suite
      as the basis
- [ ] **Register pipeline in rmconsole** -- coordinate with release-eng team
      on who owns this (graph_app team or release-eng)
- [ ] **Confirm FEWS MFE contribution capabilities** -- verify that the FEWS
      `manifest.yaml` format supports all contribution types graph_app needs
      (drawer, rich, embeddable, routing, sidebar, landing page widgets)
- [ ] **Document the experience config -> manifest contribution mapping** --
      map each of the 27 w_sox experience configs to its manifest declaration

### Phase 1: graph_app MFE Manifest (graph_app repo)

- [ ] **Migrate `app/web/manifest.yaml`** from legacy app format to full MFE
      contribution format (see C.2 skeleton)
- [ ] **Create MFE entry point** (`app/web/w_sox_app.mfe.dart`) using
      `createMfe()` pattern instead of `createApp()`
- [ ] **Create `WsoxExtension`** class extending `WdeskExtension`, registering
      all contributions via `onRegisterContributions()`
- [ ] **Update `app/build.yaml`** to enable `wdesk_sdk_builders|mfe` builder
      for the new entry point
- [ ] **Test locally** that the MFE loads correctly in wdesk via FEWS override
      URL: `https://wk-dev.wdesk.org/fews/v1/serve/wdesk+cdn-dev:graph_app@<BUILD_ID>`
- [ ] **Verify all 27 experiences render** through the manifest path

### Phase 2: Dual-Path Validation (Both Repos)

- [ ] **Deploy graph_app as MFE to wk-dev** using the new manifest
- [ ] **Verify wdesk still works with compile-time w_sox** (no regression)
- [ ] **Verify wdesk works with manifest-contributed graph_app** (new path)
- [ ] **Run full Skynet/functional test suite** against both paths
- [ ] **Confirm feature parity** -- every experience accessible via both
      the old compile-time path AND the new MFE manifest path

### Phase 3: pipeline_template.yaml (graph_app repo)

- [ ] **Add `pipeline_template.yaml`** to graph_app repo (see C.4)
- [ ] **Fill in Signals plan IDs** from Phase 0
- [ ] **Register pipeline in rmconsole**
- [ ] **Perform a dry-run promotion** through wk-dev -> staging -> pentest
- [ ] **Validate rollback works** by intentionally deploying a bad build and
      confirming automatic rollback triggers

### Phase 4: wdesk Decoupling (wdesk repo)

- [ ] **Remove `w_sox` from wdesk/pubspec.yaml** dependency
- [ ] **Remove all `w_sox.*` references from experience_registry.dart`**
      (all 27 configs, plus landing page widgets)
- [ ] **Remove `import 'package:w_sox/...'` statements** from
      experience_registry.dart and any other files
- [ ] **Update `gha-integration-testing` workflow** to test against deployed
      MFE rather than compile-time dependency
- [ ] **Keep `isGraphTestEnvironment` in utils.dart** -- still needed for
      functional testing
- [ ] **Run full wdesk CI** -- confirm build succeeds without w_sox
- [ ] **Run full Skynet suite** -- confirm all graph features load via MFE

### Phase 5: Production Cutover

- [ ] **Promote graph_app through its own pipeline** to production
- [ ] **Merge wdesk decoupling PR** and promote through wdesk's pipeline
- [ ] **Monitor** for 1 week: Signals test results, error rates, user reports
- [ ] **Remove any feature flags** used during the transition
- [ ] **Update documentation** and runbooks

### Phase 6: Cleanup

- [ ] **Remove legacy `createApp()` entry point** if MFE path is sole path
- [ ] **Remove the Docker image build** if FEWS/CDN is the sole serving
      mechanism (assess if Docker is still needed for functional tests)
- [ ] **Archive any wdesk-side code** that was solely for graph_app bundling
- [ ] **Celebrate** -- graph_app is independently releasable

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| MFE manifest doesn't support all contribution types (embeddable, landing page widgets) | Medium | [BREAKING] | Validate in Phase 0 before any code changes |
| Dual-path period introduces subtle behavior differences | Medium | [HIGH] | Extensive Signals testing in Phase 2 |
| graph_app deploys ahead of wdesk, breaking contract | Low | [HIGH] | Semver enforcement; keep consumer integration test |
| Rollback in one environment, graph_app version drift across environments | Medium | [MEDIUM] | Pipeline stages are sequential with gates |
| 70+ dependency tree causes version conflicts when graph_app moves independently | Medium | [MEDIUM] | syncdeps workflow + semver audit already in place |
| Team capacity -- migration spans multiple sprints | High | [MEDIUM] | Phase 0-1 can be done independently; Phase 4 is the big-bang |

---

## Open Questions (Need Team Input)

1. **Who registers the rmconsole pipeline?** graph_app team or release-eng?
2. **Does FEWS manifest support embeddable experiences?** The form_config
   manifest only shows drawer + rich + routing + sidebar. graph_app also
   needs embeddable (createSampleSelectionExperienceConfig, etc.) and
   landing page widgets.
3. **Should the Docker image build be preserved?** It's used for functional
   testing (Docker-based stack), but post-MFE migration the serving path
   is FEWS/CDN, not Docker/nginx.
4. **Deployment window alignment:** form_config deploys demo/prod on
   Mon-Thu at 8:50 PM CST. Should graph_app use the same windows or
   different ones to avoid collision?
5. **`VerifySmithySucceeded` step:** graph_app uses GHA, not Smithy. Is
   there a `VerifyGHASucceeded` equivalent for pipeline_template.yaml?

---

## Part D: wdesk Reference Removal and Post-Decoupling Routing

### D.1 Complete File Inventory: Every w_sox Reference in wdesk

**9 files must be modified. 0 can be skipped.**

#### File 1: `wdesk/pubspec.yaml` (Dependency Declaration)

**Line 118-122 -- REMOVE:**
```yaml
w_sox:
  hosted:
    name: w_sox
    url: https://pub.workiva.org
  version: ^10.4.29
```

Also remove any transitive graph_* packages that are ONLY pulled in via
w_sox and not used directly by wdesk. Candidates:
- `graph_app_js` (v1.0.37 -- only used by w_sox)

**Risk:** [BREAKING] -- removing w_sox breaks every file that imports it.
All other file changes must land in the same PR.

---

#### File 2: `wdesk/lib/src/experience_registry.dart` (Primary Registry)

**Lines 20-21 -- REMOVE imports:**
```dart
import 'package:w_sox/experience_configs.dart' as w_sox;
import 'package:w_sox/landing_page_widgets.dart' as ir_widgets;
```

**Lines 62-86 -- REMOVE drawer experiences (9 references):**
```dart
// REMOVE all of these:
w_sox.ResourceManagementExperienceConfig(),    // line 62
w_sox.OverviewExperienceConfig(),              // line 63
w_sox.ReportListExperienceConfig(),            // line 75
w_sox.DashboardListExperienceConfig(),         // line 76
w_sox.DataDrawerExperienceConfig(),            // line 77
w_sox.DataModelExperienceConfig(),             // line 80
w_sox.SupportExperienceConfig(),               // line 81
w_sox.TextualQueryExperienceConfig(),          // line 85
w_sox.ProjectFilesExperienceConfig(),          // line 86
```

**Lines 118-136 -- REMOVE rich experiences (16 references):**
```dart
// REMOVE all of these:
w_sox.ExportListExperienceConfig(),            // line 118
w_sox.FocusExperienceConfig(),                 // line 119
w_sox.FocusNewExperienceConfig(),              // line 120
w_sox.BulkTestFormImportExperienceConfig(),    // line 121
w_sox.SamplingRichExperienceConfig.v2(),       // line 125
w_sox.ReportExperienceConfig(),                // line 126
w_sox.DashboardRichExperienceConfig(),         // line 127
w_sox.TestFormRichExperienceConfig(),          // line 128
w_sox.TextualQueryEditExperienceConfig(),      // line 129
w_sox.EvidenceTestingExperienceConfig(),       // line 130
w_sox.GraphAttachmentViewerExperienceConfig(), // line 131
w_sox.GraphMarkupViewerExperienceConfig(),     // line 132
w_sox.ReportBuilderExperienceConfig(),         // line 133
w_sox.ResourcePlanExperienceConfig(),          // line 134
w_sox.RawGraphExperienceConfig(),              // line 135
w_sox.SuggestedPermissionsExperienceConfig(),  // line 136
```

**Lines 169-170 -- REMOVE embeddable experiences (2 references):**
```dart
// REMOVE:
w_sox.createSampleSelectionExperienceConfig(), // line 169
w_sox.createTestFormSpreadsheetExperienceConfig(), // line 170
```

**Lines 194-205 -- REMOVE landing page widgets (10 references):**
```dart
// REMOVE all ir_widgets.* references:
'ir_charts': ir_widgets.ChartWidgetConfig(),
'ir_primary_data_types': ir_widgets.DataTypesWidgetConfig(),
'ir_recent_test_forms': ir_widgets.TestFormRecencyWidgetConfig(),
'ir_recent_audit_forms': ir_widgets.AuditFormRecencyWidgetConfig(),
'ir_recent_procedure_forms': ir_widgets.ProcedureFormRecencyWidgetConfig(),
'ir_recent_issue_forms': ir_widgets.IssueFormRecencyWidgetConfig(),
'ir_recent_action_plan_forms': ir_widgets.ActionPlanFormRecencyWidgetConfig(),
'ir_recent_reports': ir_widgets.ReportRecencyWidgetConfig(),
'ir_assigned_requests': ir_widgets.AssignedRequestsWidgetConfig(),
'ir_key_resources': ir_widgets.KeyResourcesWidgetConfig(),
```

**WARNING: Landing page widget keys are stored in the view-settings
database.** The test at `test/unit/experience_registry_test.dart` line 53
explicitly warns: "these values are committed to the view-settings database
and changing them would result in broken landing page dashboards." Removing
these keys means existing user dashboards that use these widgets will
display errors. **This requires a data migration or graceful fallback.**

---

#### File 3: `wdesk/lib/src/ir_archive_mode_experience_registry.dart`

**Line 6 -- REMOVE import:**
```dart
import 'package:w_sox/experience_configs.dart';
```

**Lines 14-21 -- REMOVE drawer experiences (7 references):**
```dart
ProjectFilesExperienceConfig(),
OverviewExperienceConfig(),
ReportListExperienceConfig(),
DashboardListExperienceConfig(),
DataDrawerExperienceConfig(),
TextualQueryExperienceConfig(),
DataModelExperienceConfig(),
```

**Lines 27-28 -- REMOVE embeddable experiences (2 references):**
```dart
createSampleSelectionExperienceConfig(),
createTestFormSpreadsheetExperienceConfig(),
```

**Lines 32-40 -- REMOVE rich experiences (6 references):**
```dart
DashboardRichExperienceConfig(),
EvidenceTestingExperienceConfig(),
FocusExperienceConfig(),
ReportExperienceConfig(),
TestFormRichExperienceConfig(),
GraphMarkupViewerExperienceConfig(),
```

**Post-removal state:** This class becomes nearly empty (only `audit`
and non-w_sox experiences remain). **Consider whether archive mode needs
its own MFE manifest variant** or whether the main graph_app MFE manifest
can handle archive mode via query parameters (as it does today in
`app/web/main.dart`).

---

#### File 4: `wdesk/lib/src/ir_draft_session_experience_registry.dart`

**Line 5 -- REMOVE import:**
```dart
import 'package:w_sox/experience_configs.dart';
```

**Lines 20-28 -- REMOVE drawer experiences (8 references):**
```dart
OverviewExperienceConfig(),
// (RequestListExperienceConfig is from request_portal, NOT w_sox -- KEEP)
ReportListExperienceConfig(),
DashboardListExperienceConfig(),
DataDrawerExperienceConfig(isDraftSession: true),
TextualQueryExperienceConfig(),
DataModelExperienceConfig(),
SupportExperienceConfig(),
```

**Lines 32-42 -- REMOVE rich experiences (8 references):**
```dart
FocusExperienceConfig(),
FocusNewExperienceConfig(),
RawGraphExperienceConfig(),
ReportExperienceConfig(),
DashboardRichExperienceConfig(),
TestFormRichExperienceConfig(),
ReportBuilderExperienceConfig(),
TextualQueryEditExperienceConfig(),
```

**CRITICAL: `DataDrawerExperienceConfig(isDraftSession: true)`** -- this
passes a constructor argument. The MFE manifest must support passing
parameters to experiences for draft session mode, OR graph_app's MFE
must detect draft session mode from the URL/environment and configure
itself accordingly.

---

#### File 5: `wdesk/web/main.dart` (Web Entry Point)

**Lines 10-12 -- REMOVE deferred imports:**
```dart
import 'package:w_sox/draft_edits_header.dart' deferred as draft_edits_header;
import 'package:w_sox/graph_archive_header.dart'
    deferred as graph_archive_header;
```

**Lines 15-18 -- REMOVE deferred registry imports:**
```dart
import 'package:wdesk/src/ir_archive_mode_experience_registry.dart'
    deferred as ir_archive_registry;
import 'package:wdesk/src/ir_draft_session_experience_registry.dart'
    deferred as ir_draft_registry;
```

**Lines 58-63 -- REMOVE deferred library loading:**
```dart
if (isIrArchiveMode) {
  await graph_archive_header.loadLibrary();
}
if (isIrDraftSessionMode) {
  await draft_edits_header.loadLibrary();
}
```

**Lines 83-91 -- REMOVE sidebar header setup:**
```dart
if (isIrArchiveMode) {
  app.setPrimarySidebarHeader(graph_archive_header.createGraphArchiveHeader);
}
if (isIrDraftSessionMode) {
  final header =
      await draft_edits_header.createDraftEditsHeaderV2(app.appContext);
  app.setPrimarySidebarHeader(() => header);
}
```

**Lines 121-129 -- REMOVE/SIMPLIFY getExperienceRegistry():**
```dart
// CURRENT:
Future<ExperienceRegistry> getExperienceRegistry() async {
  if (isIrArchiveMode) {
    await ir_archive_registry.loadLibrary();
    return ir_archive_registry.IrArchiveModeExperienceRegistry();
  } else if (isIrDraftSessionMode) {
    await ir_draft_registry.loadLibrary();
    return ir_draft_registry.IRDraftSessionExperienceRegistry();
  }
  return WdeskExperienceRegistry();
}

// TARGET:
Future<ExperienceRegistry> getExperienceRegistry() async {
  return WdeskExperienceRegistry();
  // Archive/draft session modes now handled by graph_app MFE
  // detecting the mode from URL query parameters and adjusting
  // its experience set accordingly.
}
```

**KEEP lines 40-55:** The `isGraphTestEnvironment` mock session setup
must remain -- it's needed for graph_app functional testing against
wdesk.

**CRITICAL COMPLEXITY:** Archive mode and draft session mode currently
use **different experience registries** that expose a **reduced subset**
of graph experiences. Post-decoupling, graph_app's MFE must handle this
internally -- detecting the mode from query parameters (`archiveMode`,
`draftSession`) and filtering its own experience contributions
accordingly.

---

#### File 6: `wdesk/web/headless.dart` (Headless Entry Point)

**No w_sox imports to remove.** This file only references
`isGraphTestEnvironment` from `utils.dart` and `WdeskExperienceRegistry`
(which will be cleaned up in File 2).

**KEEP:** The `isGraphTestEnvironment` block (lines 31-46) must remain.

---

#### File 7: `wdesk/lib/src/utils.dart` (Utility)

**KEEP as-is:**
```dart
bool get isGraphTestEnvironment =>
    Environment.current.testEnvironment == 'graph_app';
```

This flag is test infrastructure, not a production dependency. It
enables wdesk to set up mock credentials when running graph_app's
functional test suite. Removing it would break graph_app's CI.

---

#### File 8: `wdesk/test/unit/experience_registry_test.dart`

**Line 5 -- REMOVE import:**
```dart
import 'package:w_sox/landing_page_widgets.dart' as ir_widgets;
```

**Lines 64-75 -- REMOVE all ir_widgets assertions:**
```dart
'ir_charts': ir_widgets.ChartWidgetConfig(),
'ir_primary_data_types': ir_widgets.DataTypesWidgetConfig(),
'ir_recent_test_forms': ir_widgets.TestFormRecencyWidgetConfig(),
'ir_recent_audit_forms': ir_widgets.AuditFormRecencyWidgetConfig(),
'ir_recent_procedure_forms': ir_widgets.ProcedureFormRecencyWidgetConfig(),
'ir_recent_issue_forms': ir_widgets.IssueFormRecencyWidgetConfig(),
'ir_recent_action_plan_forms': ir_widgets.ActionPlanFormRecencyWidgetConfig(),
'ir_recent_reports': ir_widgets.ReportRecencyWidgetConfig(),
'ir_assigned_requests': ir_widgets.AssignedRequestsWidgetConfig(),
'ir_key_resources': ir_widgets.KeyResourcesWidgetConfig(),
```

**Update the test assertion count** to reflect the reduced number of
widget configs after removing the 10 IR widgets.

---

#### File 9: `wdesk/.github/workflows/frontend-integration.yaml`

**Lines 14-20 -- MODIFY (do not remove):**
```yaml
# CURRENT:
integration-testing:
  name: graph_app
  steps:
    - uses: Workiva/gha-integration-testing@v4.0.37
      with:
        consumer-repository: Workiva/graph_app
        use-commit-from: release:latest

# TARGET: Keep the integration test but update it to validate
# that wdesk works correctly with the deployed graph_app MFE.
# The exact configuration depends on whether gha-integration-testing
# supports MFE-based consumer testing. If not, replace with a
# Signals or smoke test that loads wdesk and verifies graph
# experiences are available via the MFE path.
```

---

### D.2 Summary: Reference Removal by Category

| Category | Files | References Removed | Risk |
|---|---|---|---|
| **Imports** | 5 files | 7 import statements | [BREAKING] build fails if any missed |
| **Drawer experiences** | 3 files | 24 instantiations | [BREAKING] nav items vanish |
| **Rich experiences** | 3 files | 30 instantiations | [BREAKING] full-page views gone |
| **Embeddable experiences** | 2 files | 4 factory calls | [BREAKING] embedded views gone |
| **Landing page widgets** | 2 files | 20 widget configs | [HIGH] user dashboards break |
| **Deferred libraries** | 1 file | 4 deferred imports + loads | [HIGH] archive/draft modes |
| **Sidebar headers** | 1 file | 2 header setups | [MEDIUM] archive/draft mode headers |
| **Registry selection** | 1 file | 1 function (getExperienceRegistry) | [HIGH] mode-specific registries |
| **CI workflow** | 1 file | 1 integration test config | [MEDIUM] CI gate |
| **Test assertions** | 1 file | 10 widget expectations | [LOW] test-only |
| **Total** | **9 files** | **~103 references** | |

---

### D.3 How Routing Works Post-Decoupling

This is the most important architectural question. Here's how it works:

#### Current State: Compile-Time Routing

```
User navigates to /reports
    |
    v
wdesk Router (w_router)
    |
    v
ExperienceRegistry[routeSegment]
    |  (lookup by route segment string)
    v
WdeskExperienceRegistry.drawerExperiences
    |  (hardcoded list includes w_sox.ReportListExperienceConfig())
    v
ReportListExperienceConfig
    |  routeSegment => GraphRoutePaths.reports => 'reports'
    |  experienceFactory => deferred load ReportListExperience
    v
ReportListExperience renders
```

**Problem:** The config class is compiled into wdesk. Changing the route
or the experience requires a wdesk release.

#### Target State: MFE Manifest Routing

```
User navigates to /reports
    |
    v
wdesk Router (w_router)
    |
    v
RichExperienceRegistryShim (already exists in wdesk_sdk)
    |
    +---> Legacy lookup: WdeskExperienceRegistry[routeSegment]
    |       (returns null -- w_sox configs removed)
    |
    +---> MFE lookup: RoutingContributionPoint[routeSegment]
    |       (returns match from FEWS manifest)
    |
    v
FEWS manifest declares:
    core.routing:
      - name: reports_route
        details:
          route_segment: reports
          experience_name: w_sox.core.drawer_experiences.report_list
    |
    v
MFE Extension (WsoxExtension) registered the contribution:
    DrawerExperienceContributionHost(ReportListContribution())
    |
    v
graph_app's JavaScript bundle loaded from CDN/FEWS
    |
    v
ReportListExperience renders (same code, different loading path)
```

**Key insight:** `RichExperienceRegistryShim` in `wdesk_sdk` already
merges compile-time and MFE experiences into a single routing table.
When wdesk removes the compile-time `w_sox` configs, the shim simply
resolves graph routes from the FEWS manifest instead. **No router
changes needed in wdesk.**

#### Route Segment Mapping (Complete)

Every experience config in graph_app declares a `routeSegment`. These
exact same strings must appear in the MFE manifest's `core.routing`
section. Here is the complete mapping:

**Drawer Experiences (left navigation):**

| Experience Config | Route Segment | Nav Icon | Display Name |
|---|---|---|---|
| ResourceManagementExperienceConfig | `GraphRoutePaths.resourceManagementList` | `unify.calendarMonth` | Planning |
| OverviewExperienceConfig | `testing` | `unify.science` | Testing |
| ReportListExperienceConfig | `reports` | `unify.summarize` | Reports |
| DashboardListExperienceConfig | `dashboards` | `unify.dashboard` | Dashboards |
| DataDrawerExperienceConfig | `GraphRoutePaths.focusList` | `unify.database` | Data Types |
| DataModelExperienceConfig | `GraphRoutePaths.dataModel` | `unify.accountTree` | Model |
| SupportExperienceConfig | `GraphRoutePaths.support` | `unify.settings` | Support |
| TextualQueryExperienceConfig | `GraphRoutePaths.textualQuery` | `unify.editSquare` | Textual Query |
| ProjectFilesExperienceConfig | `GraphRoutePaths.projectFiles` | `unify.folder` | Project Files |

**Rich Experiences (full-page views):**

| Experience Config | Route Segment |
|---|---|
| FocusExperienceConfig | `GraphRoutePaths.focus` |
| FocusNewExperienceConfig | `GraphRoutePaths.focusNew` |
| ExportListExperienceConfig | `export_list` |
| EvidenceTestingExperienceConfig | `evidence_testing` |
| ReportExperienceConfig | `GraphRoutePaths.report` |
| ReportBuilderExperienceConfig | `GraphRoutePaths.reportBuilder` |
| DashboardRichExperienceConfig | `GraphRoutePaths.dashboard` |
| TestFormRichExperienceConfig | `test_form` |
| SamplingRichExperienceConfig | `select_samples` |
| TextualQueryEditExperienceConfig | `GraphRoutePaths.textualQueryEdit` |
| BulkTestFormImportExperienceConfig | `bulk_test_form_import` |
| GraphAttachmentViewerExperienceConfig | `SoxRoutePaths.graphAttachmentViewer` |
| GraphMarkupViewerExperienceConfig | `SoxRoutePaths.graphMarkupViewer` |
| RawGraphExperienceConfig | `GraphRoutePaths.rawGraph` |
| SuggestedPermissionsExperienceConfig | `suggested-permissions` |
| ResourcePlanExperienceConfig | `GraphRoutePaths.plan` |

**Note:** The actual string values for `GraphRoutePaths.*` constants live
in `graph_ui/lib/src/utils/graph_route_paths.dart`. These must be
resolved to literal strings when writing the manifest. Failing to match
exactly will result in broken deep links.

---

### D.4 Archive Mode and Draft Session Mode

This is the **hardest decoupling challenge**. Today, wdesk selects a
different `ExperienceRegistry` based on the URL mode:

```dart
// wdesk/web/main.dart:121-130
Future<ExperienceRegistry> getExperienceRegistry() async {
  if (isIrArchiveMode) {
    return IrArchiveModeExperienceRegistry();  // reduced set of w_sox configs
  } else if (isIrDraftSessionMode) {
    return IRDraftSessionExperienceRegistry(); // different reduced set
  }
  return WdeskExperienceRegistry();            // full set
}
```

**What's different in each mode:**

| Mode | Drawer Experiences | Rich Experiences | Special |
|---|---|---|---|
| **Normal** | All 9 + non-w_sox | All 16 + non-w_sox | Full feature set |
| **Archive** | 7 (no ResourceMgmt, Support) | 6 (no Focus/New, Sampling, etc.) | Graph archive header sidebar |
| **Draft Session** | 8 (includes Support, no ResourceMgmt) | 8 (includes Focus/New, no Sampling/Export) | Draft edits header sidebar |

**Post-decoupling options:**

**Option A: graph_app MFE handles mode internally (RECOMMENDED)**
- graph_app's MFE entry point reads the mode from URL query parameters
  (`?archiveMode=true`, `?draftSession=true`)
- The `WsoxExtension.onRegisterContributions()` method conditionally
  registers only the appropriate experiences for the detected mode
- The sidebar header components (`graph_archive_header`,
  `draft_edits_header`) become part of graph_app's MFE bundle
- **Pro:** Clean separation. wdesk doesn't need to know about graph
  modes.
- **Con:** graph_app's MFE must be loaded before wdesk can determine
  which sidebar header to show.

**Option B: Multiple MFE manifests per mode**
- Register separate MFE names: `w_sox`, `w_sox_archive`, `w_sox_draft`
- Each has its own manifest with the appropriate subset of contributions
- wdesk selects which MFE to load based on mode
- **Pro:** Declarative. Each manifest is a complete specification.
- **Con:** Three manifests to maintain. Version drift risk between them.

**Option C: FEWS manifest supports conditional contributions**
- If FEWS supports `can_user_access` expressions with mode conditions,
  a single manifest could gate contributions by mode
- **Pro:** Single manifest, declarative filtering
- **Con:** Depends on FEWS capability that may not exist. Verify in
  Phase 0.

**Recommendation:** Option A. graph_app already detects these modes in
its `app/web/main.dart`. Moving mode detection into the MFE entry point
is a natural extension.

---

### D.5 Landing Page Widgets: The Data Migration Problem

The 10 `ir_widgets.*` landing page widget configs use **string keys**
stored in the view-settings database:

```
'ir_charts', 'ir_primary_data_types', 'ir_recent_test_forms',
'ir_recent_audit_forms', 'ir_recent_procedure_forms',
'ir_recent_issue_forms', 'ir_recent_action_plan_forms',
'ir_recent_reports', 'ir_assigned_requests', 'ir_key_resources'
```

The test at `experience_registry_test.dart:53` explicitly warns:

> "these values are committed to the view-settings database and changing
> them would result in broken landing page dashboards"

**Post-decoupling, these widgets must be contributed by graph_app's MFE
manifest.** The manifest must use **the same string keys** so that
existing user dashboard configurations continue to work.

**How form_config / assessments_client handle this:**
- They do NOT currently contribute landing page widgets via manifest
- This may be a contribution type that FEWS manifest doesn't support yet

**Action required:**
1. Verify FEWS manifest supports `core.landing_page_widgets` (or
   equivalent) contribution type
2. If supported: graph_app's manifest declares widgets with the same
   keys
3. If NOT supported: the widgets must remain in wdesk temporarily, with
   graph_app providing the widget implementation classes via a
   lightweight pub package (not the full w_sox package). Or: push for
   FEWS to support this contribution type before decoupling.

---

### D.6 Deferred Loading: What Moves to graph_app

Two w_sox modules are loaded via Dart deferred imports in wdesk:

1. **`graph_archive_header`** (`package:w_sox/graph_archive_header.dart`)
   - Creates a sidebar header component for archive mode
   - Called via `app.setPrimarySidebarHeader()`

2. **`draft_edits_header`** (`package:w_sox/draft_edits_header.dart`)
   - Creates a sidebar header component for draft session mode
   - Called via `app.setPrimarySidebarHeader()`

**Post-decoupling:** These headers must be contributed by graph_app's
MFE. Options:
- Register as MFE contributions under a `core.sidebar_header`
  contribution point (if FEWS supports it)
- Or: graph_app's MFE extension calls `app.setPrimarySidebarHeader()`
  directly during initialization when it detects archive/draft mode

---

### D.7 wdesk Decoupling PR: File-by-File Change Spec

Here is the exact PR diff specification for the wdesk decoupling:

```
wdesk/pubspec.yaml
  - Remove w_sox dependency (lines 118-122)
  - Run `dart pub get` to update lockfile

wdesk/lib/src/experience_registry.dart
  - Remove lines 20-21 (w_sox imports)
  - Remove lines 62-63 (ResourceManagement, Overview)
  - Remove lines 75-77 (ReportList, DashboardList, DataDrawer)
  - Remove lines 80-81 (DataModel, Support)
  - Remove lines 85-86 (TextualQuery, ProjectFiles)
  - Remove lines 118-136 (all rich experiences)
  - Remove lines 169-170 (embeddable experiences)
  - Remove lines 194-205 (landing page widgets)
  - Keep all non-w_sox experiences (audit, markup, viewer, etc.)
  - Keep all non-ir_widgets landing page widgets (comments, tasker, esg)

wdesk/lib/src/ir_archive_mode_experience_registry.dart
  - Remove line 6 (w_sox import)
  - Remove all w_sox config instantiations (lines 14-40)
  - Keep audit and non-w_sox experiences
  - If class becomes effectively empty, consider deleting the file
    and removing archive mode registry selection from main.dart

wdesk/lib/src/ir_draft_session_experience_registry.dart
  - Remove line 5 (w_sox import)
  - Remove all w_sox config instantiations (lines 20-42)
  - Keep audit, request_portal, and grc_testing_client experiences
  - If class becomes effectively empty, consider deleting the file

wdesk/web/main.dart
  - Remove lines 10-12 (deferred w_sox header imports)
  - Remove lines 15-18 (deferred archive/draft registry imports)
  - Remove lines 58-63 (deferred library loading)
  - Remove lines 83-91 (sidebar header setup)
  - Simplify getExperienceRegistry() to always return
    WdeskExperienceRegistry() (lines 121-130)
  - KEEP lines 40-55 (isGraphTestEnvironment mock session)

wdesk/web/headless.dart
  - No changes needed (no w_sox imports)
  - KEEP isGraphTestEnvironment block

wdesk/lib/src/utils.dart
  - No changes needed
  - KEEP isGraphTestEnvironment getter

wdesk/test/unit/experience_registry_test.dart
  - Remove line 5 (ir_widgets import)
  - Remove lines 64-75 (ir_widgets assertions)
  - Update expected widget count in test assertion (line 100)

wdesk/.github/workflows/frontend-integration.yaml
  - Modify to test against deployed MFE instead of compile-time dep
  - Or replace with a smoke test that verifies graph experiences load
```

---

### D.8 Routing Architecture Diagram: Before vs After

```
BEFORE (Compile-Time Bundled):
==============================

  wdesk build
    |
    +---> pubspec.yaml declares w_sox: ^10.4.29
    |       |
    |       +---> dart pub get pulls w_sox source
    |       |
    |       +---> dart2js compiles w_sox INTO wdesk bundle
    |
    +---> main.dart
    |       |
    |       +---> getExperienceRegistry()
    |       |       |
    |       |       +---> WdeskExperienceRegistry()
    |       |               |
    |       |               +---> w_sox.ReportListExperienceConfig()
    |       |               +---> w_sox.DashboardListExperienceConfig()
    |       |               +---> ... (25+ more)
    |       |
    |       +---> createApp(experienceRegistry: registry)
    |               |
    |               +---> Router builds lookup table from registry
    |               +---> /reports -> ReportListExperienceConfig
    |
    +---> User navigates to /reports
            |
            +---> Router matches "reports" in compiled registry
            +---> Experience renders from wdesk's JS bundle


AFTER (MFE Manifest-Driven):
=============================

  graph_app build (INDEPENDENT)          wdesk build (INDEPENDENT)
    |                                      |
    +---> compile to JS                    +---> pubspec.yaml: NO w_sox
    +---> publish to CDN                   +---> dart2js compiles WITHOUT w_sox
    +---> deploy to FEWS with manifest     +---> WdeskExperienceRegistry() is
    |                                      |     smaller (no graph experiences)
    |                                      |
    v                                      v
  FEWS stores manifest:                  wdesk main.dart:
    microfrontends:                        createApp(
      w_sox:                                 experienceRegistry: registry,
        extensions:                          manifestAppName: 'wdesk',
          w_sox:                           )
            contributions:                   |
              core.routing:                  +---> createApp() calls FEWS API
                - route_segment: reports     |      fews_client.getManifest('wdesk')
                - route_segment: dashboards  |
                ...                          +---> FEWS returns merged manifest
                                             |     (includes graph_app's contributions)
                                             |
                                             v
                                           RichExperienceRegistryShim
                                             |
                                             +---> Legacy: WdeskExperienceRegistry
                                             |     (audit, markup, viewer only)
                                             |
                                             +---> MFE: RoutingContributionPoint
                                             |     (graph experiences from manifest)
                                             |
                                             v
                                           Merged routing table:
                                             /reports -> MFE contribution
                                             /dashboards -> MFE contribution
                                             /audit -> Legacy compile-time
                                             |
                                             v
                                           User navigates to /reports
                                             |
                                             +---> Router matches in MFE contributions
                                             +---> Loads graph_app JS from CDN
                                             +---> Experience renders
```

---

### D.9 Post-Decoupling Deployment Independence

After decoupling, the two repos deploy independently:

```
graph_app release cycle:         wdesk release cycle:
  merge to master                  merge to master
    |                                |
    v                                v
  GHA builds + tests               GHA builds + tests
    |                                |
    v                                v
  CDN publish + FEWS deploy        CDN publish + FEWS deploy
    |                                |
    v                                v
  pipeline_template.yaml:          wdesk's own pipeline:
    wk-dev -> staging -> ...         wk-dev -> staging -> ...
    |                                |
    v                                v
  graph_app in prod                wdesk in prod
  (serves via CDN/FEWS)           (loads graph_app MFE at runtime)
```

**Key behavioral changes:**
1. graph_app can ship a bug fix to production without waiting for a
   wdesk release
2. wdesk can ship non-graph changes without risk of graph_app regression
3. graph_app version in each environment may differ from wdesk version
4. Deep links to graph features work identically (same route segments)
5. Bookmarks and saved URLs continue to work (routes unchanged)
6. Browser back/forward navigation unchanged (client-side routing)

**What could go wrong:**
- **API contract drift:** If graph_app changes an experience's route
  segment, existing deep links break. Mitigated by: route segments are
  declared in graph_app (the MFE owns its routes), not in wdesk.
- **Static asset version mismatch:** If graph_app's CDN assets are
  updated but FEWS still serves old manifest. Mitigated by: FEWS and
  CDN are updated atomically in the same pipeline stage.
- **Cross-MFE communication:** If graph experiences communicate with
  wdesk or other MFEs via the event bus, version mismatches could cause
  protocol errors. Mitigated by: explicit API versioning on event bus
  messages.
