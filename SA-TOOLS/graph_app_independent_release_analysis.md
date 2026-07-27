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
