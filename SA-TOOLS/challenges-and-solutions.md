# graph_app Dart → TypeScript: challenges and solutions

**Audience:** engineering manager and tech leads  
**Repo:** graph_app (`w_sox`)  
**Date:** 26 Aug 2026  
**How to use:** each row is one decision. Read the **solution** as the
recommended path, not a commitment to rewrite SOX.

Evidence: **Confirmed** = seen in graph_app. **Prior research** = ts-grc
/ org TypeScript notes. **Open** = still needs an owner.

---

## 1. Frugal / NATS — do we migrate it, or replace it?

**Challenge**

graph_app talks to backends with **Frugal over NATS** (and some HTTP
Frugal). Live graph, GRC testing (`grc-services`), and form-export
**progress streams** all use this. Org position: **Frugal is not
supported in TypeScript.**

**Do we migrate Frugal to TypeScript?**

**No.** Do not write a TypeScript Frugal or browser NATS client for
SOX.

**Solution**

| Step | What to do |
| --- | --- |
| Keep Dart for NATS | Leave `GraphModule` / `w_graph_client` / export subscribers in Dart until a non-Frugal API exists. |
| Replace, do not port | Match **ts-grc**: Apollo **GraphQL** + RTK Query **REST** against `grc-evergreen` (or graph-rpc REST). For progress, use **polling** (ts-grc pattern) or GraphQL subscriptions — not NATS in the browser. |
| Confirm coverage | Ask evergreen/graph: which SOX, graph, export, dashboard, TQ calls already have GraphQL/REST? Anything still NATS-only stays Dart or needs a **backend** story. |
| Optional bridge | If a TS screen must ship before GraphQL exists: Dart experience calls Frugal, TS UI gets **props/JSON**. TS never imports `frugal`. |

**POC:** do **not** include NATS. POC-2 is one REST or GraphQL call
with the WDesk session.

---

## 2. Can we mix Dart and TypeScript? Can we migrate graph_app without migrating graph_ui and other Dart packages?

**Challenge**

graph_app **imports** Dart packages: `graph_ui`, `audit`, `w_table`,
`w_graph_client`, comments, attachments, viewer, and more. TypeScript
**cannot** `pub get` those packages. WDesk also cannot add an npm
package to a Dart pubspec.

**Can Dart and TypeScript run together?**

**Yes.** That is the only safe path. Other GRC modules already do this:
Dart WDesk shell + TypeScript MFEs (`ts-grc`).

**Must we migrate graph_ui (and other Dart packages) first?**

**No.** You can migrate **pieces of graph_app** while `graph_ui`,
`audit`, `w_table`, and the graph client **stay Dart**.

**Solution — three coexistence rules**

1. **Dart keeps owning the host.** WDesk still loads `w_sox`. Experience
   configs, routes, OAuth scopes, and `canUserAccessV2` stay Dart until
   that screen is a real MFE.
2. **TypeScript does not import Dart libraries.** A TS screen must not
   `import 'package:graph_ui/...'`. If the screen needs graph_ui
   (forms, graph services, route helpers), either:
   - leave that screen in Dart, or
   - keep graph_ui as a **Dart island** next to a TS panel (Dart
     renders the table/form; TS renders chrome), or
   - call a **backend/GraphQL** API instead of going through graph_ui.
3. **Migrate graph_app screens that do not need graph_ui first.**
   Support tabs, simple dialogs, landing-widget chrome. Leave data
   forms, raw graph, smart table, and test-form graph sections in Dart
   until graph_ui has a TS story or those screens talk REST/GraphQL.

**What this does *not* allow**

- Deleting `w_sox` from WDesk while Audit, graph_ui, embeddable
  spreadsheets, or landing widgets still register through graph_app.
- A “fully TypeScript graph_app” while those Dart packages are still
  required at runtime.

**Same pattern for other Dart packages**

| Package | Stay Dart? | How TS graph_app works with it |
| --- | --- | --- |
| `graph_ui` | Yes, for now | Dart island; do not migrate first |
| `audit` | Yes | Compiled into SOX today; keep as Dart island or future Audit MFE |
| `w_table` | Yes | Last to move; TS screens avoid the grid |
| `w_graph_client` | Yes | Live graph stays Dart until GraphQL |
| `w_comments`, attachments, outline, viewer, drawing | Yes | Platform islands; load from Dart experience assets |
| `unify_ui`, OverReact | Rewrite in TS | Use `@workiva/unify` + React — this *is* the TS UI |
| `launch_darkly`, `w_session` | Replace SDK, same behavior | ts-grc packages; shell still owns login |

---

## 3. WDesk still Dart — how does TypeScript load at all?

**Challenge**

Production SOX is not a website. WDesk compiles `w_sox`. An npm
package cannot go in WDesk’s `pubspec.yaml`.

**Solution**

Pick one house (manager ask A3):

- **Preferred (org / ts-grc):** register a **TypeScript MFE**. WDesk
  loads JS from the CDN at runtime. Dart `w_sox` stays in pubspec
  until that experience is cut over.
- **Fallback:** Dart experience hosts a TS widget (`useTsComponent` /
  Vite bundle). **Not in graph_app today** — confirm with platform
  before betting the POC on it.

Do **not** build a standalone SOX SPA (no WDesk). That drops login,
nav, and licensing.

---

## 4. OverReact UI — convert or rewrite?

**Challenge**

No Dart→TypeScript UI converter. Screens are OverReact.

**Solution**

**Rewrite** to React 18 function components + **`@workiva/unify`**
(same as ts-grc). Do not use raw MUI as the long-term look. POC = one
small Unify panel, not Testing or Reports.

---

## 5. State (Flux + Redux) — one shared store?

**Challenge**

graph_app uses **both** Flux (`SoxStore` / `GraphModule`) and Redux.
Dart and TypeScript **cannot** share one Redux store.

**Solution**

Do not put the store on `window`. During hybrid: TS is
**presentational** (props + callbacks from Dart). When a whole
experience is TS: **Redux Toolkit** local to that experience (ts-grc
pattern). Port store **per screen**, not globally.

---

## 6. `w_table` / smart table — need a TS grid first?

**Challenge**

Reports, test steps, and planning use `w_table` plus many plugins.

**Solution**

**Keep Dart.** Do not block the SOX TS program on a grid rewrite. First
TS screens must not include the smart table. Revisit when platform has
a TS grid (Open).

---

## 7. Audit is inside SOX — migrate Audit first?

**Challenge**

`package:audit` is compiled into graph_app. SOX cannot `pub get` a
TypeScript Audit package.

**Solution**

**No, do not migrate Audit first.** Keep Audit as a **Dart island**
inside `w_sox` (or a future Audit MFE). Analyze `audit` as its own
repo. SOX TS screens should avoid Audit widgets in the POC.

---

## 8. Login / session / licensing

**Challenge**

Login is WDesk. Access is `canUserV4` abilities plus workspace type
(SOX vs ERM vs ESG, PBC rules, etc.). Wrong gates are a compliance
issue.

**Solution**

- **Login:** do not build a TS login page. Use shell session
  (`@workiva/session_mfe_service` in ts-grc).
- **Licensing:** TS must apply the **same** ability IDs and
  suppression map. Until then, Dart `canUserAccessV2` stays the gate
  on the experience config. Any customer-facing TS screen needs
  Dart-vs-TS access tests.

---

## 9. Feature flags (LaunchDarkly)

**Challenge**

Dart wrapper around LaunchDarkly; many SOX/GRC flag keys.

**Solution**

Keep the **same flag keys**. Use ts-grc
`@workiva/grc-launch-darkly`. Dual-run Dart and TS on one flag.

---

## 10. Models / `built_value` / GRC YAML

**Challenge**

No `.sg.dart` → TypeScript converter. GRC types also come from
`data/grc_model.yaml` (Python → Dart).

**Solution**

Port models **with the screen that owns them**. Later: one YAML → Dart
and TS generators. Do not bulk-dump all models in the POC.

---

## 11. Tests (unit + Skynet functional)

**Challenge**

370 unit files; 44 Puppeteer/Skynet suites on a WDesk Docker stack.

**Solution**

- Unit tests: Vitest + Testing Library **in the same PR** as each TS
  widget. Keep Dart tests until that Dart screen is deleted.
- Functional: **do not port first.** Keep Dart/Skynet green. Add
  `data-testid` on TS. Playwright later, per experience, after
  dual-run.

---

## 12. Deploy / CI (CDN, FEWS, pub package)

**Challenge**

Today: Dart pub package + CDN + WDesk Docker + FEWS. Hybrid means two
artifacts.

**Solution**

POC rides **existing Dart deploy**. A TS MFE gets its own FEWS/CDN
pipeline (ts-grc pattern) only when that experience is real. Dual
deploy until Dart for that screen is removed. Rollback = Dart artifact.

---

## 13. i18n, analytics, logging

**Challenge**

Workiva wrappers (`w_intl`, `w_translate_v2`, dual analytics
pipelines).

**Solution**

- Strings: `@workiva/w_intl_ts` + react-intl; no hardcoded UI text.
- Analytics: **Next Gen event names** from
  `ANALYTICS_MIGRATION_MAPPING.md`.
- Logging: `@workiva/grc-logger`; TS errors must reach App
  Intelligence.

---

## 14. Comments, attachments, viewer, drawing, spreadsheet

**Challenge**

No public npm substitute. SOX loads their CSS/JS from Dart
experiences.

**Solution**

**Platform islands.** Dart experience keeps loading those assets. TS
does not replace `w_viewer` with a generic PDF viewer or the
spreadsheet with a generic grid.

---

## 15. Landing-page widgets and embeddable experiences

**Challenge**

graph_app registers landing widgets and embeddable spreadsheets.
MFE contribution for those is **unverified**.

**Solution**

Leave them on Dart `w_sox` until WDesk confirms an MFE contribution
point. Do not remove `w_sox` from WDesk until these have a new home.

---

## 16. Dart + TypeScript in the same product — how do we deploy? Do we need microservices?

**Challenge**

If the UI is mixed Dart and TypeScript, people often assume we must
split graph_app into many **microservices** (separate backend
services) or many independently deployed apps.

**Do we need everything as a microservice?**

**No.** This is a **frontend packaging** problem, not a backend
rewrite. GRC backends (graph, GRC services, licensing) already exist.
You do not create a new microservice per TypeScript widget.

Also: **microfrontend (MFE) ≠ microservice.**

| Term | What it is | Needed for Dart+TS? |
| --- | --- | --- |
| Microservice | A **backend** process (its own deploy, API, data) | **No** — not required for TS UI |
| Microfrontend (MFE) | A **frontend JS bundle** WDesk loads at runtime from CDN | **Optional** — one way to ship TS next to Dart WDesk |
| Static JS next to Dart | Vite/TS compiled into JS that Dart `loadStaticAssets` already loads | **Yes for a small POC** — graph_app already does this today for `graph_app_js`, Highcharts, `w_table` bindings |

**How graph_app deploys today (Confirmed)**

One GitHub Actions pipeline already publishes **four artifacts** from
the same repo — all still **one product**, not a microservice farm:

1. Dart **pub package** `w_sox` (what WDesk depends on)
2. **CDN** JS/CSS from `app/` (`dart2js`)
3. **Docker** WDesk app-server image (`Dockerfile-wdeskapp`)
4. **FEWS** deploy using `app/web/manifest.yaml` (`w_sox_app`)

**Solution — two frontend deploy models (pick one for the POC)**

### Model A — same repo, one WDesk package (best for first POC)

TypeScript lives in graph_app (for example `ts/`). Vite builds a JS
bundle. The Dart experience loads it the same way it already loads
`packages/graph_app_js/.../commonApp.bundle.js`.

```text
Developer merge
    → CI: dart2js (existing) + vite build (new, small)
    → JS file is included in the w_sox / CDN tarball
    → WDesk still pub-depends on w_sox only
    → One rollback: previous w_sox + CDN tag
```

- **Backend:** unchanged
- **WDesk pubspec:** unchanged
- **New services:** none
- **New microservice:** none

### Model B — TypeScript MFE next to Dart `w_sox` (later, ts-grc style)

A SOX package in **ts-grc** (or a graph_app MFE pipeline) publishes
its **own** FEWS/CDN artifact. WDesk **runtime-loads** that JS. Dart
`w_sox` still publishes as today for unmigrated screens.

```text
graph_app CI  → w_sox pub + Dart CDN  (Testing, reports, graph_ui, …)
ts-grc CI     → SOX TS MFE CDN         (the new Unify screen)
WDesk shell   → loads both at runtime
```

- Still **not** a backend microservice
- Two **frontend** artifacts (Dart + TS), independently rollback-able
- Need WDesk MFE registration (scopes, route), like universe/assessments

**What you do *not* do**

- Split SOX into one Kubernetes service per screen
- Stand up a new Frugal service “for TypeScript”
- Replace the Dart CDN pipeline on day one
- Require graph_ui / audit to become their own microservices first

**POC deploy:** Model A (JS inside existing CDN) unless platform says
the only supported path is Model B (ts-grc MFE). Either way, backends
stay as they are.

---

## 17. How do we convert graph_app if graph_ui and other Workiva UI
packages stay Dart?

**Challenge**

graph_app is not a standalone UI kit. It **imports Dart components**
from other Workiva packages and renders them on SOX screens. Confirmed
in `lib/` (1,238 Dart files):

| Dart package | Files that import it | What it is on screen |
| --- | --- | --- |
| `over_react` | 429 | graph_app’s own widgets (rewrite target) |
| `graph_ui` | 390 | Shared graph chrome: `ContentFrame`, forms, graph services |
| `web_skin_dart` | 303 | Legacy buttons, tabs, layout |
| `w_graph_client` | 243 | Live graph / NATS client |
| `unify_ui` | 209 | Unify widgets already used in Dart |
| `w_table` | 77 | Smart table / grids |
| `audit` | 71 | Audit experiences compiled into SOX |
| `w_session` | 49 | Shell session |
| `w_context_menu` | 34 | Context menus |
| `w_attachments` | 27 | Attachments panel |
| `w_landing_page_sdk` | 22 | Home widgets |
| `highcharts` | 24 | Charts |
| `w_outline` | 19 | Outline tree |
| `w_router` | 17 | Experience routing |
| `w_viewer` | 13 | Document viewer |
| `markup` / `drawing` | 11 / 1 | Markup / drawing |
| `w_comments` | 3 | Comments |
| `w_dashboard` | 3 | Dashboards |
| `embedded_spreadsheet_api` | 6 | Embeddable spreadsheet |

TypeScript **cannot** `import 'package:graph_ui/...'`. There is no
compiler that turns a Dart OverReact widget into a React component.
So “convert graph_app” cannot mean “replace `w_sox` with npm and keep
using Dart `graph_ui` widgets from TypeScript.”

**Do we wait until graph_ui (and the rest) are TypeScript?**

**No.** That would freeze SOX for years. Other GRC modules already
ship TypeScript **next to** Dart WDesk.

**Solution — convert screens, not the repo**

Treat every Workiva UI dependency as one of three buckets:

| Bucket | Packages | What you do in TypeScript |
| --- | --- | --- |
| **1. Rewrite** | `over_react`, `unify_ui`, `web_skin_dart` | New React + `@workiva/unify` widgets. This *is* converting graph_app UI. |
| **2. Dart island (keep)** | `graph_ui`, `w_table`, `audit`, `w_graph_client`, comments, attachments, outline, viewer, drawing, dashboard, spreadsheet | Dart still **renders** that widget. TS never imports it. |
| **3. Swap SDK** | `w_session`, LaunchDarkly, microfrontend | Use the **existing TS** packages (`session_mfe_service`, `grc-launch-darkly`, `@workiva/microfrontend`). Do not port Dart. |

A hybrid screen looks like this (same page, two runtimes):

```text
WDesk (Dart) still loads w_sox
  └─ Dart experience (route, OAuth, canUserAccessV2)
       ├─ TypeScript panel  → Unify chrome, dialogs, lists that
       │                      only need REST/GraphQL + session
       └─ Dart island       → graph_ui ContentFrame / form,
                              w_table grid, attachments, outline
```

**Three options when a screen needs graph_ui today**

1. **Leave the screen in Dart** (default for test form, data form,
   smart table, raw graph).
2. **Split the page:** Dart mounts `graph_ui` / `w_table` into a DOM
   node; TypeScript renders toolbar/chrome around it. Props/JSON
   only — no shared store.
3. **Stop using graph_ui on that screen:** call GraphQL/REST and
   rebuild the UI in Unify. That **duplicates** graph_ui work; only
   do it when backend coverage exists and product accepts the cost.

Option 3 is not “using graph_ui from TypeScript.” It is replacing
that screen’s dependency on graph_ui.

**What you convert first (inside graph_app)**

Safe: Support **tabs** that are mostly Unify/web_skin, landing-widget
chrome, simple dialogs, upgrade-wizard display, export-list chrome.

Not first: test form, reports smart table, data forms, evidence
tester, planning Gantt, dashboards — those are graph_ui + `w_table`
+ attachments + outline.

**What this does *not* allow**

- `import { ContentFrame } from 'graph_ui'` in a `.tsx` file
- Deleting `w_sox` from WDesk while any island still needs Dart
- Waiting for a full graph_ui TypeScript rewrite before any SOX TS

**POC:** one Unify widget with **no** `graph_ui`, `w_table`, Audit,
or NATS. That proves hybrid loading. It does not prove you can
rewrite Testing.

---

## Decision summary (for the meeting)

| # | Challenge | Migrate that layer? | Solution in one line |
| --- | --- | --- | --- |
| 1 | Frugal / NATS | **No** — replace | GraphQL/REST + polling; Dart keeps NATS until coverage exists |
| 2 | graph_ui and other Dart packages | **No, not first** | Coexist: TS screens that do not import Dart libs; Dart islands for the rest |
| 3 | WDesk Dart shell | **No** | TS MFE or Dart-hosted TS widget; keep `w_sox` until cutover |
| 4 | OverReact | **Yes, rewrite UI** | React + Unify |
| 5 | Shared Dart/TS Redux | **No** | Props first; RTK per experience |
| 6 | w_table | **No, not first** | Dart island |
| 7 | Audit | **No, not first** | Dart island / later Audit MFE |
| 8 | Login | **No** | Shell session |
| 9 | LaunchDarkly | **Replace SDK only** | Same flag keys |
| 10 | Models | **Per screen** | No bulk dump |
| 11 | Functional tests | **Not first** | Keep Skynet Dart |
| 12 | Deploy Dart+TS | **Not microservices** | Same repo: Vite JS on existing CDN (POC), or later a TS MFE artifact next to `w_sox` |

**POC that matches these solutions:** one Unify widget in WDesk (no
NATS, no graph_ui, no Audit, no `w_table`), then one REST/GraphQL
call.
