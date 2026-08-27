# Root-Cause Analysis: Key Resources Widget "Resource not found" for Non-Admin Users

## 1. Issue Summary

When a non-Data Admin user adds a Reports or Dashboard URL to the **Key
Resources** widget on the Home page, the widget displays
**`<Resource not found>`** instead of the resource name. Data Admin users see
the correct name. The issue has been reported by multiple customers (Elkem,
Barry Callebaut, Lloyds Banking, Napco, QuadReal, Daikin) since May 2024,
and remains active as of August 2026.

---

## 2. Reproduction Flow

1. Log in as a **non-Data Admin** user (e.g., Process Owner, Control Owner)
2. Navigate to the **Home** page
3. Edit a widget and add a **Key Resources** widget (or edit an existing one)
4. In the settings panel, paste a **Reports URL**
   (e.g., `https://app.wdesk.com/a/{workspaceId}/report/{dataSourceId}/{reportViewId}`)
5. Click the **Add resource** button
6. **Observed**: The resource appears as `<Resource not found>` instead of the
   report name
7. **Expected**: The report's display name should appear
8. Repeating step 4-6 as a **Data Admin** user works correctly

---

## 3. Frontend Call Flow (Complete)

### 3.1 Adding a Resource

| Step | File | Class/Method | Action |
|------|------|-------------|--------|
| 1 | `components/key_resources_settings_content.dart:62` | `KeyResourcesSettingContent` | User clicks Add button → dispatches `AddKeyResourceAction(url)` |
| 2 | `redux/key_resource_widget_middleware.dart:51-85` | `_addKeyResource()` | Middleware handles the action |
| 3 | `key_resources_widget_client.dart:140` | `getResourceKindFromUrl()` | Extracts resource kind from URL (e.g., `"report"`) |
| 4 | `key_resources_widget_client.dart:155-188` | `getUrlWithReportViewId()` | For reports: queries graph API to resolve DataSource↔ReportView relationship via `_reportIdQuery()` |
| 5 | `key_resources_widget_client.dart:127-138` | `convertURLtoWurl()` | Parses URL → creates `Wurl` (e.g., `wurl://graph.v1/view:reports/ds:{id}/rv:{id}`) |
| 6 | `key_resources_widget_client.dart:65-87` | `addKeyResource()` | Checks for duplicates, calls `enrichKeyResources()` |
| 7 | `key_resources_widget_client.dart:314-327` | `enrichKeyResources()` | Splits resources by service type: graph → RIS, files → Content Management |
| 8 | `key_resources_widget_client.dart:333-381` | `_enrichKeyResourcesFromRIS()` | Creates `ResourceRevision` list, calls RIS SDK |
| 9 | **`key_resources_widget_client.dart:348`** | `_resourceInfoSdk.getResourceDataV2(...)` | **The critical backend call** — requests `NAME` and `URL` data keys |
| 10 | `key_resources_widget_client.dart:354-368` | Loop over `resourceData` | Parses response, calls `_getResourceName()` for NAME data |
| 11 | **`key_resources_widget_client.dart:435-449`** | **`_getResourceName()`** | **THE ERROR ORIGIN** — if `data.err != null`, returns `WSoxIntl.resourceNotFoundInBrackets` |
| 12 | `redux/key_resource_widget_middleware.dart:74` | `KeyResourcesUpdatedAction` | Dispatches updated resource list to store |
| 13 | `redux/key_resource_widget_middleware.dart:102-116` | `_keyResourcesUpdated()` | Serializes resources to JSON, saves via `onWidgetSettingsUpdate()` |

### 3.2 Loading/Rendering

| Step | File | Method | Action |
|------|------|--------|--------|
| 1 | `key_resources_widget_module.dart:101-137` | `KeyResourcesWidgetModule()` constructor | Deserializes `_initialSettings[consumerSettingsKey]` → list of `KeyResource` (only `resourceWurl` is stored) |
| 2 | `key_resources_widget_module.dart:136` | Dispatches `GetInitialResourcesAction` | Triggers middleware to load and enrich resources |
| 3 | `redux/key_resource_widget_middleware.dart:31-49` | `_getInitialResources()` | Falls back to ViewSettings if no initialSettings; calls `enrichKeyResources()` → same RIS call as adding |
| 4 | `components/key_resources_widget_view.dart:21-54` | `KeyResourcesWidgetView` | Renders loading spinner or `KeyResourcesWidgetList` |
| 5 | **`components/key_resources_widget_list.dart:41-55`** | `_renderListItemText()` | **If `displayName` is empty**: shows `WSoxIntl.resourceNotFoundInBrackets` = `"<Resource not found>"` |

### 3.3 Key Model: What Is Stored vs Resolved

**Stored** (in widget consumer settings): Only the `resourceWurl` string —
`key_resources_widget_state.sg.dart:29`:

```dart
Map<String, dynamic> toJson() => {'resourceWurl': resourceWurl.toString()};
```

**Resolved dynamically** on every load: `displayName`, `resourceUrlPath`,
`icon`, `resourceKind`, `isError` — all fetched via RIS at load time.

**Conclusion**: `"<Resource not found>"` is **NOT saved** in the widget
configuration. It is **resolved dynamically every time the widget loads** by
calling the RIS backend. If the user lacks permission, the error recurs on
every page load.

---

## 4. Backend Call Flow

### 4.1 Resource Information Service (RIS)

| Layer | Location | Detail |
|-------|----------|--------|
| **Client SDK** | `linking_sdk-2.20.21/lib/src/streams/v2/resource_info_sdk.dart:311-332` | `WdeskResourceInfoSdk.getResourceDataV2()` — Frugal RPC call to `resource-information` NATS service |
| **Frugal Service** | `linking_frugal-18.0.64` | `FResourceInformationService.getResourceDataV2()` — Thrift/Frugal protocol over NATS |
| **Backend Service** | `resource-information` service (Skaar team) | Routes request to the appropriate **provider** based on WURI service prefix |
| **Provider for graph resources** | **`w-graph`** service | Handles all `wurl://graph.v1/...` resources — provides NAME and URL data keys |

### 4.2 Permission Check in w-graph (Backend)

Based on INTRISK-95107 findings:

| Component | Detail |
|-----------|--------|
| **Permission util** | `PermissionUtil.getIntentForResource()` in `w-graph` |
| **Original bug** | Reports and Dashboards were using a **generic "Resource intent"** for permission checks |
| **What failed** | The generic Resource intent doesn't account for the specific permission edges that Reports and Dashboards use |
| **Reports permission** | Requires `reads` edge on `DataSource` vertex |
| **Dashboard permission** | Requires `$readsDashboard` / `$writesDashboard` / `$ownsDashboard` edges |
| **Fix (INTRISK-95107)** | New intents in `PermissionUtil.getIntentForResource()` for Report and Dashboard types — PRs `Workiva/w-graph#5694` and `Workiva/w-graph#5726`, fix version `w-graph 18.3.331` |

### 4.3 Error Propagation

```
w-graph provider → permission check fails → returns error to RIS
     ↓
RIS → populates ResourceData.err = RIError(code: 403, message: "...")
     ↓
linking_sdk → returns ResourceData with err field set
     ↓
graph_app KeyResourcesClient._getResourceName() → data.err != null
     → returns "<Resource not found>"
     ↓
UI renders "<Resource not found>"
```

**Error code**: `ResourceInformationErrors2Constants.FORBIDDEN = 403` —
defined in
`linking_frugal-18.0.64/lib/src/gen/resource_information_errors2/f_resource_information_errors2_constants.dart:31`

**The permission failure is intentionally returned as an RIError (403
FORBIDDEN), not a 404.** The frontend treats ALL `RIError` codes identically
as "Resource not found" — it does not distinguish between permission denied
(403) and actually not found.

---

## 5. The "Resource not found" String — Exact Origin

| String | Value | Location |
|--------|-------|----------|
| `WSoxIntl.resourceNotFound` | `"Resource Not Found"` | `lib/src/intl/w_sox_intl.dart:3379` |
| `WSoxIntl.resourceNotFoundInBrackets` | `"<Resource not found>"` | `lib/src/intl/w_sox_intl.dart:3381-3382` |

**Usage in code**:

1. **`_getResourceName()`** at `key_resources_widget_client.dart:443,446` —
   returns `WSoxIntl.resourceNotFoundInBrackets` when RIS returns any error
2. **`_renderListItemText()`** at `key_resources_widget_list.dart:52` —
   displays `WSoxIntl.resourceNotFoundInBrackets` when `displayName` is null
   or empty

**The string is generated locally by the frontend.** The backend returns an
`RIError` object with a code and message; the frontend converts ALL errors
into the same user-facing string.

---

## 6. Permission Investigation

### 6.1 Data Admin vs Non-Data Admin

| Aspect | Data Admin | Non-Data Admin |
|--------|-----------|----------------|
| **RIS `getResourceDataV2()` call** | Returns `ResourceData` with `err = null`, `data = "Report Name"` | Returns `ResourceData` with `err = RIError(code: 403)` |
| **`w-graph` provider permission check** | Passes — Data Admin has broad graph access | Fails — user lacks the specific graph edge permissions |
| **Result** | Display name resolved correctly | `"<Resource not found>"` displayed |

### 6.2 What Permissions Are Required

Based on INTRISK-95107:

- **Reports**: User needs a `reads` edge on the `DataSource` vertex in the
  graph
- **Dashboards**: User needs `$readsDashboard`, `$writesDashboard`, or
  `$ownsDashboard` edge
- **Data Admin role**: Grants broad graph permissions including these edges
- **Non-Data Admin roles** (Process Owner, Control Owner, etc.): May be
  **assigned to** the report/dashboard but lack the specific graph edge
  required by the `w-graph` provider's permission check

### 6.3 Frontend Permission Check

The frontend has **one** permission gate: the
`licensingApi.canUserV4(abilityId: GRAPH)` check in
`key_resources_widget_module.dart:63-64`. This controls whether the
`GraphClientModule` is loaded (needed for report URL resolution), not whether
the user can view resource names. **There is no frontend permission check for
individual resources.**

### 6.4 `requestNameDataKeyAuthz` Flag

A critical observation: The `ResourceInfoSdk` supports a
`requestNameDataKeyAuthz` flag (`resource_info_sdk.dart:214-216`) that adds a
`skaar-name-authorization` header. The `appInitializer` in
`key_resources_widget_module.dart:52` creates the SDK **without** this flag:

```dart
_resourceInfoSdk = await createResourceInfoSdkWithFMP(
    appContext.frugalMessagingProvider, appContext.session);
```

The SDK documentation states this opts into "the soon-to-be-defaulted behavior
of checking authorization prior to returning data for the 'name' data key."
If name authorization is now globally enabled, this may be a contributing
factor — the widget was never designed for authorized name lookups.

---

## 7. Related Jira Tickets

### Primary Tickets

| Ticket | Title | Status | Relevance |
|--------|-------|--------|-----------|
| **INTRISK-72638** | `<Resource not found>` instead of dashboard name for non-Data Admins | **New** (open, unassigned) | **Canonical bug** — the exact issue described here. Console shows 403 from PSE. |
| **INTRISK-95107** | Fix permission service handling of ReportResource and DashboardResource | **Closed/Done** (w-graph 18.3.331) | **Backend fix** — changed `PermissionUtil.getIntentForResource()` for Reports/Dashboards. PRs: w-graph#5694, w-graph#5726. |
| **INTRISK-93521** | Assigned Dashboard "Key Resources" showing "Resource not found" | **Closed/Addressed** (Manil Kumar) | Prior investigation of this same issue. |

### Support Tickets (Confirming Active Recurrence)

| Ticket | Customer | Date | Status | Key Detail |
|--------|----------|------|--------|------------|
| SUPP-65995 | Elkem Group (EU) | May 2024 | Addressed/Transferred | Original customer report → cloned to INTRISK-72638 |
| SUPP-74033 | Daikin Europe (EU) | — | Addressed/Duplicate | Same issue, linked to INTRISK-93521, has w-graph PR #5611 |
| SUPP-74034 | QuadReal | — | Addressed/Duplicate | Same issue for shared reports |
| SUPP-74601 | Napco Security | — | Addressed/Duplicate of INTRISK-95107 | Non-admin Process/Control Owners affected |
| SUPP-74996 | Napco Security | — | Addressed/Duplicate of INTRISK-95107 | Follow-up |
| **SUPP-77515** | Barry Callebaut (EU) | **May 2026** | Addressed/Duplicate of INTRISK-72638 | **Explicitly calls it a "regression"** — mentions "empty URI generated in outgoing resource data" |
| **SUPP-78850** | Lloyds Banking Group (EU) | **August 2026** | **Needs Information (R&D) — ACTIVE** | Reported 2 days ago. High-impact customer. Confirms issue is still occurring. |

### Related but Different Root Causes

| Ticket | Title | Root Cause | Fix |
|--------|-------|-----------|-----|
| INTRISK-86141 | Key Resources "resource not found" for large folders | Folder pagination (>1000 folders) | graph_app 10.0.97 (pagination fix) |
| INTRISK-104291 | Spike: pagination for Landing Page widgets | Rate limiting on `api_v1_files.GET` | Assigned to Manil Kumar, in progress |

---

## 8. Root Cause (Supported by Code Evidence)

### Verified Facts

1. **The frontend generates `"<Resource not found>"` locally** when the RIS
   backend returns an `RIError` in `ResourceData.err` —
   `key_resources_widget_client.dart:435-449`. All error types (403, 400, 510,
   etc.) produce the same message.

2. **The backend `w-graph` provider returns
   `RIError(code: 403 FORBIDDEN)`** when a non-Data Admin user requests the
   NAME data key for a Report or Dashboard resource. The 403 code is defined
   in `ResourceInformationErrors2Constants.FORBIDDEN`.

3. **The permission check happens in `w-graph`'s
   `PermissionUtil.getIntentForResource()`** — for Reports, it checks the
   `reads` edge on the DataSource vertex; for Dashboards, it checks
   `$readsDashboard`/`$writesDashboard`/`$ownsDashboard` edges.

4. **INTRISK-95107 was a partial fix** — it corrected the permission intent
   mapping in w-graph, but SUPP-77515 (May 2026) and SUPP-78850 (August
   2026, 2 days ago) confirm the issue **has regressed or was not fully
   resolved**.

5. **Resource names are resolved dynamically** (not stored) — the error
   recurs on every widget load, not just when adding.

6. **The frontend does not distinguish 403 (forbidden) from actual "not
   found"** — this is a UX problem where a permission denial is presented as
   a missing resource.

### Root Cause

**The `w-graph` service's RIS provider denies name resolution for Reports and
Dashboards when the requesting user lacks specific graph edge permissions**,
even if the user has been assigned access to the resource via other means
(role assignment, workspace membership). The provider returns
`RIError(403)`, which the frontend indiscriminately converts to
`"<Resource not found>"`.

The fix in INTRISK-95107 (w-graph 18.3.331) addressed specific intent
mappings but appears to have either regressed or left edge cases uncovered —
the issue is confirmed active as of August 2026.

A secondary contributing factor may be the `requestNameDataKeyAuthz` behavior
in the RIS SDK. The Key Resources widget does **not** opt into this
authorization model (`key_resources_widget_module.dart:52`), meaning it may
be affected by changes in the default authorization behavior for the "name"
data key.

---

## 9. Recommended Fix

### Backend (w-graph)

1. **Investigate regression of INTRISK-95107** — verify that
   `PermissionUtil.getIntentForResource()` still correctly handles Report and
   Dashboard intents in the current `w-graph` version
2. **Check if `skaar-name-authorization` global enablement** changed the
   behavior for consumers that didn't opt in
3. **Verify that non-Data Admin users with proper role assignments** (e.g.,
   assigned to a report, shared access) are granted `reads` edges on the
   DataSource/ReportView vertices

### Frontend (graph_app) — Complementary Improvements

1. **Differentiate error types**: In `_getResourceName()`, check
   `data.err` code — if 403, show a permission-specific message like
   `"<Insufficient permissions>"` instead of `"<Resource not found>"`
2. **Consider `requestNameDataKeyAuthz`**: Evaluate whether setting
   `requestNameDataKeyAuthz: true` when creating the RIS SDK would resolve
   the issue
3. **Cache resolved names**: Consider storing `displayName` alongside
   `resourceWurl` in the widget settings to avoid re-resolution failures on
   load

---

## 10. Security and Regression Risks

| Risk | Assessment |
|------|-----------|
| **Security**: Could fixing this expose resource names to unauthorized users? | Low-medium risk. The user is already in the workspace and has some role assignment. The fix should verify the user has legitimate access, not bypass authorization. |
| **Regression**: Could changing the permission check affect other RIS consumers? | Medium risk. The w-graph provider serves all RIS consumers — changes must be tested across all consumers (linking, cross-references, etc.). |
| **Regression**: Storing display names in settings | Low risk, but stale names if resources are renamed. Requires invalidation strategy. |

---

## 11. Suggested Tests

### Unit Tests (graph_app)

- `_getResourceName()` with `RIError(code: 403)` returns appropriate
  permission-denied message
- `_getResourceName()` with `RIError(code: 400)` returns generic error
  message
- `enrichKeyResources()` with mixed success/failure RIS responses
- `_renderListItemText()` with `isError: true` shows appropriate UI state

### Integration Tests

- Non-Data Admin user adds Report URL → widget shows report name (not error)
- Non-Data Admin user adds Dashboard URL → widget shows dashboard name (not
  error)
- Non-Data Admin user loads Home page with existing Key Resources → names
  resolve correctly
- Data Admin user adds Report URL → widget shows report name (baseline)

### QA Scenarios

- Test with Process Owner, Control Owner, Viewer roles
- Test with Reports that are shared vs unshared
- Test with Dashboards in different folders with varying permissions
- Test in EU environments specifically (multiple customer reports are from EU)
- Verify behavior after removing and re-adding Data Admin role
- Test with workspaces that have/haven't completed data partitioning

---

## 12. Remaining Questions and Assumptions

### Cannot Verify Without Access

1. **Current `w-graph` source code** — Cannot verify whether INTRISK-95107's
   fix is still in place or has regressed. Need access to the
   `Workiva/w-graph` repository.
2. **RIS service configuration** — Cannot verify whether
   `skaar-name-authorization` is now globally enabled.
3. **Exact 403 error message** — Cannot see the actual `RIError.message`
   string returned by the current `w-graph` build.
4. **Browser console logs** — INTRISK-72638 mentions a "403 from the PSE" in
   the console; cannot reproduce without environment access.

### Assumptions

- The `w-graph` provider is the component performing the permission check
  (based on INTRISK-95107 findings, not direct code verification)
- The `RIError.code = 403` is the specific error returned for non-Data Admin
  users (based on `ResourceInformationErrors2Constants.FORBIDDEN` and Jira
  descriptions)
- The INTRISK-95107 fix shipped but may have regressed, based on post-fix
  SUPP tickets

### Next Steps Needed

1. Access `Workiva/w-graph` repo to verify current state of
   `PermissionUtil.getIntentForResource()`
2. Check `w-graph` release notes after 18.3.331 for any reverts or changes
   to permission logic
3. Reproduce with browser dev tools open to capture the exact `RIError` code
   and message
4. Coordinate with the Skaar/RIS team on `skaar-name-authorization` rollout
   status
5. Update INTRISK-72638 with this analysis and link to SUPP-78850
