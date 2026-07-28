# graph_app MFE Migration: Manual Pre-Work Checklist

What needs to be done **by humans** before the actual MFE coding starts.
Based on auditing the graph_app codebase against the
[`dart-experience-to-mfe`](https://github.com/Workiva/fef-ai/pull/25)
skill requirements.

---

## 1. Slack and Team Setup

These are non-technical tasks that require coordination with other
teams.

| Task | Who | Notes |
|---|---|---|
| Create `#alert-graph-app` Slack channel | Team lead | For pipeline failure notifications |
| Decide who registers rmconsole pipeline | Team lead + release-eng | graph_app team or release-eng? |
| Set up Signals test plans (7 environments) | QA / team lead | staging, pentest, sandbox, demo, APAC, EU, prod |
| Create LaunchDarkly flags for rollout | Any team member | One flag per experience or per batch |
| Align on deployment windows | Team lead | Same as form_config (Mon-Thu 8:50 PM CST) or different? |

---

## 2. Verify FEWS Manifest Capabilities

These are questions only the **frontend architecture team** can answer.
Ask in `#support-frontend-architecture`.

### 2a. Embeddable experiences -- NOT covered by the skill

graph_app has **2 embeddable experiences** that are NOT drawer or rich
type:

- `createSampleSelectionExperienceConfig()`
- `createTestFormSpreadsheetExperienceConfig()`

The `dart-experience-to-mfe` skill only covers `core.drawer_experiences`
and `core.rich_experiences`. **Ask:** Does FEWS manifest support an
embeddable experience contribution type? If not, what is the migration
path?

### 2b. Landing page widgets -- NOT covered by the skill

graph_app contributes **10 landing page widget configs** to wdesk:

```
ir_charts, ir_primary_data_types, ir_recent_test_forms,
ir_recent_audit_forms, ir_recent_procedure_forms,
ir_recent_issue_forms, ir_recent_action_plan_forms,
ir_recent_reports, ir_assigned_requests, ir_key_resources
```

These keys are stored in the **view-settings database**. Changing them
breaks existing user dashboards. **Ask:** Does FEWS manifest support a
`core.landing_page_widgets` contribution type? If not, how should these
be handled post-decoupling?

### 2c. Archive mode and draft session mode

graph_app uses different experience subsets for archive mode and draft
session mode. **Ask:** Can the MFE detect the mode from URL query
parameters and conditionally register contributions? Or does FEWS
support conditional contributions via `can_user_access` expressions?

### 2d. Sidebar headers

graph_app provides custom sidebar headers for archive mode
(`graph_archive_header`) and draft session mode
(`draft_edits_header`). **Ask:** Is there a `core.sidebar_header`
contribution point? If not, can the MFE extension call
`app.setPrimarySidebarHeader()` directly?

### 2e. `VerifyGHASucceeded` pipeline step

graph_app uses GitHub Actions, not Smithy. **Ask:** Is there a
`VerifyGHASucceeded` step available for `pipeline_template.yaml`? The
form_config template uses `VerifySmithySucceeded`.

---

## 3. Code Fixes Required Before Migration

These are code changes that must be made in **separate PRs** before the
MFE scaffolding work begins.

### 3a. `configurationData` must be typed as `String`

**Status: FIX NEEDED**

`DashboardRichExperienceConfig` casts `configurationData` as a
`ddt.Dashboard` object (not a `String`):

```
lib/src/reports/dashboard/rich/dashboard_rich_experience.dart:78
    ddt.Dashboard? dashboardData =
        _factoryContext.configurationData as ddt.Dashboard?;
```

**What to fix:**
- Change to accept a JSON-encoded `String` and deserialize inside the
  factory.
- Update all callsites that pass a `Dashboard` object to serialize it
  to a JSON string first.

The embedded spreadsheet experiences also use typed
`EmbeddedSpreadsheetParams`:

```
lib/src/sox/sox_ui/src/module/test_phase_form/
    test_phase_form_store.dart:1377
lib/src/sox/sample_selection/src/flux/sample_selection/
    sample_selection_store.dart:768
```

However, these are for the embeddable experiences (which are a separate
migration concern -- see section 2a).

### 3b. `navigator.goTo` must become `navigator.goToExperience`

**Status: MOSTLY DONE, 1 callsite remaining**

Most callsites already use `goToExperience`. One remaining `goTo` call:

```
lib/src/data_experience/focus_list/components/
    data_list_toolbar.dart:261
    navigator.goTo('suggested-permissions', resourceId: 'suggestions')
```

**What to fix:**
- Change to `navigator.goToExperience('suggested-permissions',
  resourceId: 'suggestions')`.

The other `goTo` variants (`goToUrl`, `goToTab`, etc.) are fine -- only
`goTo` without a suffix needs migration.

### 3c. `titleContextMenuGroups` must be migrated

**Status: FIX NEEDED -- 2 experiences affected**

The legacy `titleContextMenuGroups` API does not work in MFEs. Found in:

```
lib/src/planning/rich/resource_plan_module.dart:175
    ContextMenuGroupsFactory get titleContextMenuGroups => () { ... }

lib/src/planning/rich/resource_plan_experience.dart:118
    titleContextMenuGroups: module.components.titleContextMenuGroups

lib/src/_shared/graph_experience_base.dart:258
    titleContextMenuGroups: module.components!.titleContextMenuGroups
```

**What to fix:**
- In `ResourcePlanExperience`: replace `titleContextMenuGroups` with
  `experienceContext.shell.setContextMenuGroups(...)`.
- In `graph_experience_base.dart`: same migration for the shared base
  class (this affects **all experiences** that use the shared base).
- In `rich_module.dart`: the base class declares
  `get titleContextMenuGroups => null` -- update the pattern.

**This is a cross-cutting change.** Because `graph_experience_base.dart`
is the shared base for many experiences, this fix touches most of them.
Test thoroughly.

### 3d. `appInitializer` overrides need adapter wrapping

**Status: AWARENESS NEEDED -- 24 experience configs affected**

Almost every experience config overrides `appInitializer`. Most point
to a shared `graph_experience_lib.appInitializer`. When migrating to
MFE, each contribution class needs `legacyAppInitializerAdapter` from
`wdesk_sdk` to wrap these.

**No code fix needed now**, but be aware during migration:
- Use `legacyAppInitializerAdapter` to wrap the existing initializer.
- Wrap it to ensure it only runs once (the adapter does not prevent
  multiple calls).
- Call it before `legacyDrawerExperienceFactoryAdapter` in each
  contribution.

The shared initializers:

| Initializer | Used by |
|---|---|
| `graph_experience_lib.appInitializer` | ~15 experiences (data, planning, reports, model, support, etc.) |
| `graph_experience_lib.appServiceInitializer` | 4 experiences (evidence testing, attachment/markup viewers, sample selection) |
| `experience_lib.testFormInitializer` | TestFormRichExperience |
| `experience_lib.testFormListExperienceAppInitializer` | TestFormListExperience (Overview) |
| `experience_lib.bulkImportAppInitializer` | BulkTestFormImport |
| `export_list_lib.exportListAppInitializer` | ExportList |
| `experience_lib.appInitializer` (textual query) | TextualQuery, TextualQueryEdit, ReportBuilder |

---

## 4. Hard Blockers to Identify

### 4a. Rich experiences that embed other experiences

**Status: 2 experiences embed -- INVESTIGATE**

Two experiences use `embeddableExperienceManagerV2` to embed
spreadsheets:

| Experience | File | What it embeds |
|---|---|---|
| **TestPhaseFormStore** (part of TestFormRichExperience) | `lib/src/sox/sox_ui/src/module/test_phase_form/test_phase_form_store.dart:1369` | Embedded spreadsheet via `createExperience<EmbeddedSpreadsheetParams>` |
| **SampleSelectionStore** (part of SamplingRichExperience) | `lib/src/sox/sample_selection/src/flux/sample_selection/sample_selection_store.dart:764` | Embedded spreadsheet via `createExperience<EmbeddedSpreadsheetParams>` |

Per the skill: **"If the rich experience embeds another experience, it
cannot be migrated to an MFE yet."**

**Decision needed:**
- Can `TestFormRichExperience` and `SamplingRichExperience` be migrated
  despite embedding? Has this limitation been relaxed?
- If not, these 2 experiences stay as compile-time dependencies in wdesk
  until embedding support is added.
- Ask `#support-frontend-architecture` for the latest status.

### 4b. Rich experiences with `KeyBindingManager`

**Status: 3 experiences use key bindings -- NOT a hard blocker but
needs acceptance**

| Experience | Key bindings | File |
|---|---|---|
| **FocusExperience** (Data) | CTRL+SHIFT+R (refresh) | `lib/src/data_experience/focus/focus_module.dart:190` |
| **DataListExperience** (Data Types) | CTRL+SHIFT+R (refresh) | `lib/src/data_experience/focus_list/data_list_module.dart:124` |
| **TestFormRichExperience** | PAGE_UP, PAGE_DOWN, HOME, END (scroll), plus custom bindings | `lib/src/sox/sox_ui/src/test_form/form_scroll_wrapper.dart:67-70` |

Per the skill: MFE key bindings are isolated -- they won't appear in
`KeybindingModal` and can collide with wdesk shortcuts.

**Decision needed:**
- Accept the isolation as a known limitation? CTRL+SHIFT+R is unlikely
  to collide. PAGE_UP/DOWN/HOME/END are standard scroll keys that
  may collide.
- Or: remove key bindings from the MFE version and rely on standard
  browser behavior.

---

## 5. Infrastructure Already In Place

Good news -- some things are already done:

| Item | Status |
|---|---|
| `microfrontend` package in pubspec.yaml | Already a dependency |
| `rich_experience_contribution` in pubspec.yaml | Already a dependency |
| Legacy `app/web/manifest.yaml` exists | Yes (`version: 1, app: {name: w_sox_app}`) -- needs replacement, not creation from scratch |
| GHA CI pipeline | Fully operational |
| CDN publishing | Working |
| FEWS deployment | Working |
| Semver + Rosie version bumping | Working |
| `goToExperience` migration | ~95% done (1 remaining `goTo` call) |

---

## 6. Summary: Ordered Action Items

Here is everything that needs to happen **before writing MFE code**,
in priority order:

### Must do (blockers)

1. **Ask `#support-frontend-architecture`** about embeddable experience
   support, landing page widget contribution type, archive/draft mode
   handling, sidebar headers, and `VerifyGHASucceeded` pipeline step
   (section 2).

2. **Fix `DashboardRichExperienceConfig` configurationData typing** --
   change from `ddt.Dashboard` object to JSON-encoded `String`
   (section 3a). **Separate PR.**

3. **Fix the 1 remaining `navigator.goTo` call** in
   `data_list_toolbar.dart:261` to `goToExperience` (section 3b).
   **Separate PR** (or combine with #2).

4. **Migrate `titleContextMenuGroups` to
   `experienceContext.shell.setContextMenuGroups`** in
   `graph_experience_base.dart` and `resource_plan_module.dart`
   (section 3c). **Separate PR.** Test all experiences that use the
   shared base.

5. **Get a decision on embedding blocker** -- ask frontend architecture
   if `TestFormRichExperience` and `SamplingRichExperience` can be
   migrated despite using `embeddableExperienceManagerV2`, or if they
   must wait (section 4a).

### Should do (reduces risk)

6. **Create `#alert-graph-app` Slack channel** (section 1).

7. **Create LaunchDarkly flags** for incremental rollout (section 1).

8. **Create Signals test plans** for each environment (section 1).

9. **Get a decision on key binding isolation** -- accept or rework for
   Focus, DataList, and TestForm experiences (section 4b).

### Nice to have (can happen in parallel)

10. **Align on deployment windows** with release-eng (section 1).

11. **Decide who registers rmconsole pipeline** (section 1).

12. **Document the full experience config -> manifest field mapping**
    for all 27 experiences (route segments, display names, icons, sort
    orders, oauth2 scopes, can_user_access expressions).

---

## Quick Reference: Experience Migration Readiness

| Experience | configurationData | titleContextMenuGroups | Embedding | Key bindings | appInitializer | Ready? |
|---|---|---|---|---|---|---|
| ResourceManagement | OK | OK | No | No | Yes (shared) | YES |
| Overview (TestFormList) | OK | OK | No | No | Yes (custom) | YES |
| ReportList | OK | OK | No | No | Yes (shared) | YES |
| DashboardList | OK | OK | No | No | Yes (shared) | YES |
| DataDrawer (DataTypes) | OK | OK | No | CTRL+SHIFT+R | Yes (shared) | YES (accept KB isolation) |
| DataModel | OK | OK | No | No | Yes (shared) | YES |
| Support | OK | OK | No | No | Yes (shared) | YES |
| TextualQuery | OK | OK | No | No | Yes (custom) | YES |
| ProjectFiles | OK | OK | No | No | Yes (shared) | YES |
| ExportList | OK | OK | No | No | Yes (custom) | YES |
| Focus | OK | Uses shared base | No | CTRL+SHIFT+R | Yes (shared) | FIX titleContextMenuGroups first |
| FocusNew | OK | Uses shared base | No | No | Yes (shared) | FIX titleContextMenuGroups first |
| BulkTestFormImport | OK | OK | No | No | Yes (custom) | YES |
| Sampling | OK | OK | **EMBEDS** | No | Yes (shared) | **BLOCKED** (embedding) |
| Report | OK | Uses shared base | No | No | Yes (shared) | FIX titleContextMenuGroups first |
| DashboardRich | **TYPED** | Uses shared base | No | No | Yes (shared) | FIX configurationData + titleContextMenuGroups |
| TestForm | OK | OK | **EMBEDS** | PAGE_UP/DOWN etc. | Yes (custom) | **BLOCKED** (embedding) |
| TextualQueryEdit | OK | Uses shared base | No | No | Yes (custom) | FIX titleContextMenuGroups first |
| EvidenceTesting | OK | OK | No | No | Yes (custom) | YES |
| GraphAttachmentViewer | OK | OK | No | No | Yes (custom) | YES |
| GraphMarkupViewer | OK | OK | No | No | Yes (custom) | YES |
| ReportBuilder | OK | Uses shared base | No | No | Yes (custom) | FIX titleContextMenuGroups first |
| ResourcePlan | OK | **DIRECT USE** | No | No | Yes (shared) | FIX titleContextMenuGroups first |
| RawGraph | OK | OK | No | No | Yes (custom) | YES |
| SuggestedPermissions | OK | OK | No | No | Yes (custom) | YES |
| SampleSelection (embeddable) | N/A | N/A | N/A | N/A | N/A | SEPARATE CONCERN |
| TestFormSpreadsheet (embeddable) | N/A | N/A | N/A | N/A | N/A | SEPARATE CONCERN |

**Summary:**
- **13 experiences** ready to migrate now
- **8 experiences** need `titleContextMenuGroups` fix first (shared
  base class fix covers most of them)
- **1 experience** needs `configurationData` fix (DashboardRich) +
  titleContextMenuGroups
- **2 experiences** blocked on embedding support (Sampling, TestForm)
- **2 embeddable experiences** are a separate migration concern
- **1 callsite** needs `goTo` -> `goToExperience` fix
