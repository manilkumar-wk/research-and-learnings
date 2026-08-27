# Key Resources widget: “Resource not found” for non-Data Admins

**Date:** 27 Aug 2026  
**Primary repo:** graph_app (`w_sox`)  
**Do not change product code** — investigation only.  
**Live reproduce:** not done (no WDesk session in this environment).

Evidence labels: **Confirmed** = this repo / pub-cache SDK / Jira+Splunk quoted in tickets. **Assumption** = not verified in graph-backend source. **Unavailable** = repo or tool missing.

---

## 1. Issue summary

When a **Reports** (or Dashboard) URL is added to the Home **Key Resources** widget, **non-Data Admin** users see **`<Resource not found>`** instead of the report/dashboard name. **Data Admin** users see the name. Removing Data Admin makes the placeholder return.

This is a **known, recurring defect**, not a missing widget-side role check in graph_app.

Master ticket: [INTRISK-72638](https://jira.atl.workiva.net/browse/INTRISK-72638) (New).  
Support original: [SUPP-65995](https://jira.atl.workiva.net/browse/SUPP-65995).  
Intended backend fix: [INTRISK-95107](https://jira.atl.workiva.net/browse/INTRISK-95107) (Closed / Done, graph-backend / w-graph) — **customers still report the same symptom in 2026** ([SUPP-77515](https://jira.atl.workiva.net/browse/SUPP-77515), [SUPP-78850](https://jira.atl.workiva.net/browse/SUPP-78850)).

---

## 2. Reproduction flow (from tickets + code)

**Confirmed UI steps** (settings content):

1. Open Home / a landing page in **Edit** mode.
2. Add or edit the **Key Resources** widget (`KeyResourcesWidgetConfig`,
   `lib/src/landing_page_widgets/key_resources/key_resources_widget_config.dart`).
3. Paste a Reports URL in **Resource URL** (`KeyResourcesSettingContent`).
   Typical shape (**Confirmed** by tests):
   `https://<host>/a/<workspaceId>/report/<dataSourceOrReportViewId>[/<reportViewId>][?flags]`
4. Click add (`AddKeyResourceAction`).
5. Save the page (widget settings persist via landing-page consumer
   settings).
6. View the widget.

**Data Admin:** display name is the report/dashboard title.  
**Non-Data Admin:** list text is `<Resource not found>`
(`WSoxIntl.resourceNotFoundInBrackets`). Users can still **open** the
report in Reports experience (SUPP-77515).

**Toggle Data Admin** (SUPP-65995, **Confirmed** in ticket): name appears
when the role is added and disappears when it is removed — so the label
is **resolved at load time**, not baked in as a string at save.

---

## 3. Is “Resource not found” saved or calculated?

**Calculated every time the widget enriches names. Not stored as the label.**

`KeyResource.toJson` persists **only** `resourceWurl`:

```29:32:lib/src/landing_page_widgets/key_resources/models/key_resources_widget_state.sg.dart
  Map<String, dynamic> toJson() => {'resourceWurl': resourceWurl.toString()};

  factory KeyResource.fromJson(Map<String, dynamic> json) {
    return KeyResource((b) => b..resourceWurl = Wurl.parse(json['resourceWurl'] as String));
```

Save path: `KeyResourcesWidgetMiddleware._keyResourcesUpdated` JSON-encodes
the list into landing `consumerSettings` key `keyResources`. Display name
is dropped.

Load path: `GetInitialResourcesAction` → parse wurls →
`enrichKeyResources` → RIS again.

If RIS fails, `_getResourceName` **locally** sets the string
`<Resource not found>`. The backend does not return that English phrase
(**Confirmed** in graph_app). Splunk shows empty URI / uninitialized
contentType instead.

The list also shows the same string if `displayName` is empty
(`key_resources_widget_list.dart`).

---

## 4. Frontend call flow (graph_app)

### 4.1 Components and state

| Piece | Path | Role |
| --- | --- | --- |
| Config | `key_resources_widget_config.dart` | Landing widget registration |
| Module | `key_resources_widget_module.dart` | RIS SDK, GraphModule, ContentService, Redux store |
| Client | `key_resources_widget_client.dart` | URL→Wurl, graph query, RIS enrich, VS read |
| Middleware | `redux/key_resource_widget_middleware.dart` | Add / load / persist |
| Settings UI | `components/key_resources_settings_content.dart` | Paste URL, dispatch add |
| List UI | `components/key_resources_widget_list.dart` | Renders name or placeholder |
| View | `components/key_resources_widget_view.dart` | Read-only list + navigate |
| Intl | `lib/src/intl/w_sox_intl.dart` | `resourceNotFoundInBrackets` → `'<Resource not found>'` |

Redux: `KeyResourcesWidgetState` + actions in
`redux/key_resource_widget_actions.dart`. **No Data Admin / licensing
check in this widget** (**Confirmed**).

### 4.2 Add a Reports URL (end to end)

```text
KeyResourcesSettingContent onClick
  → dispatch AddKeyResourceAction(url)
  → KeyResourcesWidgetMiddleware._addKeyResource
       getResourceKindFromUrl(url)  // segment after workspace id
       if kind == GraphRoutePaths.report:
         getUrlWithReportViewId(url)   // graph typedQueryOnce
       convertURLtoWurl(resourceUrl)
       if wurl == null → SetErrorAddingResourceMessageAction(invalid URL)
       else addKeyResource(wurl) → enrichKeyResources (RIS)
       KeyResourcesUpdatedAction
       → events.onWidgetSettingsUpdate({ keyResources: json })
```

**URL parse** (`convertURLtoWurl` / `_isValidUrl`):

- Split on `/` and `?`.
- Workspace id must match `session.context.accountResourceId`.
- Reports need **two** ids after `report` (DataSource + ReportView), or
  `getUrlWithReportViewId` inserts the missing one.
- Graphdev URLs add +2 to indexes.

**IDs extracted:**

- `urlSegments[5 + graphDev]` = DataSource **or** ReportView vertex id
- `urlSegments[6 + graphDev]` = the other id after the graph query
- Wurl (**Confirmed** `graph_ui` `buildWurl` + extra `Segment('rv', reportViewId)`):

  `wurl://graph.v1/view:report/report:<dataSourceId>/rv:<reportViewId>`

  Dashboard (no report-view query):

  `wurl://graph.v1/view:d/dashboard:<id>`

**Report-only graph call** (`getUrlWithReportViewId`):

- `GraphClientModule.api.typedQueryOnce(_reportIdQuery(vertexId))`
- Structured query: optional reverse `source` to `ReportView` (`viewOf != true`)
  and optional `source` to `DataSource`
- If trees empty or module null → **returns null** → UI shows
  **invalid URL**, not Resource not found
- So a user who **sees Resource not found** already got a valid Wurl
  (**Confirmed** by control flow)

**Name resolution** (`_enrichKeyResourcesFromRIS`):

- `ResourceInfoSdk.getResourceDataV2(revisions, [NAME, URL], PLAIN)`
- NATS subject **`resource-information`** (linking_sdk
  `ResourceInfoSdk.serviceName`)
- Scope `linking|r`
- Created in `appInitializer` via
  `createResourceInfoSdkWithFMP(appContext.frugalMessagingProvider, session)`
  with **`requestNameDataKeyAuthz` left default `false`**

**`_getResourceName`:** any `data.err` (including non-`RIError`) →
`WSoxIntl.resourceNotFoundInBrackets`. Unit test
`test/unit/common/landing_page_widgets/key_resources_client_test.dart`
(`enrichKeyResources handles errors correctly`) **Confirms** this.

**Permission on frontend:** none for Data Admin. Graph module load
requires licensing **GRAPH** ability (`LicensingFrugalV4AbilityModuleConstants.GRAPH`)
in `appInitializer` — that gates `typedQueryOnce`, not RIS.

### 4.3 Load / display

`GetInitialResourcesAction` → consumer settings **or** legacy
`ViewSettingsClient.getEntries` (NATS `view-settings-service`) →
`enrichKeyResources` again.

Display: empty or error name → `<Resource not found>`.

---

## 5. Backend call flow (how owners were identified)

graph_app does **not** contain the graph permission service. Owners
were identified from:

1. Dart SDKs in pub-cache (`linking_sdk`, `graph_ui` ViewSettingsClient,
   `w_graph_client`)
2. Jira Splunk excerpts (SUPP-65995, SUPP-77515)
3. INTRISK-95107 description (w-graph / graph-backend)

**Could not clone** `w-graph` / permission-service / linking-directory
(GitHub MCP unavailable; not in this workspace).

| Call | Transport | Service | What it does |
| --- | --- | --- | --- |
| `typedQueryOnce` | Graph client / NATS graph | **graph / w-graph** | Resolve ReportView ↔ DataSource ids |
| `getResourceDataV2` | Frugal NATS `resource-information` | **linking-directory** fans out to **graph StreamsResourceInfoProvider** | NAME + URL for the Wurl |
| Provider `getResourceData` | Frugal HTTP `/resourceProvider` | **graph-server** `StreamsResourceInfoProvider` | Permission + vertex lookup |
| `getPermission` | (internal) | **permission service** | `resourceType: ReportResource` or `DashboardResource` |
| Landing `onWidgetSettingsUpdate` | Home / landing SDK | **home / w_landing_page** | Persist wurls (not names) |
| Optional VS `get/set` | NATS `view-settings-service` | **view-settings** | Legacy key `graph.keyResources*` |

Splunk (**Confirmed** quoted on tickets):

- Message: `Empty URI found in an outgoing ResourceData`
- `frugalEndpoint`: `...FResourceInformationProviderService...getResourceData`
- `clientId`: `linking-directory`
- `Processor`: `StreamsResourceInfoProvider`
- `getPermission.resourceType`: **`DashboardResource`** or **`ReportResource`**
- `skipPermissions: false`, `isSuperAdmin: false`, `hasAdminRequestHeader: false`
- `access_level: regular`
- `contentType: unknown/uninitialized`
- Resource Wurl matches frontend `buildWurl` format

So the backend **does not return the string** “Resource not found”. It
returns **empty / uninitialized ResourceData** (and/or an error on NAME)
when permission evaluation fails. graph_app maps that to the placeholder.

Data Admin vs not: Data Admin can resolve NAME/URL; non-admin gets empty
URI. INTRISK-95107: Data Admin can see it; non-admin should be allowed if
they have a **`reads`** edge from the **DataSource** to an **AccessRole**
reachable by the user (reports), or dashboard `$readsDashboard` /
`$writesDashboard` / `$ownsDashboard` edges. Until that intent is used
correctly, **generic Resource intent** is wrong for these Wurls.

---

## 6. Permission investigation

| Question | Finding |
| --- | --- |
| Check in graph_app Key Resources? | **None** for Data Admin. |
| What fails? | Graph **permission service** for `ReportResource` / `DashboardResource` while resolving RIS NAME/URL. |
| Report / dashboard / folder / workspace / data? | **Report or dashboard vertex ACL / AccessRole edges**, not folder/workspace IAM in this path. Users often **can** open the report UI (SUPP-77515) — so **report view permission ≠ RIS name permission** today. |
| IAM action | Licensing **GRAPH_DB_ADMIN** (`isDBAdmin` in `lib/src/_shared/utils/is_dbadmin.dart`) is the product “Data Admin” used elsewhere; Splunk uses `isSuperAdmin` / abilities list including `FULL_GRAPH_READ`. Exact mapping in permission-service **Assumption** without that repo. |
| 404 vs not found string | Backend: empty URI / uninitialized. Frontend: always `<Resource not found>` for **any** RIS NAME error (does not distinguish `FORBIDDEN` vs missing). graph_ui `link_properties_provider.dart` **does** special-case `ResourceInformationErrorsConstants.FORBIDDEN`. Key Resources does not. |
| Wrong access model? | **Yes (product defect).** Widget uses **current user’s** RIS lookup. It should succeed if the user can **read the report/dashboard**, not only if they are Data Admin. INTRISK-95107 documents the correct intents. |
| Expected behavior? | **No.** Workaround in SUPP-65995: grant Data Admin — that is a **workaround**, not intended product behavior. |

---

## 7. Related Jira (compared to this code path)

| Key | Title | Status / resolution | Same problem? |
| --- | --- | --- | --- |
| **INTRISK-72638** | `<Resource not found>` instead of dashboard name for non-Data Admins | New | **Yes — master.** Same RIS + Data Admin toggle. 403 noted in intake. |
| **SUPP-65995** | same | Addressed / Transferred → 72638 | **Yes.** Splunk `DashboardResource` + empty URI. |
| **INTRISK-95107** | Fix permission service handling of ReportResource and DashboardResource | Closed / Done (graph-backend, w-graph) | **Same root cause (backend).** Intents in `PermissionUtil.getIntentForResource()`. Frontend not required to change. |
| **SUPP-77515** | Report shows as resource not found in widget | Addressed / Duplicate of 72638 | **Yes.** Reports Wurl + `ReportResource` + empty URI May 2026. |
| **SUPP-78850** | Non-admins see error in Home Dashboard widgets | Needs Information | **Yes** (support confirmed 72638), Aug 2026. |
| **INTRISK-93944** / **SUPP-74034** | Assigned Dashboard Key Resources / shared reports | 93944 New; SUPP Duplicate | **Related.** Shared-report viewers vs Business Controls group — same placeholder, extra **sharing** angle. |
| **INTRISK-93521** | Failed to resolve resource / getResourceDataV2 | Closed / Addressed → 95107 | **Same backend API.** |
| **INTRISK-86141** | resource not found for **folders** (pagination) | Closed / Done | **No.** CM `getFiles` cursor, not RIS graph Wurl. |
| SUPP-74707 / 74996 / 74601 | boards / home widgets | Duplicate of 95107 | Same class of RIS permission bug. |
| INTRISK-92586 / 51847 | unable to **add** resources | Done | **Different** (add/invalid URL), not this display path unless they hit null Wurl. |

Do **not** treat 86141 (folders) as this bug.

---

## 8. Root cause (code-backed)

1. **Frontend (Confirmed):** Key Resources never stores names. It asks
   **Resource Information** for NAME/URL. **Any** RIS NAME error becomes
   `<Resource not found>` (`_getResourceName`).
2. **Backend (Confirmed via Splunk on tickets, not via graph source):**
   Graph `StreamsResourceInfoProvider.getResourceData` evaluates
   `ReportResource` / `DashboardResource`. For non-Data Admins it
   emits **Empty URI / uninitialized** ResourceData. Data Admins succeed.
3. **Mismatch (Confirmed product + INTRISK-95107):** Those Wurls were
   treated as generic **Resource** intent instead of report/dashboard
   AccessRole / `$readsDashboard` logic. Data Admin bypasses that.
4. **Why it can still happen after 95107 Closed:** **Assumption** —
   incomplete rollout, Wurl format gaps, or **regression**. SUPP-78850
   (Aug 2026) still matches 72638.

**Not the cause:** saving the literal string; a graph_app `if (isDataAdmin)`
branch (there is none); folder pagination (86141).

---

## 9. Recommended fix (do not implement in this pass)

**Primary (backend, already specified on 95107):**  
`PermissionUtil.getIntentForResource()` must map report/dashboard Wurls
to the documented intents so a user with `reads` / `$readsDashboard` gets
NAME+URL without GRAPH_DB_ADMIN.

**Verify 95107 in the environment that still fails** (commit, deploy
train, Wurl formats including `/report/<id>` without `rv`).

**Frontend (optional, graph_app):**

- If `RIError.code == FORBIDDEN`, show a **permission** message, not
  “not found” (pattern already in graph_ui link properties).
- Do **not** persist a guessed name as a security substitute without
  product review (stale names vs leaking titles).

**Workaround:** Data Admin role (SUPP-65995) — **security tradeoff**.

---

## 10. Security and regression risks

- Granting Data Admin as workaround **over-privileges** (FULL_GRAPH_READ
  and related abilities appear on Splunk for some users).
- Caching/storing display names could leak titles to users who later
  lose access — re-resolve on load is correct **if** RIS permissions
  match report read.
- Changing intents can **over-expose** names if AccessRole edges are
  wrong; test both deny and allow.
- `requestNameDataKeyAuthz` is **false** in Key Resources; turning it on
  without backend alignment could worsen denials (**Assumption**).

---

## 11. Tests and QA

**Unit (graph_app):**

- RIS NAME `err` with `FORBIDDEN` vs missing — assert copy (today both
  are `<Resource not found>`).
- Report URL with only DataSource id: mock `typedQueryOnce` empty →
  invalid URL, not placeholder.
- `KeyResource.toJson` still wurls-only.

**Integration / Skynet:**

- Non-Data Admin with report **viewer** + `reads` edge: widget shows
  title.
- Same user without report access: no title (or explicit denied).
- Data Admin: title.
- Toggle Data Admin without re-adding the URL: name appears/disappears
  (proves dynamic resolve).
- Dashboard URL and report URL with `/rv:` segment.

**QA:** Home edit → paste report URL from address bar → save → view as
Process/Controls Owner vs Data Admin; confirm report experience still
opens.

---

## 12. What could not be verified

| Item | Why | Needed |
| --- | --- | --- |
| Live click-through | No WDesk in this session | Staging workspace + two users |
| `PermissionUtil.getIntentForResource()` source | graph-backend / w-graph not in workspace; GitHub MCP down | Clone w-graph; confirm 95107 commit is on the failing env |
| Exact IAM action IDs | Only licensing constants in graph_app | permission-service + Splunk `getPermission` payload |
| Whether 95107 fully shipped | Ticket Done but 2026 SUPPs remain | Deploy/version of graph-server vs INTRISK-95107 |

---

## 13. Remaining questions

1. Is INTRISK-72638 still open because 95107 did not cover Home widgets /
   all Wurl shapes?
2. Should Key Resources use `requestNameDataKeyAuthz: true`?
3. For shared reports (INTRISK-93944), is the missing edge `reads` to
   AccessRole for the **viewer group**, not Data Admin?
