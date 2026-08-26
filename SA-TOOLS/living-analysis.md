# Dart to TypeScript Migration — Living Technical Analysis

**Status:** Living document. Analyzed repositories are listed below.
Update this file when each new repository is reviewed. Do not delete
prior findings.

| Repository | Package name | Role | Analysis date | Status |
| --- | --- | --- | --- | --- |
| **graph_app** | `w_sox` (library) + `w_sox_app` (dev/test app) | Main GRC / IntRisk client | 2026-08-25 | Complete for this repo |
| audit | — | — | — | Not yet analyzed |
| graph_ui | — | — | — | Not yet analyzed |
| graph_admin | — | — | — | Not yet analyzed |
| WDesk / other platform repos | — | — | — | Not yet analyzed |

**Sources used for graph_app**

- Traced source, configs, tests, and workflows in this workspace
  (confirmed).
- User-provided challenge list:
  `Dart-TS-Migration-challenges.xlsx`, sheet `Overall-list-by-AI`
  (spreadsheet; treated as prior analysis, not re-verified against
  other repos). Sheet `Top-List` contained no extractable cell data.
- Prior private research (not re-opened as source of graph_app
  behavior; used only to name existing TS packages):
  `SA-TOOLS/dart-to-typescript-migration-poc-analysis_claude.md`,
  `SA-TOOLS/dart-to-typescript-repo-migration-guide.md`,
  `SA-TOOLS/dart-pubsub-to-typescript-guide.md` in
  `research-and-learnings`. Those docs cover **ts-grc** and other GRC
  clients. They do **not** replace this graph_app code trace.

**Correction vs prior research:** the 2026-08-19 POC analysis says
`graph_app` pub-depends on `grc_universe_client`,
`framework_explorer`, `assessments_client`, and `requests_client`.
**Confirmed in this pass:** those four packages are **not** in
`graph_app/pubspec.yaml` and are **not** imported from `lib/`.
graph_app today compiles **audit**, **graph_ui**,
**grc_testing_client**, landing-page, and viewer configs. Universe /
assessments / frameworks / requests may now live in WDesk or ts-grc
directly — verify when those repos are reviewed.

**How to read evidence labels**

| Label | Meaning |
| --- | --- |
| **Confirmed** | Observed in graph_app source, pubspec, tests, or CI |
| **Spreadsheet** | From the known-challenges workbook; not independently verified in this pass |
| **Assumption** | Reasonable inference; not proven from this repo's code |
| **Open** | Unanswered; needs platform or other-repo confirmation |

This document does not recommend starting a rewrite. It explains what
graph_app is, what it depends on, and what must be true before
TypeScript work is safe.

**Manager briefing (share before POC):**
[`graph_app-ts-migration-manager-briefing.xlsx`](graph_app-ts-migration-manager-briefing.xlsx)
— complexities, challenges, blockers, and decisions needed before a
POC. Use that workbook in a short review; keep this markdown for
engineers.

---

## 1. Repository Overview

**What graph_app is, in simple terms**

graph_app is the main browser UI for Workiva IntRisk GRC (historically
called SOX). Users do not open this repo as a standalone website in
production. WDesk (the Workiva desktop shell) loads it as a Dart
package named `w_sox`. The shell shows GRC screens as "experiences":
sidebar drawers (lists) and rich panels (forms, reports, testers).

The published Dart package is `w_sox` version `10.4.54`
(`pubspec.yaml`). A nested package `app/` (`w_sox_app`) is the local
dev and FEWS deploy app. Functional tests live in `dart_functional/`.

**Why this repo is the center of the migration**

- It owns GRC product experiences: testing, data, reports, dashboards,
  planning, evidence testing, exports, support, landing-page widgets.
- It **compiles Audit into the same WDesk app** via `package:audit`.
- It depends on **graph_ui** for shared graph forms, tables, and
  services.
- It talks to many backend services over **Frugal + NATS**, plus some
  REST/HTTP.
- WDesk today consumes it with `w_sox:` in a Dart `pubspec.yaml`. A
  TypeScript package cannot be added that way.

**Scale (confirmed file inventory, 2026-08-25)**

| Area | Count |
| --- | --- |
| Dart files in `lib/` | 1,238 (about 1,099 handwritten; 139 generated `.sg.dart` / `.sg.gen.dart` / `.g.dart`) |
| Unit/test Dart in `test/` | 460 files (370 `*_test.dart`) |
| Functional Dart in `dart_functional/` | 103 files (44 `*_test.dart`) |
| Nested app Dart | 7 |
| Experience config files | 30 |
| Redux-related files under `lib/` | 104 |
| Flux-related files under `lib/` | 14 |
| SCSS | 210 |
| GitHub workflow files | 14 |
| TypeScript application code | **None** (only a few helper JS files for functional-test Chrome extensions and FastClick) |

**Largest `lib/src` areas by file count**

| Area | Files | Product meaning |
| --- | --- | --- |
| `sox/` | 394 | Testing: test forms, bulk import, sample selection, GRC services |
| `reports/` | 289 | Reports, smart table, charts, dashboards, textual query |
| `data_experience/` | 132 | Data list, data forms, raw graph, snapshots, suggested permissions |
| `_shared/` | 72 | Graph module, access, HTTP, logging, panels |
| `planning/` | 69 | Resource plans / Gantt |
| `landing_page_widgets/` | 62 | Home/landing widgets registered with WDesk |
| `evidence_testing/` | 60 | Evidence tester + markup/attachment viewers |
| `upgrade_api/` + `upgrade_config/` | 79 | OpenAPI/Dio upgrade/migration REST client and wizard |
| `configuration/` | 31 | Data model editor + Support tools |
| `export_list/` | 29 | Testing export jobs list |

---

## 2. Architecture and Responsibilities

### 2.1 Main responsibilities

graph_app is responsible for:

1. **Registering GRC screens with WDesk** (drawer, rich, embeddable
   experiences, plus landing-page widgets).
2. **Connecting to the GRC graph** through `w_graph_client` and
   `graph_ui`, using NATS subjects and some HTTP Frugal/REST URLs.
3. **SOX testing workflows** (test form, test phases, bulk import,
   sample selection, exports).
4. **Reports, dashboards, and charts** (including Highcharts and
   `w_table` "smart table" plugins).
5. **Evidence testing** (attachments, markup, PDF.js via LaunchDarkly
   path).
6. **Access control** (licensing abilities + workspace solution types).
7. **Hosting Audit experiences** by re-exporting Audit configs in the
   same registry WDesk loads.

It is **not** responsible for login. The WDesk shell owns OAuth and
the session cookie.

### 2.2 Packages and entry points (confirmed)

| Package | Path | Purpose |
| --- | --- | --- |
| `w_sox` | repo root | Published Dart library consumed by WDesk |
| `w_sox_app` | `app/` | Local serve / FEWS deploy host |
| `dart_functional` | `dart_functional/` | Puppeteer + Skynet functional tests |

**Public Dart exports (what WDesk and the test app import)**

- `lib/experience_configs.dart` — every GRC `*ExperienceConfig`
- `lib/landing_page_widgets.dart` — landing-page widget configs
- `lib/draft_edits_header.dart`, `lib/graph_archive_header.dart` —
  header widgets for draft/archive modes

**WDesk / local app registry**

`app/lib/src/graph_experience_registry.dart` is the production-shaped
registry used by the nested app. It:

- Orders drawer nav items (landing page, planning, testing, **Audit
  drawers**, reports, dashboards, data, TQ, model, support, time
  entry, project files, create menu).
- Registers rich experiences (data forms, reports, dashboards, test
  form, evidence testing, viewers, bulk import, export list, Audit
  rich experiences, report builder, resource plan).
- Registers embeddable experiences (Audit embeddables + sample
  selection spreadsheet + test-form spreadsheet).
- Switches to a reduced set in archive/snapshot mode.

Comment in
`app/lib/src/ir_archive_mode_experience_registry.dart`:
the archive registry is **no longer used for production WDesk**, but
is still useful for local graph development.

**Runtime module entry (not a `main()` for production)**

`lib/src/_shared/graph_module.dart` (`GraphModule extends Module`) is
the shared load/unload unit. `README.md` still says consumers call
`SoxModule.loadStaticAssets(...)`. **Confirmed:** there is no class
named `SoxModule` in this repo. Static assets are loaded per
experience via `loadStaticAssets` on each `*ExperienceConfig`. The
README name is stale. The real pattern is WDesk
`ExperienceRegistry` + `GraphModule.onLoad()`.

### 2.3 Important workflows (confirmed)

| Workflow | How it starts | What it uses |
| --- | --- | --- |
| Open a GRC screen | WDesk matches `routeSegment` on an experience config, checks `canUserAccessV2`, loads deferred Dart library + CSS/JS assets | `wdesk_sdk` experience framework |
| Load graph data | `GraphModule.onLoad` builds `GraphClientModule` with NATS subjects and optional HTTP for tables | `w_graph_client`, `messaging_sdk` |
| SOX testing | Testing drawer / test form rich experience | `grc_services_frugal` over NATS (HTTP factory exists), licensing abilities |
| Bulk test-form import | Dedicated rich experience; completion events also reach Testing list | Redux + `event_bus` pub/sub via getIt |
| Reports / dashboards | Report list + rich report + dashboard experiences | graph RPC, `w_dashboard`, Highcharts, `w_table` |
| Evidence testing | Rich experience + attachment/markup viewers | `w_viewer`, `drawing`/`markup`, Audit attachment services |
| Upgrade / model migration | Support / upgrade wizard | OpenAPI client (`dio`) to `graph-rpc-service` REST |
| Landing page | `TabbedLandingPageExperienceConfig` with widget map | `w_landing_page` + widgets from this repo and Audit form recency |

### 2.4 Dual state systems (confirmed)

graph_app is **not** Redux-only and **not** Flux-only:

- **Flux (`w_flux`)** powers the shared `GraphModule` / `SoxStore` and
  older screens (test form, focus, dashboards, textual query, report
  builder, sample selection). 14 Flux-path files under `lib/`.
- **Redux (`redux` + `over_react_redux`)** powers newer slices (data
  list, bulk import, planning rich, export list, suggested
  permissions, some landing widgets). 104 Redux-path files under
  `lib/`.

These stores are per experience or per module. They are not one global
app store.

---

## 3. Dependencies and Integrations

### 3.1 How graph_app talks to other applications

```text
WDesk shell (Dart)
  pub-depends on w_sox
    registers GraphExperienceRegistry-equivalent configs
    provides AppContext (session, licensing, FrugalMessagingProvider,
             modal/notification managers, navigator)
      w_sox experiences
        depend on graph_ui (shared graph UI/services)
        depend on audit (Audit experiences compiled in)
        call Frugal/NATS backends (graph, GRC services, licensing,
             form export, attachments, identity)
        call some REST (graph-rpc upgrade API, jobs HTTP Frugal,
             suggested permissions URL)
        load CDN CSS/JS from sibling packages (w_table, workflow_forms,
             graph_app_js, Highcharts, drawing, w_viewer)
```

**Confirmed consumer contract** (`README.md` "Consuming SOX"):

- Add `w_sox` to a Dart `pubspec.yaml`.
- Load static assets from CDN:
  `https://cdn-prod.wdesk.com/graph_app/[tag]/`.
- Production is **not** an npm/MFE-only app today.

**Confirmed CI consumer coupling** (`.github/workflows/ci.yml`):
full builds also run on `_integration/Workiva/{highcharts,
language_translator_client, w_table, w_outline, w_comments, w_router,
wdesk_sdk, wdesk_login, data_sharing_service_sdk_dart}/**`.
`.github/workflows/run-wdesk-also-run-test.yml` runs graph_app
functional tests in a WDesk-like Docker compose stack, including
`doc_plat_client` MFE version overrides.

### 3.2 Internal Workiva Dart dependencies (from `pubspec.yaml`, confirmed)

Grouped by role. Versions are those pinned/ranged in graph_app; they
will drift.

**Shell / MFE / session**

`wdesk_sdk`, `wdesk_browser_environment`, `microfrontend`,
`rich_experience_contribution`, `modal_mfe_service`,
`notification_mfe_service`, `w_session`, `w_module`, `w_router`,
`static_asset_loader`, `home`, `w_landing_page_sdk`

**Graph / GRC APIs**

`graph_api`, `graph_form_api`, `graph_rpc_api`, `grc_services_frugal`,
`w_graph_client`, `graph_ui`, `frugal`, `thrift`, `messaging_sdk`

**Auth / identity / licensing**

`licensing_api`, `licensing_frugal`, `identity_sdk`

**Audit product (compiled in)**

`audit`, `audit_api`

**UI platform**

`over_react`, `react`, `unify_ui`, `web_skin`, `web_skin_dart`,
`w_table`, `w_comments`, `w_attachments`, `w_attachments_client`,
`w_outline`, `w_viewer`, `w_dashboard`, `w_dashboard_frugal`,
`w_context_menu`, `w_clipboard`, `w_input_validation`,
`w_virtual_components`, `drawing`, `markup`,
`embedded_spreadsheet_api`, `highcharts`, `graph_app_js`,
`publish_ui`

**Content / files / linking**

`content_management_sdk`, `bigsky_rest_files`, `linking_sdk`,
`wuri_sdk`

**i18n / analytics / flags / cache**

`w_intl`, `w_translate_v2`, `language_translator_client`, `analytics`,
`user_analytics`, `app_intelligence`, `launch_darkly`,
`browser_storage`, `stash_4_5_3`, `stash_memory_4_5_3`

**Testing client (compiled into the app registry)**

`grc_testing_client` (Matrix experience)

**Current git overrides (confirmed in `pubspec.yaml`)**

- `audit` → GitHub `Workiva/audit` at a pinned SHA
- `wdesk_sdk` → GitHub `Workiva/wdesk_sdk` at a pinned SHA

These overrides mean graph_app is already coupled to **unreleased**
commits of Audit and WDesk SDK. Migration planning must treat those
SHAs as moving targets.

### 3.3 Public / third-party Dart dependencies (confirmed)

`archive`, `async`, `built_collection`, `built_value`, `collection`,
`contextual_message`, `dio`, `dnd`, `event_bus`, `fluri`, `get_it`,
`http_parser`, `intl`, `js`, `logging`, `memoize`, `meta`, `mime`,
`one_of_serializer`, `opentracing`, `path`, `quiver`, `redux`,
`rxdart`, `synchronized`, `tuple`, `uuid`, `w_common`, `w_flux`,
`w_transport`, `yaml`, `csv` (hosted on pub.dartlang.org)

### 3.4 Nested app extra dependencies (`app/pubspec.yaml`)

`w_landing_page` (not only the SDK), `intl_strings`,
`wdesk_sdk_builders` (dev). The app path-depends on `w_sox: path: ../`.

**Note:** `app/` still declares SDK `'>=2.11.0 <3.0.0'` while the root
package requires `'>=2.19.0 <3.0.0'`. Confirmed inconsistency; not a
TS issue, but the nested app is not on the same null-safety floor as
the library.

### 3.5 Backend service names and transports (confirmed in `lib/src/environment.dart` and callers)

| Service / subject | Transport | Used for |
| --- | --- | --- |
| `GRC_SERVICES_NATS_SUBJECT` default `grc-services` | NATS Frugal (HTTP factory also exists) | SOX/GRC testing APIs (`GRCServices`) |
| `DEFAULT_NATS_SUBJECT` default `graphJavaUser16` | NATS | Main graph |
| `GRAPH_TABLE_NATS_SUBJECT` | NATS | Table queries |
| `GRAPH_HISTORY_NATS_SUBJECT` | NATS | Historical graph |
| `ARCHIVE_NATS_SUBJECT` default `graphArchiveOperations16` | NATS | Archives/snapshots |
| `TQ_NATS_SUBJECT` / `graph-tq-server` | NATS or HTTP Frugal URL | Textual query |
| `DASHBOARD_NATS_SUBJECT` default `graphJavaDashboard` | NATS | Dashboards |
| `GRAPH_FORM_ATTACHMENT_SERVICE_NATS_SUBJECT` default `form-service-v2-attachments` | NATS | Form attachments |
| `graphFormExportService` | NATS RPC + NATS subscriber for statuses | Form/PDF export progress |
| `AUDIT_REQUEST_SERVICE_NATS_SUBJECT` | NATS | Audit requests |
| `graph-api-bridge` `/frugal` and `/jobs` | HTTP Frugal (jobs `useHttp: true`) | Graph + jobs |
| `graph-rpc-service` `/api/service/...`, `/rest`, `/api/permissions`, `/api/reports`, `/api/widget` | HTTP / REST | RPC, upgrade OpenAPI, reports, widgets |
| `identity` | HTTP OAuth client | Identity host in `http_graph_api.dart` |
| suggested permissions URL from graph_ui form-service | HTTP | Suggested permissions |

LaunchDarkly flag `ir-graph-ops-http` can switch **table** queries to
HTTP (`GraphModule._buildServiceConfig`).

---

## 4. Current Dart Functionality

Each row is a migration finding. Complexity uses Low / Medium / High.

### 4.1 WDesk integration and experience registry

| Field | Detail |
| --- | --- |
| Area | WDesk shell integration |
| Current Dart | `w_sox` is a Dart pub package. WDesk (and `w_sox_app`) import `package:w_sox/experience_configs.dart` and register drawer/rich/embeddable configs. Each config has `routeSegment`, deferred `experienceFactory`, `canUserAccessV2`, `oauth2Scopes`, and `loadStaticAssets`. |
| TS replacement | **Cannot** stay a Dart pub dependency. Options: (1) **hybrid** — keep Dart `w_sox` in WDesk pubspec and embed TS widgets (`useTsComponent` / Vite bundle in Dart — spreadsheet; **not in this repo**); (2) thin Dart facade that loads TS JS; (3) full TS MFE in **ts-grc** using `@workiva/microfrontend` + drawer/rich experience contributions (**prior research** — this is how universe/assessments already ship). Production standalone SPA without WDesk is **not** acceptable. |
| Related repos | WDesk, `wdesk_sdk`, `audit`, `graph_ui`, `w_landing_page`, `grc_testing_client`, `content_management_sdk`, `w_viewer` |
| Complexity | **High** |
| Risk / blocker | **Blocker for a full-repo TS cutover.** Until WDesk is TS or the shell loads an MFE, graph_app must keep a Dart surface. |
| Next step | Confirm with WDesk/platform that hybrid `useTsComponent` is the approved golden path for SOX. Do not remove `experience_configs.dart` until every experience is cut over. |
| Evidence | `lib/experience_configs.dart`; `app/lib/src/graph_experience_registry.dart`; `README.md` Consuming SOX; spreadsheet rows 21–27 |

**Registered GRC experiences (confirmed; Audit extras omitted until the audit repo is analyzed)**

| Kind | Config | `routeSegment` | OAuth scopes (where declared) |
| --- | --- | --- | --- |
| Drawer | Landing page (widgets) | from `w_landing_page` | shell |
| Drawer | Resource management | `GraphRoutePaths.resourceManagementList` | linking + sox + scim + fs_adapter |
| Drawer | Testing overview | `RouteSegments.overview` | `sox\|r`, `sox\|w`, `scim\|r` |
| Drawer | Reports | `GraphRoutePaths.reports` | linking + sox + scim + fs_adapter |
| Drawer | Dashboards | `GraphRoutePaths.dashboards` | same |
| Drawer | Data | `GraphRoutePaths.dataDrawer` | (via graph_ui paths) |
| Drawer | Textual query | `GraphRoutePaths.textualQuery` | linking + sox + scim + fs_adapter |
| Drawer | Model | `GraphRoutePaths.dataModel` | same |
| Drawer | Support | `GraphRoutePaths.support` | same |
| Drawer | Project files | `GraphRoutePaths.projectFiles` | same |
| Rich | Data form / focus | `GraphRoutePaths.focus` | same |
| Rich | Create data | `GraphRoutePaths.focusNew` | same |
| Rich | Raw graph | `GraphRoutePaths.rawGraph` | same |
| Rich | Report | `GraphRoutePaths.report` | same |
| Rich | Dashboard | `GraphRoutePaths.dashboard` | same |
| Rich | Test form | `SoxRoutePaths.test_form` (`test_form`) | sox + scim |
| Rich | TQ edit | `GraphRoutePaths.textualQueryEdit` | linking + sox + scim + fs_adapter |
| Rich | Evidence testing | `evidence_testing` | `viewer\|r` |
| Rich | Markup / attachment viewers | `graph_markup_viewer`, `graph_attachment_viewer` | `viewer\|r` |
| Rich | Bulk import | `bulk_test_form_import` | sox + scim |
| Rich | Export list | `export_list` | sox + scim |
| Rich | Report builder | `GraphRoutePaths.reportBuilder` | linking + sox + scim + fs_adapter |
| Rich | Resource plan | `GraphRoutePaths.plan` | same |
| Rich | Sample selection | `select_samples` | sox + scim |
| Rich | Suggested permissions | `suggested-permissions` | (config present) |
| Embeddable | Sample selection spreadsheet, test-form spreadsheet | proxy spreadsheet configs | embedded |

`GraphRoutePaths` comes from **graph_ui**, not this repo. Deep links
must not change.

### 4.2 UI components and reusable components

| Field | Detail |
| --- | --- |
| Area | OverReact UI |
| Current Dart | `over_react` `UiComponent2` / `UiProps` and some `uiFunction`. Unify widgets via `unify_ui` (100+ lib files). Custom CSS from 210 SCSS files. |
| TS replacement | React 18 function components + `@workiva/unify` (prior research: ts-grc uses `@workiva/unify` with `@mui/material`). No OverReact→TS codemod. Hybrid: embed TS in Dart via `useTsComponent` (**spreadsheet**; **not present in this repo today**). Org golden path in prior research is **MFE replacement in ts-grc**, not TS-in-Dart. Both may be needed for leftover SOX screens. |
| Related repos | `unify_ui`, `over_react`, `web_skin` / `web_skin_dart` |
| Complexity | **High** (volume + Unify parity) |
| Risk | Visual/a11y drift if TS uses raw MUI instead of Workiva Unify |
| Next step | Inventory Unify Dart vs Unify TS component map on the platform Ecosystem Map. Pilot one screen. |
| Evidence | Widespread `package:unify_ui` imports; `pubspec.yaml`; no `useTsComponent` usage in this repo |

| Field | Detail |
| --- | --- |
| Area | `w_table` + Smart Table plugins |
| Current Dart | Heavy `w_table` plugin layer (cell editors, pills, markdown, context menus, bit cells, no-edit indicators). Used by reports, test steps, planning, changelog. |
| TS replacement | Keep as **Dart island** until a platform TS grid exists (**spreadsheet: check**). Plugins do not map 1:1. |
| Related repos | `w_table`, `graph_ui` |
| Complexity | **High** |
| Risk | **Blocker for report/test-step TS** if the grid must move with the screen. |
| Next step | Treat smart table as last-to-migrate. POC should not include a full grid rewrite. |
| Evidence | `lib/src/reports/smart_table/**`; many `package:w_table` imports |

| Field | Detail |
| --- | --- |
| Area | Platform UI islands |
| Current Dart | Comments, attachments, outline, viewer, dashboard, drawing/markup, embedded spreadsheet, workflow form JS/CSS, permissions editor CSS (transitive via graph_ui), Highcharts. |
| TS replacement | Stay Dart/MFE islands, or wait for official TS MFEs. Do not replace viewer with a generic PDF viewer or spreadsheet with a generic grid. |
| Related repos | `w_comments`, `w_attachments`, `w_outline`, `w_viewer`, `w_dashboard`, `drawing`, `markup`, `embedded_spreadsheet_api`, `workflow_forms`, `highcharts`, `graph_app_js` |
| Complexity | **High** |
| Risk | Missing TS libraries for several of these (**spreadsheet: check**) |
| Next step | For each island, ask platform: Dart-only, dual Dart+TS, or TS MFE already shipping? |
| Evidence | `lib/src/sox/sox_ui/src/experiences/static_asset_constants.dart`; experience `loadStaticAssets` lists |

### 4.3 State management

| Field | Detail |
| --- | --- |
| Area | Flux + Redux (both) |
| Current Dart | `GraphModule` owns `SoxStore` (Flux). Newer experiences use `redux` + `over_react_redux`. `w_module` `DispatchKey` / `EventsCollection` for module events. |
| TS replacement | Redux Toolkit + react-redux for Redux slices; conceptual map from Flux to RTK/zustand. **Dart and TS cannot share one store** (**spreadsheet**). Pass props/callbacks while a screen is hybrid. |
| Related repos | `graph_ui` (also mixed), `audit` (Redux-heavy per spreadsheet) |
| Complexity | **High** (two patterns + no shared store) |
| Risk | Duplicate state and event races during hybrid |
| Next step | Pick one TS state approach. Port store **per experience**, not globally. |
| Evidence | `lib/src/_shared/graph_module.dart`; `lib/src/_shared/flux/sox_store.dart`; Redux dirs under data/planning/export/bulk import |

### 4.4 Caching and local storage

| Field | Detail |
| --- | --- |
| Area | In-memory SOX caches |
| Current Dart | `stash_4_5_3` / `stash_memory_4_5_3`: test-form list, titles, controls; 8-hour touched expiry; hyperbolic eviction. Decorators on test-form and test-step repositories. |
| TS replacement | `lru-cache` or equivalent in-memory cache (**spreadsheet: partial/yes**). Behavior (TTL, max entries, title index) must be re-specified. |
| Related repos | none (local to SOX) |
| Complexity | **Low**–**Medium** |
| Risk | Stale test-form lists if TTL/invalidation is copied wrong |
| Next step | Document cache keys and invalidation when the Testing list is ported. |
| Evidence | `lib/src/sox/sox_ui/src/sox_caching.dart` |

| Field | Detail |
| --- | --- |
| Area | Browser `localStorage` |
| Current Dart | `browser_storage` `SyncStorage`: data-model autocomplete, data-list module, view options, sample-selection PBC trigger, evidence-testing frustration reporter. |
| TS replacement | Same keys via `localStorage` / `idb-keyval`. **Keep key names** during hybrid or users lose UI prefs. |
| Related repos | `browser_storage` |
| Complexity | **Low** |
| Risk | Key rename breaks existing users |
| Next step | Inventory all `SyncStorage` namespace strings before any TS storage helper. |
| Evidence | `lib/src/configuration/model/data_model_store.dart` (`'data-model'`, `'autocomplete'`); other `browser_storage` imports |

### 4.5 API clients, Frugal, REST, NATS

| Field | Detail |
| --- | --- |
| Area | Frugal + NATS (primary API) |
| Current Dart | `AppContext.frugalMessagingProvider` (`messaging_sdk`). `newNatsRpcClient` / `newNatsSubscriber` / `newHttpRpcClient`. GRC testing service defaults to NATS; HTTP factory exists. Graph client configured with multiple NATS subjects. Form export uses NATS RPC **and** a statuses subscriber (progress stream). Attachments, identity, linking streams also use the provider. |
| TS replacement | **Org prior research:** Frugal is **not supported in TypeScript**. ts-grc replaced Frugal with **Apollo Client (GraphQL)** + **RTK Query (REST)** against `grc-evergreen`. Pub/sub progress is often **Apollo polling**, not browser NATS. **This does not automatically cover graph_app:** SOX still uses `grc-services` NATS, `w_graph_client` NATS subjects, and form-export **subscribers**. GraphQL coverage for those APIs is **Open**. Until coverage exists, TS SOX must use a BFF, Dart bridge, or new GraphQL — not a drop-in Frugal JS client. |
| Related repos | `messaging_sdk`, `frugal`, `thrift`, `grc_services_frugal`, `graph_api`, `graph_form_api`, `graph_rpc_api`, `w_graph_client`, `graph_ui` |
| Complexity | **High** |
| Risk | **Top technical blocker.** HTTP-only drops live streams (export progress, graph change, licensing). |
| Next step | Platform decision: browser NATS TS SDK vs BFF. Do not assume HTTP Frugal is enough. |
| Evidence | `lib/src/_shared/graph_module.dart`; `lib/src/sox/grc_app/grc_services.dart`; `lib/src/_shared/services/graph_form_export_service.dart`; `lib/src/environment.dart` |

| Field | Detail |
| --- | --- |
| Area | REST / OpenAPI (upgrade) |
| Current Dart | Generated OpenAPI client under `lib/src/upgrade_api/gen/` using `dio` + `WorkivaDioInterceptor` + `get_it`. Base URL `graphRPCServiceRESTUrl`. Also uses graph version-change **stream** from `w_graph_client`. |
| TS replacement | `openapi-generator` / orval + `fetch`/`axios` (**spreadsheet: yes** for the REST part). Stream still needs NATS/graph SDK. |
| Related repos | `graph-rpc-service`, `graph_ui` (Dio interceptor) |
| Complexity | **Medium** (REST) + **High** (live progress) |
| Risk | Wizard "in progress" UX depends on graph streams, not only REST |
| Next step | Good **secondary** POC: call one upgrade REST endpoint from TS with the shell session. |
| Evidence | `lib/src/upgrade_api/upgrade_api_service.dart`; `tool/oas/upgrade_api_config.yaml` |

| Field | Detail |
| --- | --- |
| Area | HTTP helpers (`w_transport`, `dio`) |
| Current Dart | `http_graph_api.dart` uses `w_transport` OAuth2 clients for graph-api-bridge, identity, history — "things not supported over messaging". Jobs service is Frugal over HTTP (`useHttp: true`). |
| TS replacement | `fetch`/`axios` with shell-provided tokens (**never embed tokens**). Does **not** replace NATS. |
| Related repos | `w_transport`, `w_session` |
| Complexity | **Low** for REST; **High** if used as a false substitute for Frugal |
| Risk | Teams may overfit to Dio/REST and skip streams |
| Next step | List which calls are HTTP-only vs NATS-required before any BFF design. |
| Evidence | `lib/src/_shared/http_graph_api.dart`; `lib/src/configuration/support/services/job_service.dart` |

| Field | Detail |
| --- | --- |
| Area | API translators |
| Current Dart | Hand-written mapping from Frugal types to `built_value` / UI models (e.g. report list translators, evidence-testing model translator, SOX testing models). |
| TS replacement | Hand-written TS mappers / Zod. **No generator** for this layer (**spreadsheet**). |
| Related repos | generated `*_api` / `*_frugal` packages |
| Complexity | **High** |
| Risk | Business rules live here; a naive IDL→TS dump will be wrong |
| Next step | Port translators with the experience that owns them, with golden fixtures. |
| Evidence | `lib/src/reports/report_list/translators.dart`; `lib/src/evidence_testing/models/ete_model_translator.dart`; `lib/src/sox/grc_app/grc_services.dart` |

### 4.6 Authentication and authorization

| Field | Detail |
| --- | --- |
| Area | Login |
| Current Dart | None in this repo. Shell cookie + `w_session` on `AppContext`. |
| TS replacement | Prior research: `@workiva/session_mfe_service` (same session API from the shell). Spreadsheet also names `ts-w-session` / `@workiva/environment`. Do not build a TS login page. |
| Related repos | WDesk, `w_session` / `ts-w-session` (**spreadsheet: check**) |
| Complexity | **Low** (if shell-owned) |
| Risk | Low if followed; High if a separate TS auth is invented |
| Next step | POC must run **inside** WDesk or `w_sox_app` with real/mock shell session. |
| Evidence | `SoxAccess` mockAuth short-circuit; README local accounts (not copied here) |

| Field | Detail |
| --- | --- |
| Area | OAuth scopes |
| Current Dart | Per experience, e.g. test form `['sox\|r','sox\|w','scim\|r']`; data/reports add `linking\|*` and `fs_adapter\|r`; viewers `viewer\|r`. |
| TS replacement | **Copy the same scope strings** onto any MFE manifest. |
| Related repos | WDesk shell |
| Complexity | **Medium** |
| Risk | Shell rejects the experience if scopes drift |
| Next step | Export a machine-readable scope matrix from configs (do not invent new strings). |
| Evidence | `*experience_config.dart` `oauth2Scopes` getters |

| Field | Detail |
| --- | --- |
| Area | Licensing / abilities |
| Current Dart | `appContext.licensingApi.canUserV4` / `canUserAny`. `SoxAccess` + `canUser()` in `lib/src/_shared/access/access.dart` also checks workspace `contextKindIds` (SOX vs ERM vs ESG, etc.) and suppressed solutions (PBC, planning). Data access uses `ABILITY_DATA_ACCESS`. Testing uses `ABILITY_TESTING_ACCESS` and many phase/workflow abilities. DB admin via `isDBAdmin`. |
| TS replacement | `iam-sdk-ts` + licensing TS or BFF (**spreadsheet: check**). Reimplement the **same** ability IDs and suppression map. |
| Related repos | `licensing_api`, `licensing_frugal`, `graph_ui` (`ir_solution_constants`) |
| Complexity | **High** |
| Risk | **Compliance blocker** if gates differ. Snapshot/archive registries hide screens; those gates must be copied. |
| Next step | Golden tests: Dart vs TS `canUser` for SOX, ERM-only, ESG, classic account, mockAuth. |
| Evidence | `lib/src/_shared/access/access.dart`; `lib/src/sox/sox_access.dart`; `lib/src/sox/sox_abilities.dart`; `lib/src/data_experience/ability_access.dart` |

| Field | Detail |
| --- | --- |
| Area | Identity |
| Current Dart | `identity_sdk` `IdentityApi.fromFrugalMessagingProvider` in `IdentityService`. graph_app-only vs other GRC apps (**spreadsheet**). |
| TS replacement | Identity TS client or BFF. Still Frugal-backed today. |
| Related repos | `identity_sdk` |
| Complexity | **Medium**–**High** |
| Risk | Person pickers and SCIM scopes depend on this |
| Next step | Confirm TS identity SDK; else keep Dart identity island. |
| Evidence | `lib/src/_shared/services/identity_service.dart` |

### 4.7 Routing and navigation

| Field | Detail |
| --- | --- |
| Area | WDesk routes |
| Current Dart | `routeSegment` on each config; in-app navigation via `appContext.navigator.goToExperience(...)`. `w_router` used in graph experience base and several modules. Query params: `wkdev`, `localAuth`, `mockAuth`, `debug`, `dev`, `log`, archive/draft session params. |
| TS replacement | Same URI segments on the MFE; React Router only **inside** a screen. Deep links must not change. |
| Related repos | `w_router`, `graph_ui` (`GraphRoutePaths`), WDesk |
| Complexity | **Medium** |
| Risk | Broken bookmarks and landing-page recency links |
| Next step | Freeze a route-segment table as a compatibility contract. |
| Evidence | `lib/src/sox/sox_ui/src/experiences/sox_route_paths.dart`; experience configs; `lib/src/_shared/graph_experience_base.dart` |

### 4.8 Event handling and communication

| Field | Detail |
| --- | --- |
| Area | Module events |
| Current Dart | `w_module` `Event` / `DispatchKey` / `EventsCollection` across modules. Flux action streams. Redux middleware. |
| TS replacement | React callbacks, RTK listeners, or shell/MFE events. FEA ADR 0009 protobuf between MFEs (**spreadsheet**; not in this repo). |
| Related repos | `w_module`, WDesk |
| Complexity | **Medium**–**High** |
| Risk | Cross-experience events (import completed → testing list refresh) |
| Next step | Catalog cross-experience events before splitting MFEs. |
| Evidence | many `package:w_module` imports |

| Field | Detail |
| --- | --- |
| Area | Cross-experience pub/sub |
| Current Dart | `BulkTestFormImportPubSub` wraps `event_bus` and is registered in getIt so Testing list can hear import completion even though import is another experience. |
| TS replacement | Shared event bus, shell events, or keep Dart bus until both experiences move. |
| Related repos | none |
| Complexity | **Medium** |
| Risk | Silent missed refreshes if one side is TS |
| Next step | Include this bus in any bulk-import or testing-list POC. |
| Evidence | `lib/src/sox/bulk_test_form_import/bulk_test_form_import_pub_sub.dart` |

### 4.9 Shared Dart utilities and libraries

| Field | Detail |
| --- | --- |
| Area | graph_app `_shared` |
| Current Dart | Graph module cache, access helpers, logging, HTTP graph API, panels, i18n `w_sox_intl`, permission extensions, GRC header constants (Raven-monitored). |
| TS replacement | Rewrite as TS utils **or** keep Dart shared until multiple experiences move. |
| Related repos | consumed only inside `w_sox` |
| Complexity | **Medium** |
| Risk | Copy-paste drift if each TS island reimplements access/logging |
| Next step | Extract a small "host bridge" API (session, flags, navigate, log) for TS islands. |
| Evidence | `lib/src/_shared/**` |

| Field | Detail |
| --- | --- |
| Area | graph_ui |
| Current Dart | Imported throughout: `GraphRoutePaths`, `graph_services` / getIt-style `graphServices`, form services, stores, Dio interceptor, IR solution constants. |
| TS replacement | Stay Dart until audit + SOX are TS; then a TS graph_ui (**spreadsheet**). Migrating graph_ui first would break Dart consumers. |
| Related repos | **graph_ui**, audit, WDesk |
| Complexity | **High** |
| Risk | **Cross-repo blocker.** SOX cannot fully leave Dart while it imports graph_ui. |
| Next step | Analyze graph_ui as the next or second repo (user sequence permitting). |
| Evidence | `lib/src/_shared/graph_experience_base.dart`; `lib/src/_shared/services/identity_service.dart` (`graphServices`) |

| Field | Detail |
| --- | --- |
| Area | audit package |
| Current Dart | Registry spreads `auditDrawerExperiences`, `auditExperiences`, `auditEmbeddableExperiences`. Direct imports: Audit components, requests, attachments, models, redux store, evidence-testing helpers. Landing widgets delegate `canUserAccessV2` to Audit form configs. |
| TS replacement | Audit TS MFE **or** keep Audit as a Dart island inside SOX (**spreadsheet**). SOX cannot `pub get` a TypeScript Audit. |
| Related repos | **audit** |
| Complexity | **High** |
| Risk | **Blocker for "SOX is fully TS".** Git override already pins a custom Audit SHA. |
| Next step | Analyze audit next; decide island vs joint MFE. |
| Evidence | `app/lib/src/graph_experience_registry.dart`; `package:audit` imports under `lib/` |

### 4.10 Data models, serialization, codegen

| Field | Detail |
| --- | --- |
| Area | `built_value` / `.sg.dart` |
| Current Dart | 63 `.sg.dart` + 63 `.sg.gen.dart` models. `make gen` / `dart_dev source-gen`. |
| TS replacement | Hand TS types + Immer + Zod (**spreadsheet**). No `.sg.dart`→TS converter. |
| Related repos | `built_value` |
| Complexity | **High** |
| Risk | Silent field mismatch |
| Next step | Port models with the owning slice; do not bulk-dump. |
| Evidence | file inventory; `README.md` built_value section; `AGENTS.md` never-edit generated files |

| Field | Detail |
| --- | --- |
| Area | GRC meta-model |
| Current Dart | `data/grc_model.yaml` → generated `lib/src/grc_model/grc_model.dart` via Python/Mako (`make gen-dart`). Vertex/edge/property names without mirrors. |
| TS replacement | Generate TS constants from the **same YAML** (new generator) or share JSON. |
| Related repos | backends that share this model (Assumption: graph services; not verified here) |
| Complexity | **Medium** |
| Risk | Dual generators drift |
| Next step | One YAML → Dart and TS artifacts, or YAML → JSON consumed by both. |
| Evidence | `data/README.md`; `data/grc_model.yaml` |

| Field | Detail |
| --- | --- |
| Area | OpenAPI upgrade models |
| Current Dart | `openapi_generator_cli` + `*.g.dart` under `lib/src/upgrade_api/gen/`. |
| TS replacement | Same OpenAPI spec → TS client. |
| Related repos | graph-rpc-service OpenAPI |
| Complexity | **Low** |
| Risk | Low if spec is the source of truth |
| Next step | Point TS codegen at the same spec `make upgrade_api` uses. |
| Evidence | `pubspec.yaml` `openapi_generator_cli`; `lib/src/upgrade_api/` |

### 4.11 Feature flags and configuration

| Field | Detail |
| --- | --- |
| Area | LaunchDarkly |
| Current Dart | Workiva `launch_darkly` `flagManager`. Flags in `lib/src/feature_flags.dart` include `graph-app-pdfjs-static-file-path`, `enable-issue-management`, `grc-await-messaging-connection`, `grc-access-logging`, `ir-enable-workspace-filtering`, `grc-enable-mui-issue-section`, `grc-static-reports`, `grc-enable-sox-pending-op-alerts`, `grc-sox-export-list-enabled`, `grc-sox-export-max-count`, `grc-graph-form-connections`, `grc-update-collapse-tree-in-all-charts`, `grc-groupby-static-report-cache-enabled`, `grc-add-column-search-update`. Additional in-code: `ir-graph-ops-http`. Functional tests tag `launchDarklyEnabled`. |
| TS replacement | Prior research: `@workiva/grc-launch-darkly` + `@workiva/feature-flags`. Spreadsheet: `launchdarkly-js-client-sdk`. **Keep the same flag keys** for dual-run. |
| Related repos | `launch_darkly` |
| Complexity | **Low** |
| Risk | Default-value mismatch between Dart wrapper and JS SDK |
| Next step | Export flag key list; verify JS SDK defaults match `variationOr` fallbacks. |
| Evidence | `lib/src/feature_flags.dart`; `lib/src/_shared/graph_module.dart` |

| Field | Detail |
| --- | --- |
| Area | Environment / deploy |
| Current Dart | `wdesk_browser_environment` `Environment.current` (deploy, mockAuth, service URIs, variables). Local URL rewriting in `GraphModule` when `Deploy.localhost`. |
| TS replacement | `@workiva/environment` / `window.__wk_environment` (**spreadsheet: check**). |
| Related repos | `wdesk_browser_environment` |
| Complexity | **Medium** |
| Risk | Local `?wkdev` / `?localAuth` behavior is easy to get wrong in a TS dev server |
| Next step | POC must honor the same query params or document replacements. |
| Evidence | `lib/src/environment.dart`; `lib/src/_shared/graph_module.dart` |

### 4.12 Logging, monitoring, error handling

| Field | Detail |
| --- | --- |
| Area | Logging + App Intelligence |
| Current Dart | `package:logging` + `configureLogging()` (`?log=` query). `app_intelligence` handlers and `ContextualMessage`. OpenTracing middleware on Frugal clients (`clientTracingMessagingMiddleware`). `opentracing` in GRC services. Module tracers (`module_tracer.dart`, test phase tracer). Frustration event reporter (evidence testing) uses browser storage. |
| TS replacement | Prior research: `@workiva/grc-logger`, `@workiva/ts-grc-analytics` + `@workiva/analytics`. Keep Next Gen event names from `ANALYTICS_MIGRATION_MAPPING.md`. |
| Related repos | `app_intelligence`, `analytics`, `user_analytics` |
| Complexity | **Medium** |
| Risk | Blind spots in hybrid if TS errors bypass App Intelligence |
| Next step | Confirm TS App Intelligence / tracing packages; wire error boundary to the same pipeline. |
| Evidence | `lib/src/_shared/logging.dart`; `lib/src/sox/grc_app/grc_services.dart`; `aviary.yaml` |

| Field | Detail |
| --- | --- |
| Area | Analytics event names |
| Current Dart | Dual-send: legacy `user_analytics` + Next Gen `analytics` via `trackLegacyAnalyticEvent`. Mapping documented in `ANALYTICS_MIGRATION_MAPPING.md`. |
| TS replacement | Same **Next Gen event names** and properties (`initiated_from`, etc.). |
| Related repos | `analytics`, `user_analytics` |
| Complexity | **Medium** |
| Risk | Duplicate or dropped product metrics |
| Next step | TS must emit Next Gen names from the mapping file, not legacy snake names. |
| Evidence | `lib/src/_shared/next_gen_analytics.dart`; `ANALYTICS_MIGRATION_MAPPING.md` |

| Field | Detail |
| --- | --- |
| Area | Security scanning |
| Current Dart | `aviary.yaml` Raven monitors permissions paths and GRC header secrets used for pseudo-admin. `gha-security-scanner.yaml`. |
| TS replacement | Equivalent secret/CSP scanning on TS + dual scan during hybrid. |
| Related repos | Infosec Raven |
| Complexity | **Medium** |
| Risk | GRC header constants must not leak into a public TS bundle |
| Next step | Include Raven/TS equivalents in CI before shipping a TS island. |
| Evidence | `aviary.yaml`; `.github/workflows/gha-security-scanner.yaml` |

### 4.13 Testing

| Field | Detail |
| --- | --- |
| Area | Unit tests |
| Current Dart | `package:test`, `over_react_test`, `react_testing_library`, `mocktail`, `w_test_tools`. Generated runner `test/unit/generated_runner_test.dart`. Target 95% coverage (`test/unit/README.md`). CI: `gha-dart/test-unit` release mode, 45 min timeout. |
| TS replacement | Vitest + Testing Library (**spreadsheet: yes**). Port tests **with** each TS component. Keep Dart tests until that Dart screen is deleted. |
| Related repos | — |
| Complexity | **Medium** |
| Risk | Coverage gap during hybrid |
| Next step | Require unit tests in the same PR as each TS island. |
| Evidence | `test/`; `.github/workflows/test_and_analysis.yml` |

| Field | Detail |
| --- | --- |
| Area | Functional tests |
| Current Dart | `dart_functional` + Puppeteer + Docker compose + Skynet. Also-run against WDesk stack (`run-wdesk-also-run-test.yml`). Page objects depend on `w_table_test`, `w_comments_test`, `w_outline_test`, `wdesk_login_test`, etc. |
| TS replacement | Playwright later, **per experience**, after testids stabilize (**spreadsheet**). Do **not** port functional tests first. |
| Related repos | Skynet, WDesk, `doc_plat_client` |
| Complexity | **High** |
| Risk | Largest cost after UI. Dual-run required before cutover. |
| Next step | Keep Dart functional green. Add data-testid on any TS island. |
| Evidence | `dart_functional/README.md`; `dart_functional/pubspec.yaml`; compose YAML under `compose/` |

### 4.14 Build, deployment, CI/CD

| Field | Detail |
| --- | --- |
| Area | Build |
| Current Dart | Dart 2.19 / dart2js via `build_web_compilers`, `webdev_proxy`, `dart_dev`. `make serve`, `make gen`, `make sass`, `make lint`, `make test`. Sass checked in CI (`ddev sass --check`). `boundaries.yaml` / `ddev boundaries`. |
| TS replacement | Vite + esbuild + pnpm (**spreadsheet**). Hybrid means **both** Dart and Vite until Dart is gone. |
| Related repos | `dart_dev_workiva`, `gha-dart` |
| Complexity | **Medium**–**High** |
| Risk | Dual artifacts, slower CI, version skew |
| Next step | Add a `ts/` (or platform-standard) package without removing dart2js CDN publish. |
| Evidence | `Makefile`; `app/build.yaml`; `.github/workflows/ci.yml` |

| Field | Detail |
| --- | --- |
| Area | Deploy |
| Current Dart | CI: pub package, lockfiles, generate CDN assets from `app/`, publish CDN, Docker image `Dockerfile-wdeskapp` (WDesk app server + `cdn_assets.tar.gz`), FEWS deploy via `app/web/manifest.yaml` (`name: w_sox_app`). Helm `helm/grc-shell` is a **wk-dev test** chart, not production WDesk. |
| TS replacement | Extra CDN MFE artifact + Skynet registration (**spreadsheet**). Dual deploy until Dart retired. |
| Related repos | FEWS, Skynet, CDN, `wdesk_sdk_app_server` |
| Complexity | **High** |
| Risk | Rollback is two artifacts; FEWS still points at Dart app |
| Next step | Platform: how a TS island inside Dart CDN is versioned vs a true MFE. |
| Evidence | `.github/workflows/ci.yml`; `Dockerfile-wdeskapp`; `app/web/manifest.yaml`; `helm/grc-shell/Chart.yaml` |

### 4.15 graph_app_js and JS interop

| Field | Detail |
| --- | --- |
| Area | Existing JS bundles |
| Current Dart | Loads `packages/graph_app_js/.../commonApp.bundle.js` and `graphApp.bundle.js`. `package:js` interop for Highcharts, clipboard, form export, Gantt. `graph_app_js` is a **separate published package**, not source in this repo. |
| TS replacement | Native TS/JS; drop Dart JS interop when the host is TS. During hybrid, Dart still loads these bundles. |
| Related repos | **graph_app_js** (not in this workspace) |
| Complexity | **Medium** |
| Risk | Duplicate JS if TS Highcharts and graph_app_js both load |
| Next step | Analyze `graph_app_js` contents (Open: not in this repo). |
| Evidence | `static_asset_constants.dart`; `chart_interop.dart`; `graph_form_export_service_interop.dart` |

---

## 5. TypeScript Migration Options

Applies to **graph_app only**. Other repos may change the mix later.

### Option A — TypeScript islands inside Dart `w_sox` (recommended start)

WDesk keeps `w_sox` in pubspec. New UI is written in TS, built with
Vite, loaded from Dart via the org hybrid loader (`useTsComponent` /
`dart_ts_hmr` — **spreadsheet**; **not implemented in this repo
yet**).

- Pros: No WDesk pubspec change; incremental; rollback is a Dart
  revert plus dropping a JS asset.
- Cons: Two toolchains; Dart still owns experiences, licensing,
  Frugal; engineers must learn the bridge.

### Option B — Thin Dart facade + TS implementation of one experience

Keep `*ExperienceConfig` in Dart (route, scopes, `canUserAccessV2`,
assets). The experience body is almost all TS.

- Pros: Deep links and licensing stay on the proven Dart path.
- Cons: Still Dart for the hard platform parts; factory/deferred load
  stays Dart.

### Option C — Full TS MFE replacing `w_sox` in WDesk

Remove `w_sox` from WDesk pubspec; add a SOX package in **ts-grc**
(same pattern as `phoenix-universe` / `phoenix-assessments` — prior
research). Use `@workiva/microfrontend` +
`@workiva/drawer_experience_contribution` (and rich-experience
equivalent).

- Pros: Matches how GRC already ships TypeScript; independent FEWS
  deploy; no Dart pub dependency.
- Cons: graph_app still owns SOX testing, reports, data, planning,
  evidence testing — **none of those have a ts-grc equivalent today**
  (prior research lists ts-grc coverage for universe / assessments /
  frameworks / requests, not SOX). Also requires Audit + graph_ui
  strategy, GraphQL coverage for graph/GRC services, landing widgets,
  and embeddables.

### Option D — Standalone TS SPA

Rejected for production (**spreadsheet**). Loses WDesk nav, wtabs,
licensing, session.

**Recommendation:** Prefer **C as the end state**, using ts-grc as
the house, because Frugal-in-the-browser is an org hard blocker and
ts-grc already solved it with GraphQL for other GRC modules. Use
**A/B only** for leftover Dart experiences that cannot move until
graph_ui / Audit / `w_table` / graph NATS have a TS path. Do not
start by rewriting all of graph_app. Do not use D.

---

## 6. Challenges and Gaps

### Already listed (spreadsheet) and confirmed in this repo

| Challenge | Confirmed in graph_app? |
| --- | --- |
| No Dart→TS transpiler; rewrite | Yes — no TS app code |
| OverReact rewrite | Yes |
| WDesk cannot pub-depend on TS | Yes — README + experience configs |
| Audit compiled into SOX | Yes |
| graph_ui is a Dart library dependency | Yes |
| Frugal + NATS, not only HTTP | Yes — multiple subjects + subscribers |
| Dual Redux stores Dart vs TS | Yes, plus **Flux still in use** (spreadsheet understated Flux here) |
| `built_value` / `.sg.dart` | Yes |
| LaunchDarkly Dart wrapper | Yes |
| `w_table` plugins | Yes, extensive |
| Functional tests on Skynet/Puppeteer | Yes |
| Dual-stack deploy | Yes — CDN + Docker + FEWS + pub package |

### Gaps the spreadsheet did not emphasize for graph_app (confirmed)

1. **Flux and Redux coexist.** A "just use RTK" plan must cover
   `SoxStore` / `GraphModule`.
2. **README `SoxModule` is stale**; real entry is experience configs +
   `GraphModule`.
3. **Git overrides** of `audit` and `wdesk_sdk` mean SOX already
   tracks unpublished platform commits.
4. **Cross-experience `event_bus`** for bulk import vs testing list.
5. **GRC model YAML codegen** is Python, not `built_value`.
6. **Upgrade wizard is mixed REST + graph streams + get_it.**
7. **`graph_app_js` is an external JS package** already in the
   runtime.
8. **Nested `app/` SDK constraint still allows 2.11.**
9. **Analytics is mid-migration** (legacy + Next Gen dual-send).
10. **Helm grc-shell is wk-dev test only**, not the production WDesk
    integration path.

### Missing TypeScript libraries / platform capabilities (mix of spreadsheet + Open)

Must be confirmed on the Workiva Ecosystem Map (Open unless noted):

| Capability | Dart today | TS status |
| --- | --- | --- |
| Browser Frugal + NATS | `messaging_sdk` + `frugal` | **Org: Frugal not supported in TS.** ts-grc uses Apollo + REST. Coverage for **graph NATS + grc-services + export subscribers** is **Open** |
| `w_graph_client` live graph | Dart | Open — no ts-grc equivalent found in prior research |
| `w_table` + plugins | Dart | Open; treat as island. Org also flags `w_outline` as a TS gap |
| Unify | `unify_ui` | Prior research: `@workiva/unify` + `@mui/material` in ts-grc |
| Session | `w_session` | Prior research: `@workiva/session_mfe_service` |
| Environment | `wdesk_browser_environment` | Spreadsheet: `@workiva/environment` (check) |
| Licensing / IAM | `licensing_*` | Still **check** — SOX ability map is local to this repo |
| Feature flags | `launch_darkly` | Prior research: `@workiva/grc-launch-darkly` |
| i18n | `w_intl` / `w_translate_v2` | Prior research: `@workiva/w_intl_ts` + react-intl |
| Logging / analytics | logging / analytics | Prior research: `@workiva/grc-logger`, `@workiva/ts-grc-analytics` |
| MFE host | `wdesk_sdk` | Prior research: `@workiva/microfrontend` + drawer contribution |
| Comments / attachments / outline / viewer / drawing | Dart packages | Spreadsheet **check**; outline is an org-wide gap |
| `useTsComponent` / dart_ts_hmr | Not in this repo | Spreadsheet golden path; **not evidenced here**. ts-grc golden path is MFE, not TS-in-Dart |
| `graph_app_js` TS source | External package | Open |
| GRC YAML → TS model | Python → Dart | Missing (would be new) |
| SOX / reports / data / planning / evidence testing in ts-grc | This repo | **Missing** — prior research has no SOX MFE |

---

## 7. Risks and Blockers

| ID | Type | Item | Why it blocks or hurts |
| --- | --- | --- | --- |
| B1 | Blocker | WDesk Dart pubspec ↔ `w_sox` | Full TS package is not importable |
| B2 | Blocker | No TS Frugal; graph_app APIs still NATS | Org already forbids Frugal-in-TS. ts-grc GraphQL does **not** yet cover graph subjects, `grc-services`, or export status streams |
| B3 | Blocker | `package:audit` compiled in | SOX cannot be fully TS while Audit is a Dart library |
| B4 | Blocker | `package:graph_ui` compiled in | Same as B3 for shared graph UI |
| B5 | Blocker | Licensing ability + workspace suppression logic | Compliance if TS gates differ |
| R1 | High risk | `w_table` / smart table | Highest UI cost; last to move |
| R2 | High risk | Dual Flux + Redux + no shared Dart/TS store | State bugs in hybrid |
| R3 | High risk | Dual deploy (CDN Dart + TS JS + optional MFE) | Version skew, rollback |
| R4 | High risk | Functional test stack (Skynet + WDesk compose) | Do not port first; still must pass |
| R5 | Medium | Analytics dual-send still in progress | Easy to emit the wrong names |
| R6 | Medium | Git-pinned audit/wdesk_sdk | Platform API can move under SOX |
| R7 | Medium | Stale README (`SoxModule`) | Onboarding/migration docs could target the wrong API |
| R8 | Medium | XSS/CSP on user content (forms, TQ, markup) | Must keep React escaping + shell CSP |

**Assumptions (not blockers until disproven)**

- Platform will support TS-in-Dart hybrid (`useTsComponent`). **Not
  found in this repo.** Prior research instead treats **ts-grc MFE**
  as the golden path.
- `grc-evergreen` GraphQL can be extended to SOX/graph/export APIs
  (Assumption — not verified).
- Highcharts license covers the official npm package (spreadsheet).
- Production WDesk will remain Dart while SOX hybrid/MFE work lands.

---

## 8. Recommended Migration Approach

Safe incremental path for **graph_app**:

1. **Do not rewrite graph_app as a TS SPA.** Keep `w_sox` in WDesk
   until a SOX MFE (or set of MFEs) can replace it experience by
   experience.
2. **Follow ts-grc conventions** for new GRC TypeScript: React 18,
   `@workiva/unify`, Redux Toolkit, Apollo/RTK Query, Vite MFE
   plugin, `@workiva/session_mfe_service`, `@workiva/grc-launch-darkly`
   (prior research). Do not invent a second stack.
3. **Ask evergreen/graph backend** which SOX, graph, export, and
   dashboard operations already have GraphQL/REST. Anything still
   NATS-only stays Dart or needs a backend story before UI rewrite.
4. **If a screen must ship TS before an MFE exists**, use a Dart
   host bridge (Option A/B): session, licensing booleans, navigator,
   flags, logger. TS must not call Frugal.
5. **Migrate UI in this order (inside graph_app):**
   1. Isolated presentational panels (Support tabs, landing widget
      chrome, simple dialogs).
   2. Screens that can use REST already (upgrade wizard display,
      export list chrome).
   3. Testing list chrome (not `w_table` / graph core).
   4. Last: test form, smart table/reports, evidence testing,
      dashboards, planning Gantt, raw graph.
6. **Keep platform islands in Dart:** `w_table`, comments,
   attachments, outline, viewer, drawing/markup, spreadsheet, graph
   client streams — until those products publish TS MFEs.
7. **Keep Dart functional tests** until a given experience has TS
   parity sign-off; dual-run that experience.
8. **Do not migrate graph_ui or audit from this repo.** Analyze them
   next. SOX cannot drop Dart while it imports them.
9. **Remove `w_sox` from WDesk pubspec** only after the last
   remaining experience (including landing widgets and embeddable
   spreadsheets) has an MFE or an explicit Dart island owned by
   another package.

---

## 9. POC Recommendation

**Goal:** Validate the assumptions that would kill the plan if false,
not rewrite a flagship screen.

### POC-1 (do this first) — TS widget inside Dart WDesk/SOX

**Scope (small):** One presentational Unify panel in an existing Dart
experience with low blast radius, for example:

- a Support tab (admin-only), or
- one landing-page recency widget chrome (not the data fetch), or
- a simple confirmation dialog already using Unify.

**Must prove**

1. Dart experience still registers with WDesk (`routeSegment`,
   scopes, `canUserAccessV2` unchanged).
2. TS widget renders inside OverReact (hybrid loader).
3. Shell session is available (user id / org) **without** a TS login.
4. One LaunchDarkly flag is read from TS with the **same key** as Dart.
5. App Intelligence or logging sees a TS error (error boundary).
6. Dart unit test host still mounts; one RTL/Vitest test for the TS
   widget.
7. CDN/dev load does not break `make serve` / FEWS for that
   experience.

**Must not include:** NATS, `w_table`, Audit forms, test form save.

**Success:** Hybrid is real in this repo, not only in a spreadsheet.

### POC-2 (only after POC-1) — one API assumption

Pick **one** (in order of realism given org Frugal policy):

- **2a REST / GraphQL (preferred):** Confirm whether
  `grc-evergreen` or `graph-rpc-service` REST already exposes one
  SOX or export operation. Call it from TS with Apollo or RTK Query
  using the shell session — same pattern as ts-grc.
- **2b Existing OpenAPI:** TS calls `graph-rpc-service` REST the way
  `upgrade_api_service.dart` does.
- **2c NATS TS SDK:** only if platform reverses "Frugal not in TS".
  Unlikely; do not block the program on this.

If 2a/2b cannot cover live graph or export **streams**, the plan must
include polling (ts-grc pattern), GraphQL subscriptions, a BFF
WebSocket, or a Dart island for those streams.

**Suggested owner mix:** one graph_app UI engineer + one
evergreen/graph API engineer + WDesk/MFE as needed.

---

## 10. Open Questions

1. Is `useTsComponent` / `dart_ts_hmr` an approved path for leftover
   SOX Dart screens, or should all new TS go straight into ts-grc as
   MFEs? It is **not** in this codebase.
2. Does **grc-evergreen GraphQL** (or any REST) cover SOX testing,
   graph table/history, form export progress, dashboards, and TQ?
   Prior research solved Frugal for universe/assessments, not these
   APIs.
3. Should the first SOX TS package live in **ts-grc** (likely) or a
   new Vite tree inside graph_app?
4. Should Audit stay a Dart island inside SOX, or move to its own MFE
   first?
5. Same question for **graph_ui**.
6. Is there a TS **w_table** (or successor)? If no, reports/testing
   tables stay Dart.
7. What is inside **graph_app_js**, and is it already TypeScript?
8. Landing-page **widget** and **embeddable spreadsheet** contribution
   points for MFEs — still unverified (also listed in prior POC
   analysis).
9. Who owns GraphQL/REST for graph NATS subjects if evergreen does
   not?
10. How should GRC `grc_model.yaml` generate TS without drifting from
    Dart?
11. Nested `app/` SDK `<3.0.0` vs root `>=2.19.0` — hybrid serve
    implications?
12. Are git overrides of `audit` and `wdesk_sdk` temporary?
13. Prior research said graph_app still bundles universe/assessments/
    frameworks/requests. **This pass found they are gone from
    graph_app.** Where do they register now (WDesk vs ts-grc vs
    audit)? Confirm before planning `w_sox` removal.

---

## 11. Repository-Specific Summary

graph_app (`w_sox`) is the **GRC product UI**, delivered as a **Dart
library** that WDesk compiles and loads as experiences. It is large
(1,100+ handwritten library files), mixed Flux/Redux, and deeply tied
to **Frugal/NATS**, **graph_ui**, **audit**, **w_table**, and WDesk
`AppContext`.

A TypeScript rewrite of this repository in one step is not a
reasonable plan. Other GRC modules already moved to **ts-grc MFEs**
with GraphQL. graph_app's remaining job is the **SOX / graph / report
/ evidence** core, which is still Dart, still NATS, and still compiled
into WDesk as `w_sox`.

Safe plan: new TS follows ts-grc; Dart `w_sox` remains until each
experience has GraphQL/REST (or a Dart island) and a WDesk MFE
registration; do not call Frugal from TypeScript.

**Highest-value next analyses:** `audit`, `graph_ui`, then WDesk
experience registry (what still lists `w_sox`), then `grc-evergreen`
API coverage for SOX/graph.

**POC:** one low-risk TS UI (MFE in ts-grc **or** island in Dart,
platform-chosen) plus one REST/GraphQL call — not a NATS rewrite.

---

## 12. Cross-Repository Findings

Only **graph_app** has been analyzed in this living document so far.
The table below is what graph_app **proves about other repos**. Those
repos have not been opened in this pass.

| Other repo / package | What graph_app shows | Implication for TS | Evidence quality |
| --- | --- | --- | --- |
| **WDesk** | Consumes `w_sox` as Dart; owns login, `AppContext`, FEWS also-run, `wdesk_sdk` | Any full SOX TS cutover is a WDesk integration project | Confirmed (README, CI, registry) |
| **audit** | Compiled into SOX registry and imported from test form, landing widgets, evidence testing, upgrade module | SOX cannot be fully TS while Audit is a Dart library; git SHA override | Confirmed |
| **graph_ui** | Route paths, graph services locator, forms, Dio interceptor, solution constants | Shared library; migrating it first breaks Dart SOX/Audit | Confirmed |
| **wdesk_sdk** | Experience framework, AppContext, FEWS app server image | Shell APIs must exist for TS MFEs; currently git-overridden | Confirmed |
| **messaging_sdk / frugal** | Provider on AppContext; NATS + HTTP Frugal | Org: no Frugal in TS. ts-grc uses GraphQL; SOX/graph coverage Open | Confirmed usage |
| **ts-grc** | Not a graph_app dependency | Production TS house for other GRC MFEs; **no SOX package today** | Prior research (not opened this pass) |
| **grc-evergreen** | Not imported here | Likely GraphQL/REST target if SOX leaves NATS | Prior research |
| **w_graph_client** | GraphModule core | Live graph likely last to leave Dart | Confirmed |
| **w_table** | Smart table plugins | Last-to-migrate UI | Confirmed |
| **graph_app_js** | JS bundles loaded as static assets | Already a JS runtime dependency; source not here | Confirmed load; internals Open |
| **grc_testing_client** | Matrix experience in registry | Another Dart experience compiled in | Confirmed |
| **w_landing_page** | Hosts SOX/Audit widgets | Landing widget MFE contribution still Open | Confirmed |
| **w_viewer / drawing / markup** | Evidence testing viewers | Platform islands | Confirmed |
| **graph-rpc-service** | REST OpenAPI + RPC URLs | Possible REST beachhead (upgrade wizard) | Confirmed |
| **universe / assessments / frameworks / requests** | **Not** direct graph_app deps in this pass | Prior research said they bundled via w_sox — **stale for current pubspec** | Confirmed absence here |
| **graph_admin** | Spreadsheet / prior research: Dart MFE | Pattern for independent admin MFE | Not in this repo |
| Unify / session / LD / analytics TS | Named in ts-grc prior research | Reuse those packages; do not invent new ones | Prior research |

When audit, graph_ui, and others are reviewed, **append** new sections
(1–11 per repo) and **update this table**. Do not replace graph_app
rows.

---

## Appendix A — Review coverage (graph_app)

Reviewed before finalizing this repository's analysis:

| Category | Paths |
| --- | --- |
| Package manifests | `pubspec.yaml`, `app/pubspec.yaml`, `dart_functional/pubspec.yaml` |
| Public API | `lib/experience_configs.dart`, `lib/landing_page_widgets.dart` |
| App entry / registry | `app/lib/w_sox_app.dart`, `app/lib/src/graph_experience_registry.dart`, archive/draft registries |
| Core runtime | `lib/src/_shared/graph_module.dart`, `graph_experience_base.dart`, `environment.dart`, `feature_flags.dart`, `http_graph_api.dart`, `logging.dart` |
| Access | `lib/src/_shared/access/access.dart`, `sox_access.dart`, `sox_abilities.dart`, `ability_access.dart` |
| APIs | `grc_services.dart`, `graph_form_export_service.dart`, `job_service.dart`, `upgrade_api_service.dart`, `identity_service.dart` |
| UI assets | `static_asset_constants.dart`, experience `loadStaticAssets` |
| Data model | `data/README.md`, `data/grc_model.yaml` (generation process) |
| Tests | `test/unit/README.md`, `dart_functional/README.md` + pubspec |
| CI/CD | `.github/workflows/ci.yml`, `test_and_analysis.yml`, `run-wdesk-also-run-test.yml`, `run-functional-tests.yml`, other workflows listed |
| Deploy | `Dockerfile-wdeskapp`, `app/web/manifest.yaml`, `helm/grc-shell/Chart.yaml`, `Makefile` |
| Security | `aviary.yaml` |
| Known challenges | `Dart-TS-Migration-challenges.xlsx` sheet `Overall-list-by-AI` |
| Not in this workspace | WDesk source, audit source, graph_ui source, graph_app_js source, npm registry |

---

## Appendix B — Finding index (graph_app)

Use this when adding later repos so rows stay comparable.

| Area | Complexity | TS available (this pass) | Rewrite / wrap / reuse |
| --- | --- | --- | --- |
| WDesk pub dependency | High | No | Wrap: keep Dart `w_sox`; later MFE |
| OverReact UI | High | Yes (React) | Rewrite components |
| Unify | Medium–High | Check | Replace with Unify TS, not raw MUI |
| Flux + Redux | High | Partial | Rewrite per experience; wrap during hybrid |
| built_value models | High | Partial | Rewrite with slice |
| GRC YAML model | Medium | Missing generator | New codegen |
| Frugal/NATS | High | Check / missing | Wrap Dart API or BFF; do not HTTP-only |
| REST upgrade API | Medium | Yes (OpenAPI) | Rewrite client from same spec |
| LaunchDarkly | Low | Yes | Replace SDK, same keys |
| Session/login | Medium | Check | Reuse shell |
| Licensing | High | Check | Rewrite rules; wrap Dart first |
| w_table | High | Check | Reuse Dart island |
| Audit / graph_ui | High | No as Dart libs | Reuse as islands |
| Cache/stash | Low | Yes | Rewrite |
| browser_storage | Low | Yes | Rewrite, keep keys |
| Functional tests | High | Later | Reuse Dart/Skynet first |
| CI/CDN/FEWS | High | Partial | Dual-run |
| Analytics names | Medium | Check | Reuse Next Gen names |
