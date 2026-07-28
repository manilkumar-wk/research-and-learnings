# Dart Experience to MFE Migration: Step-by-Step Guide

A plain-language walkthrough of the
[`dart-experience-to-mfe`](https://github.com/Workiva/fef-ai/pull/25)
skill from `Workiva/fef-ai`. This skill converts a legacy Workiva
experience (a UI panel or full page inside wdesk) into a microfrontend
(MFE) -- a self-contained app that loads independently instead of being
compiled into wdesk.

---

## Background: Two types of experiences

Before diving into the steps, know which type you are migrating:

| Type | Base class | Where it renders | Example |
|---|---|---|---|
| **Drawer** | `DrawerExperienceConfig` | Left sidebar panel | Reports list, Dashboards list, Data Types |
| **Rich** | `RichExperienceConfig` | Main content area (full page) | A Report, a Test Form, a Dashboard view |

Both follow the same overall workflow. The differences are noted at each
step.

---

## Step 0: Pre-flight checks

Before writing any code, verify these five requirements. If any fail,
fix them in a separate PR first.

### Check 1: `configurationData` must be a `String`

If the experience passes data via `configurationData`, it must be a
JSON-serialized `String`, not an arbitrary Dart object.

**What to do:**
- Find all callsites that use `navigator.goTo` and change them to
  `navigator.goToExperience`, which enforces the `String` type.
- If the experience passes complex objects, JSON-encode them before
  passing and JSON-decode on the receiving end.

### Check 2: Context menus must call `.showContextMenu()`

MFEs are sandboxed -- they do not share a `ContextMenuManager` with the
main wdesk app. The MFE must explicitly call `show` after adding menu
items.

**What to do:**
- Search for context menu usage in the experience code.
- Make sure every context menu creation ends with an explicit
  `contextMenuManager.showContextMenu(...)` call.

### Check 3: No `titleContextMenuGroups`

The legacy `titleContextMenuGroups` API does not work inside MFEs.

**What to do:**
- Replace `titleContextMenuGroups` with
  `experienceContext.shell.setContextMenuGroups(...)`.

### Check 4: (Rich only) No embedding

If a rich experience embeds another experience inside itself, it cannot
be migrated to an MFE yet. This is a hard blocker.

**What to do:**
- Check if the experience calls any embedding APIs.
- If it does, skip this experience for now and migrate others first.

### Check 5: (Rich only) No key bindings dependency

MFE key bindings are isolated from the main app. They will not appear
in the `KeybindingModal` and may collide with wdesk shortcuts.

**What to do:**
- Check if the experience registers key bindings via
  `KeyBindingManager`.
- If it does, decide whether the isolation is acceptable or if you need
  to rework the key binding approach first.

### Not a blocker

`NotificationManager` used to require migration to
`NotificationService`, but a `NotificationManager` API is now exposed
on `AppContext` for legacy microfrontends. No action needed.

---

## Step 1: Add dependencies

Add these packages to your `pubspec.yaml`:

```yaml
dependencies:
  microfrontend:
    hosted:
      name: microfrontend
      url: https://pub.workiva.org
    version: ^1.5.0

  # Only needed for rich experiences:
  rich_experience_contribution:
    hosted:
      name: rich_experience_contribution
      url: https://pub.workiva.org
    version: ^1.43.3

dev_dependencies:
  wdesk_sdk_builders:
    hosted:
      name: wdesk_sdk_builders
      url: https://pub.workiva.org
    version: ^2.1.29
```

| Package | When needed | Purpose |
|---|---|---|
| `microfrontend` | Always | Core MFE framework |
| `rich_experience_contribution` | Rich experiences only | Rich experience base classes and contribution host |
| `wdesk_sdk_builders` | Always (dev only) | Generates `build_meta.g.dart` with asset loader |

Bump to the latest published versions at the time of migration.

---

## Step 2: Create a `web/` directory

Create a `web/` directory at the **repository root** (not inside
`app/`).

```
graph_app/
├── app/          <-- existing legacy app (keep during transition)
├── web/          <-- NEW: MFE entrypoints go here
├── lib/
├── pubspec.yaml
└── manifest.yaml <-- NEW: will create in Step 6
```

**Why at the root?** This simplifies build configuration. The existing
`app/` directory stays in place during the transition period -- both
paths work simultaneously. Once all experiences are stable as MFEs, you
can remove `app/`.

---

## Step 3: Create a Contribution class

A Contribution class is a **wrapper** around your existing experience
factory. You create one class per experience. The key insight is that
you do not need to rewrite the experience -- the adapter pattern wraps
your existing code as-is.

### For a Drawer experience

```dart
import 'package:wdesk_sdk/experience_contributions.dart';

class ReportListContribution extends DrawerExperienceContribution {
  @override
  String get extensionName => 'w_sox';

  @override
  String get simpleName => 'report_list';

  @override
  Future<DrawerExperience> openExperience() =>
      legacyDrawerExperienceFactoryAdapter(
        // Pass your EXISTING factory function -- no rewrite needed
        reportListExperienceFactory,
        // Copy from the experience config's routeSegment field
        routeSegment: 'reports',
        // Copy from the experience config's stylesheets field
        stylesheets: [
          'packages/w_sox/src/reports/report_list.css',
        ],
      );
}
```

### For a Rich experience

```dart
import 'package:microfrontend/client_sdk.dart';
import 'package:rich_experience_contribution/contribution.dart';
import 'package:static_asset_loader/static_asset_loader.dart';
import 'package:wdesk_sdk/legacy_rich_experience_adapter.dart';

class ReportContribution extends RichExperienceContribution {
  @override
  String get extensionName => 'w_sox';

  @override
  String get simpleName => 'report';

  // Called when the experience is opened without a resource ID
  // (e.g. "create new" flows). Return a placeholder string.
  @override
  Future<String> createResource() async {
    return 'create';
  }

  @override
  Future<RichExperience> open(OpenArgs args) async {
    // Load stylesheets from the experience config's staticAssets
    final sal = services.get<StaticAssetLoader>();
    await sal.loadAll([
      'packages/w_sox/src/reports/report.css',
    ]);

    return legacyRichExperienceFactoryAdapter(
      // Pass your EXISTING factory function -- no rewrite needed
      reportExperienceFactory,
      shell: args.shell,
      // Copy from the experience config's routeSegment field
      routeSegment: 'report',
      resourceId: args.resourceId,
      configurationData: args.configurationData,
    );
  }
}
```

### Where to find the values

| Property | Where to look |
|---|---|
| `extensionName` | Choose a name (convention: snake_case, often the repo name). Must be the same everywhere. |
| `simpleName` | Unique ID for this experience. Used in `manifest.yaml`. |
| `routeSegment` | Copy from the experience config class's `routeSegment` getter. |
| `stylesheets` | Copy from the experience config class's `stylesheets` or `staticAssets` getter. |
| Factory function | The existing factory method referenced in the experience config. |

### About `createResource()` (rich only)

This method is called when a rich experience is opened without a
resource ID (common for "create new" flows). If your factory handles
`null` resource IDs, return a placeholder like `'create'` and update
the factory to treat `'create'` the same as `null`. The actual
migration of creation logic into `createResource` can be deferred.

### About the adapter functions

- `legacyDrawerExperienceFactoryAdapter` wraps an existing drawer
  factory into an MFE-compatible one.
- `legacyRichExperienceFactoryAdapter` wraps an existing rich factory
  into an MFE-compatible one.

These adapters avoid the need to rewrite experience code during the
initial migration. They also make it easier to toggle between legacy
and MFE paths via LaunchDarkly flags. Long-term, you should migrate to
native MFE base classes (`BasicDrawerExperience`, MFE
`RichExperience`), but that is a separate effort after the transition
is stable.

---

## Step 4: Create an Extension class

An Extension groups multiple Contribution classes together and
registers them with the MFE framework.

```dart
import 'package:wdesk_sdk/experience_contributions.dart';

class WsoxExtension extends WdeskExtension {
  WsoxExtension(StaticAssetLoader assetLoader)
      : super.withStaticAssetLoader(assetLoader, 'w_sox');

  @override
  void onRegisterContributions(RegisterContribution register) {
    // Drawer experiences -- wrap each with DrawerExperienceContributionHost
    register(DrawerExperienceContributionHost(
        ReportListContribution()));
    register(DrawerExperienceContributionHost(
        DashboardListContribution()));
    // ... one line per drawer experience

    // Rich experiences -- wrap each with RichExperienceContributionHost
    register(RichExperienceContributionHost(ReportContribution()));
    register(RichExperienceContributionHost(DashboardContribution()));
    // ... one line per rich experience
  }
}
```

### Incremental rollout with multiple Extensions

For a repo with many experiences (graph_app has 27), you do not have to
migrate all at once. Use multiple extensions:

1. **Stable extension** -- contains experiences that are fully tested
   and validated. No LaunchDarkly flag. Always active.
2. **In-progress extension** -- contains experiences still being
   validated. Controlled by a LaunchDarkly flag. Can be toggled off if
   issues arise.
3. **Per-experience extensions** -- each experience gets its own
   extension and LD flag for maximum control.

This avoids big-bang risk and lets you validate one experience at a
time.

---

## Step 5: Create the MFE entrypoint

Create `web/main.dart` in the `web/` directory from Step 2:

```dart
import 'package:wdesk_sdk/create_mfe.dart';
import 'build_meta.g.dart'; // auto-generated by wdesk_sdk_builders

Future<void> main() async {
  // Write build metadata so wdesk can identify this MFE
  writeMfeBuildMetadataToWindow();

  // Initialize the MFE framework
  await createMfe(
    assetLoader: assetLoader,  // from build_meta.g.dart
    intlName: 'w_sox',         // internationalization name
    mfeName: 'w_sox',          // must match manifest.yaml microfrontend name
  );

  // Register all experience contributions
  WsoxExtension(assetLoader);
}
```

**What is `build_meta.g.dart`?** It is a generated file created by the
`wdesk_sdk_builders|mfe` builder. It provides two things:
- `assetLoader` -- loads static assets (CSS, images) for the MFE
- `writeMfeBuildMetadataToWindow()` -- writes version info to the browser
  window so wdesk can identify the MFE

You do not write this file. It appears automatically after you configure
`build.yaml` in Step 7.

---

## Step 6: Write `manifest.yaml`

Create `manifest.yaml` at the **repository root**. This file tells FEWS
(the frontend web service) what your MFE contributes to wdesk.

```yaml
version: 1

microfrontends:
  w_sox:                          # microfrontend name
    apps: [wdesk]                 # target application
    extensions:
      w_sox:                      # must match extensionName in Dart code
        entrypoint: main.dart.js  # compiled JS from web/main.dart
        oauth2_scopes:            # from experience config oauth2Scopes
          - iam|r
          - w_sox|r
          - w_sox|w
        contributions:

          # --- DRAWER EXPERIENCES ---
          # Declares: "I provide these sidebar panels"
          core.drawer_experiences:
            - name: report_list             # must match simpleName
              details:
                display_name:
                  intl_message_name: reportList
                  default_text: Reports
                can_user_access: ability.MODULE_GRAPH
                is_valid_landing_page: true  # can this be a landing page?

          # --- RICH EXPERIENCES ---
          # Declares: "I provide these full-page views"
          core.rich_experiences:
            - name: report
              details:
                display_name:
                  intl_message_name: report
                  default_text: Report
                can_user_access: true

          # --- ROUTING ---
          # Declares: "These URL paths map to my experiences"
          core.routing:
            - name: report_list_route
              details:
                route_segment: reports      # the URL path segment
                experience_name: w_sox.core.drawer_experiences.report_list
                # Pattern: <extension>.<contribution_point>.<simpleName>

            - name: report_route
              details:
                route_segment: report
                experience_name: w_sox.core.rich_experiences.report

          # --- NAVIGATION SIDEBAR ---
          # Declares: "Add these links to the left nav"
          # Only for drawer experiences that are visible in nav
          core.navigation_sidebar:
            - name: reports_nav
              details:
                icon: unify.summarize       # Unify icon name
                text:
                  intl_message_name: reports
                  default_text: Reports
                url: reports                # matches route_segment
                location: default@12        # default@<sortOrder>

          # --- CREATE MENU (rich only) ---
          # Declares: "Add this to the create/new menu"
          menu.menus:
            - name: create_report
              details:
                icon: unify.autoAwesome
                text:
                  intl_message_name: report
                  default_text: Report
                url: report_route
                location: z@1001
                menu: create
```

### Where to find each manifest value

| Manifest field | Source in existing code |
|---|---|
| `display_name` | Experience config's `displayName` getter |
| `can_user_access` | Ability/licensing expression. See [can-user-access docs](https://github.com/Workiva/wdesk_sdk/blob/master/doc/microfrontends/can-user-access.md) |
| `route_segment` | Experience config's `routeSegment` getter |
| `experience_name` | `<extensionName>.core.<drawer_experiences\|rich_experiences>.<simpleName>` |
| `icon` | `DrawerExperienceConfig.navItemIcon` -- prefix Unify icons with `unify.` |
| `location` (sidebar) | `default@<sortOrder>` from experience config's `sortOrder` |
| `location` (menu) | `z@<position>` or `base@<position>` from `addCreateMenuItem` call |
| `oauth2_scopes` | Experience config's `oauth2Scopes` getter |

### Nested drawer experiences (tabbed UIs)

If a drawer experience composes child experiences (e.g. tabs), add
`core.drawer_composition` entries:

```yaml
          core.drawer_composition:
            - name: child_one_composition
              details:
                parent: w_sox.core.drawer_experiences.parent_experience
                child: w_sox.core.drawer_experiences.child_one
                sort_order: 1
            - name: child_two_composition
              details:
                parent: w_sox.core.drawer_experiences.parent_experience
                child: w_sox.core.drawer_experiences.child_two
                sort_order: 2
```

The parent contribution class returns a `ComposingDrawerExperience`
from `openExperience()`, and each child gets its own
`DrawerExperienceContribution`.

---

## Step 7: Update `build.yaml`

Two changes are needed:

### 7a: Enable the MFE builder

Follow the
[wdesk_sdk_builders configuration docs](https://github.com/Workiva/wdesk_sdk_builders#configuration-1)
to enable `wdesk_sdk_builders|mfe`. This generates `build_meta.g.dart`.

### 7b: Tell the compiler about the new entrypoint

If `build_web_compilers|entrypoint` has a `generate_for` list, add the
MFE entrypoint:

```yaml
targets:
  $default:
    builders:
      build_web_compilers|entrypoint:
        generate_for:
          - "test/**"
          - "web/main.dart"   # <-- add this line
```

Without this, the compiler will not compile `web/main.dart` into
`main.dart.js` and the MFE will not load.

---

## Step 8: Configure local serving

Update `tool/dart_dev/config.dart` to use the MFE serve tool:

```dart
import 'package:dart_dev_workiva/dart_dev_workiva.dart';

final config = {
  ...workivaConfig,
  'serve': MicrofrontendServeTool()
    ..app = 'simple_mfe_app'           // lightweight MFE shell
    ..localMicrofrontends = ['w_sox']  // your microfrontend name
};
```

Then run:

```bash
ddev serve
```

Open `localhost:8080` in a browser. Your MFE loads inside
`simple_mfe_app`, a lightweight shell designed for local MFE
development. You can also use `wdesk` as the app, but it is heavier
and slower to start.

---

## Step 9: Set up mock auth

For local development, you need mock authentication.

### Simple case

If the existing `app/web/main.dart` uses `createApp` with no custom
`MockSession`, just add `?mockAuth=true` to the URL and it works
automatically.

### Custom credentials

If the app needs specific mock credentials, set environment variables:

| Env variable | What it sets | Default |
|---|---|---|
| `IAM_HOST` | Session host | wkdev or staging |
| `MOCK_SESSION_ACCOUNT_RESOURCE_ID` | Account resource ID | `QWNjb3VudB8xMTE=` |
| `MOCK_SESSION_MEMBERSHIP_RESOURCE_ID` | Membership resource ID | `TWVtYmVyc2hpcB8yMjI=` |
| `MOCK_SESSION_USER_RESOURCE_ID` | User resource ID | `V0ZVc2VyHzMzMw==` |
| `MOCK_SESSION_EMAIL` | User email | `jon.snow@workiva.com` |
| `MOCK_SESSION_USERNAME` | Username | `jon.snow` |
| `MOCK_SESSION_NAME` | Display name | `Jon Snow` |

Two ways to set these:

**Option A:** In `tool/dart_dev/config.dart` via `browserEnvVars` on
the `MicrofrontendServeTool`.

**Option B:** As shell env vars with `BROWSER_ENV_` prefix:
```bash
BROWSER_ENV_MOCK_SESSION_EMAIL=sox.admin@workiva.com ddev serve
```

---

## Step 10: Release preparation

### 10a: Set up the release pipeline

Configure the release pipeline to deploy `manifest.yaml` to FEWS
(frontend-web-service). FEWS reads the manifest and makes the MFE
available to wdesk at runtime.

### 10b: Create a LaunchDarkly flag

Create an LD flag to control rollout. This lets you toggle between the
legacy compile-time experience and the new MFE path. If the MFE has
issues in production, flip the flag to revert to legacy instantly
without a deploy.

### 10c: Roll out gradually

1. Enable the flag in **wk-dev** first and test.
2. Enable in **staging** and run Signals tests.
3. Enable in **production** and monitor.

---

## Troubleshooting checklist

If the MFE does not load, check these in order:

- [ ] **Entrypoint filename** -- does the `entrypoint` in
      `manifest.yaml` match the actual compiled JS filename?
- [ ] **Extension name consistency** -- is `extensionName` the same in
      the Contribution class, Extension class, and `manifest.yaml`?
- [ ] **build.yaml** -- does `generate_for` include
      `web/main.dart`?
- [ ] **wdesk_sdk_builder disabled** -- is
      `wdesk_sdk_builders|wdesk_sdk_builder` disabled so the serve
      tool's `index.html` is not overwritten?
- [ ] **OAuth2 scopes** -- are the scopes in the manifest correct for
      the current user and workspace?
- [ ] **can_user_access** -- does the access expression evaluate to
      `true` for the test user?
- [ ] **Routing experience_name** -- does `core.routing` correctly
      reference the experience using the
      `<extension>.core.<type>.<simpleName>` pattern?

For help, reach out on Slack: **`#support-frontend-architecture`**.

---

## Summary: What goes where

```
repository-root/
├── web/
│   └── main.dart              <-- Step 5: MFE entrypoint
├── lib/
│   └── src/
│       └── mfe/
│           ├── contributions/
│           │   ├── report_list_contribution.dart   <-- Step 3
│           │   ├── report_contribution.dart        <-- Step 3
│           │   └── ...
│           └── w_sox_extension.dart                <-- Step 4
├── manifest.yaml              <-- Step 6: MFE manifest (repo root)
├── pubspec.yaml               <-- Step 1: add dependencies
├── build.yaml                 <-- Step 7: enable MFE builder
├── tool/
│   └── dart_dev/
│       └── config.dart        <-- Step 8: local serving config
└── app/                       <-- existing legacy app (kept during transition)
    └── web/
        └── main.dart          <-- old createApp() entrypoint
```

---

## Reference

- Skill source:
  [`Workiva/fef-ai` PR #25](https://github.com/Workiva/fef-ai/pull/25)
- Install: `pnpx skills add workiva/fef-ai`
- Drawer templates:
  `skills/dart-experience-to-mfe/references/drawer-templates.md`
- Rich templates:
  `skills/dart-experience-to-mfe/references/rich-templates.md`
- can-user-access docs:
  [wdesk_sdk/doc/microfrontends/can-user-access.md](https://github.com/Workiva/wdesk_sdk/blob/master/doc/microfrontends/can-user-access.md)
- Support: `#support-frontend-architecture` on Slack
