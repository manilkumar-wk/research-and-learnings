# Root Cause Analysis: NoSuchMethodError in GetEntityPermissions

## Error Summary

```
type: NoSuchMethodError: method not found: 'toString' on null
location: permission_actions.dart:46:118 — GetEntityPermissions.call
```

Appears as a frontend crash but the root cause is the **Policy Evaluator backend timing out under high-volume permission evaluation**, returning all-false results for entities it couldn't evaluate in time. The frontend then crashes because it doesn't handle this case.

---

## End-to-End Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND (Dart/Browser)                            │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐         │
│  │ 1. GetEntityPermissions.call()                                      │         │
│  │    permission_actions.dart:33                                        │         │
│  │                                                                     │         │
│  │    Sends 4 actions: [deny, read, write, own]                        │         │
│  │    Sends N entity keynames as resources                             │         │
│  └──────────────────────────┬──────────────────────────────────────────┘         │
│                             │                                                    │
│                             ▼                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐         │
│  │ 2. IamClient.canMany()                                              │         │
│  │    iam_sdk/iam_client.dart:144                                       │         │
│  │                                                                     │         │
│  │    ┌─────────────────────────────────────────────┐                  │         │
│  │    │ SessionCache (10s TTL, sessionStorage)      │                  │         │
│  │    │ Key = SHA256(sorted_actions::sorted_resources)│                │         │
│  │    │                                             │                  │         │
│  │    │  HIT? ──► Return cached Map                 │                  │         │
│  │    │  MISS? ──► Call _evaluatorApi.canMany()      │                 │         │
│  │    └─────────────────────────────────────────────┘                  │         │
│  └──────────────────────────┬──────────────────────────────────────────┘         │
│                             │                                                    │
└─────────────────────────────┼────────────────────────────────────────────────────┘
                              │
                              │  HTTP POST /canMany
                              │  Authorization: Bearer {accessToken}
                              │  X-Correlation-Id: {uuid}
                              │  Wk-Principal / Wk-Workspace headers
                              │
                              │  Request Body:
                              │  {
                              │    "actions": ["deny","read","write","own"],
                              │    "resources": ["keyname1","keyname2",...]
                              │  }
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                     BACKEND: Policy Evaluator Service                            │
│                     (Workiva microservice — no BFF layer)                        │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐         │
│  │ 3. POST /api/v1/canMany                                             │         │
│  │    policy_evaluator/default_api.dart                                 │         │
│  │                                                                     │         │
│  │    Evaluates each (resource, action) pair against IAM policies       │         │
│  │    Returns ONLY the actions that were requested                      │         │
│  └──────────────────────────┬──────────────────────────────────────────┘         │
│                             │                                                    │
│                             ▼                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐         │
│  │ 4. IAM Policy Database                                              │         │
│  │                                                                     │         │
│  │    Role assignments + policy rules determine:                       │         │
│  │    For each resource × action → allowed: true/false                 │         │
│  │                                                                     │         │
│  │    ┌───────────────────────────────────────────────┐                │         │
│  │    │ Example: Entity with FEW permissions          │                │         │
│  │    │ (e.g., user has "viewer" role)                │                │         │
│  │    │                                               │                │         │
│  │    │ deny: false, read: TRUE, write: false, own: false│            │         │
│  │    │ ──► At least one basic action is TRUE ✅       │               │         │
│  │    └───────────────────────────────────────────────┘                │         │
│  │                                                                     │         │
│  │    ┌───────────────────────────────────────────────┐                │         │
│  │    │ Example: Entity with MORE/COMPLEX permissions │                │         │
│  │    │ (user has only scoped/granular roles that     │                │         │
│  │    │  don't map to the 4 basic actions)            │                │         │
│  │    │                                               │                │         │
│  │    │ deny: false, read: false, write: false, own: false│           │         │
│  │    │ ──► ALL basic actions are FALSE ❌              │               │         │
│  │    │ ──► User may still have access via other       │               │         │
│  │    │     actions like entity.read, file.write etc.  │               │         │
│  │    └───────────────────────────────────────────────┘                │         │
│  └──────────────────────────┬──────────────────────────────────────────┘         │
│                             │                                                    │
│                             │  Response Body:                                    │
│                             │  {                                                 │
│                             │    "results": [                                    │
│                             │      {                                             │
│                             │        "resource": "keyname1",                     │
│                             │        "actionsToAllowed": [                       │
│                             │          {"action":"deny","allowed":false},         │
│                             │          {"action":"read","allowed":false},         │
│                             │          {"action":"write","allowed":false},        │
│                             │          {"action":"own","allowed":false}           │
│                             │        ]                                            │
│                             │      }                                             │
│                             │    ]                                                │
│                             │  }                                                 │
│                             │                                                    │
└─────────────────────────────┼────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                        FRONTEND: Response Processing                             │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐         │
│  │ 5. IamClient deserializes to Map<String, Map<String, bool>>         │         │
│  │    iam_client.dart:158-161                                           │         │
│  │                                                                     │         │
│  │    Result:                                                          │         │
│  │    {                                                                │         │
│  │      "keyname1": {                                                  │         │
│  │        "deny": false, "read": false, "write": false, "own": false   │         │
│  │      }                                                              │         │
│  │    }                                                                │         │
│  └──────────────────────────┬──────────────────────────────────────────┘         │
│                             │                                                    │
│                             ▼                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐         │
│  │ 6. permission_actions.dart:40-47 — forEach over permissions         │         │
│  │                                                                     │         │
│  │    permissions.forEach((key, actions) {                             │         │
│  │      // actions = {deny: false, read: false, write: false, own: false}│       │
│  │      // actions.isEmpty == false, so we don't skip                  │         │
│  │                                                                     │         │
│  │      actions.keys.where((key) => actions[key]!)                     │         │
│  │      // Filters to keys where value is TRUE                         │         │
│  │      // Result: {} (EMPTY SET — no action is true)                  │         │
│  │                                                                     │         │
│  │      ContentEntityActions({})                                       │         │
│  │      // Created with empty actions set                              │         │
│  │    })                                                               │         │
│  └──────────────────────────┬──────────────────────────────────────────┘         │
│                             │                                                    │
│                             ▼                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐         │
│  │ 7. entityPermissionFromAction()                                     │         │
│  │    content_migration_utils.dart:109-126                              │         │
│  │                                                                     │         │
│  │    Receives: ContentEntityActions({})  ← non-null, but empty        │         │
│  │                                                                     │         │
│  │    if (action == null)   → NO (it's not null, just empty)           │         │
│  │    if (action.isOwner)   → NO (empty ∩ {'own'} = {})               │         │
│  │    if (action.isSharer)  → NO (empty ∩ {'write'} = {})             │         │
│  │    if (action.isEditor)  → NO (empty ∩ {'write'} = {})             │         │
│  │    if (action.isViewer)  → NO (empty ∩ {'read'} = {})              │         │
│  │    if (action.isDenied)  → NO (empty ∩ {'deny'} = {})              │         │
│  │                                                                     │         │
│  │    return null;  ◄──── RETURNS NULL                                 │         │
│  └──────────────────────────┬──────────────────────────────────────────┘         │
│                             │                                                    │
│                             ▼                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐         │
│  │ 8. 💥 CRASH — permission_actions.dart:46                            │         │
│  │                                                                     │         │
│  │    ...entityPermissionFromAction(...)!                               │         │
│  │                                        ▲                            │         │
│  │                                        │                            │         │
│  │                              Bang operator (!) on null              │         │
│  │                              ══════════════════════                  │         │
│  │                                                                     │         │
│  │    NoSuchMethodError: method not found: 'toString' on null          │         │
│  └─────────────────────────────────────────────────────────────────────┘         │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## Why It Only Affects Certain Folders (Volume, Not Role Type)

This is a **volume problem**, not a role-type problem. When a folder contains many files, `GetEntityPermissions` sends a large batch of entity IDs to `canMany`. The Policy Evaluator must evaluate `N entities x 4 actions` permission checks against the IAM database. Under high volume, the backend **times out before finishing** and returns incomplete/all-false results for entities it couldn't evaluate in time.

```
┌─────────────────────────────────────────┬──────────────────────────────────────────┐
│   SMALL FOLDER / FEW FILES (Works ✅)    │   LARGE FOLDER / MANY FILES (Crashes ❌)  │
├─────────────────────────────────────────┼──────────────────────────────────────────┤
│                                         │                                          │
│  canMany request:                       │  canMany request:                        │
│  4 actions × 10 entities = 40 evals     │  4 actions × 500 entities = 2000 evals   │
│  Backend completes within timeout ✅     │  Backend TIMES OUT before finishing ❌    │
│                                         │                                          │
│  Backend returns real results:          │  Backend returns incomplete results:     │
│  {                                      │  {                                       │
│    deny: false,                         │    deny: false,                          │
│    read: TRUE,  ◄── actual grant        │    read: false,  ◄── timed out, not     │
│    write: false,                        │    write: false,     actually evaluated  │
│    own: false                           │    own: false                            │
│  }                                      │  }                                       │
│                                         │                                          │
│  Filtered set: {"read"}                 │  Filtered set: {} (EMPTY)                │
│                                         │                                          │
│  ContentEntityActions({"read"})         │  ContentEntityActions({})                │
│    .isViewer = true ✅                   │    .isOwner = false                      │
│                                         │    .isEditor = false                     │
│  entityPermissionFromAction() returns:  │    .isSharer = false                     │
│    WDataEntityPermission.read ✅         │    .isViewer = false                     │
│                                         │    .isDenied = false                     │
│  Bang (!) succeeds                      │                                          │
│                                         │  entityPermissionFromAction() returns:   │
│                                         │    null ❌                                │
│                                         │                                          │
│                                         │  Bang (!) on null → 💥 CRASH             │
└─────────────────────────────────────────┴──────────────────────────────────────────┘
```

### Customer Workaround

Pull files out of the root of the affected folder to reduce the batch size, then re-attempt. This reduces the volume of permission evaluations so the backend can finish within its timeout window.

---

## Layer-by-Layer Analysis

### Layer 1: Frontend — Redux Action
**File:** `lib/src/redux/actions/permission_actions.dart`

```dart
// Lines 13-18: Only 4 basic actions are mapped
Map<String, WDataEntityPermission> _actionPermissionMapping = {
  IamActions.deny: WDataEntityPermission.none,   // 'deny'
  IamActions.read: WDataEntityPermission.read,   // 'read'
  IamActions.write: WDataEntityPermission.share,  // 'write'
  IamActions.own: WDataEntityPermission.own       // 'own'
};

// Lines 34-37: Sends these 4 actions + entity keynames to policy evaluator
homeContext.iamClient.canMany(
    actions: _actionPermissionMapping.keys,
    resources: entityIds.map(utils.getKeynameFromResourceId).whereNotNull().toList())
```

**No issue here** — the request is correctly formed with the 4 basic actions.

---

### Layer 2: IAM SDK Client — HTTP + Caching
**File:** `iam_sdk-3.78.0/lib/src/iam_client.dart`

```dart
// Lines 144-166
Future<Map<String, Map<String, bool>>> canMany({...}) {
  final request = CanManyRequest((b) => b
    ..actions = ListBuilder(actions)
    ..resources = ListBuilder(resources));

  // 10-second SessionCache (browser sessionStorage)
  return _canManyCache.getOrFetch(key, () async {
    final response = await _evaluatorApi.canMany(canManyRequest: request);
    // Deserializes response into Map<String, Map<String, bool>>
    return Map.fromEntries(response.data!.results.map((element) => MapEntry(
        element.resource,
        Map.fromEntries(element.actionsToAllowed
            .map((a) => MapEntry(a.action, a.allowed))))));
  });
}
```

**No issue here** — faithful deserialization. The SDK returns exactly what the backend sends. The response `Map<String, bool>` will contain entries for each of the 4 requested actions with their `allowed` boolean.

---

### Layer 3: Network — HTTP POST
**Endpoint:** `POST {policy-evaluator-uri}/api/v1/canMany`

```
Headers:
  Authorization: Bearer {session.accessToken}
  X-Correlation-Id: {uuid-v4}
  Content-Type: application/json
  Wk-Principal: {principalId}
  Wk-Workspace: {workspaceId}

Request:
  { "actions": ["deny","read","write","own"], "resources": ["keyname1",...] }

Response:
  { "results": [
      { "resource": "keyname1",
        "actionsToAllowed": [
          {"action":"deny","allowed":false},
          {"action":"read","allowed":false},
          {"action":"write","allowed":false},
          {"action":"own","allowed":false}
        ] } ] }
```

**No BFF/middleware layer** — the frontend calls the Policy Evaluator service directly.

---

### Layer 4: Backend — Policy Evaluator Service
**Service:** `policy-evaluator` (Workiva microservice)

This service evaluates IAM policy rules stored in the IAM database:

```
┌─────────────────────────────────────────────────────────────────┐
│                   Policy Evaluator Service                       │
│                                                                 │
│  For each (resource, action) pair:                              │
│                                                                 │
│  1. Look up principal's role assignments for the resource       │
│  2. Look up which actions each role grants                     │
│  3. Evaluate policy rules (allow/deny/conditions)              │
│  4. Return allowed: true/false                                 │
│                                                                 │
│  The service ONLY evaluates the requested actions.             │
│  It does NOT return additional actions the user has.           │
│  It does NOT distinguish between "no role" and                 │
│  "has roles but they don't grant these specific actions".      │
└─────────────────────────────────────────────────────────────────┘
```

### Layer 5: IAM Database

```
┌─────────────────────────────────────────────────────────────────┐
│                        IAM Database                              │
│                                                                 │
│  Role Assignments Table:                                        │
│  ┌──────────┬────────────┬───────────────────────┐              │
│  │ Principal│ Resource   │ Role                  │              │
│  ├──────────┼────────────┼───────────────────────┤              │
│  │ user-123 │ entity-A   │ resourceViewer        │ → few perms  │
│  │ user-123 │ entity-B   │ reportingProjectOwner │ → more perms │
│  │ user-123 │ entity-B   │ taskAdmin             │              │
│  │ user-123 │ entity-B   │ customFieldAdmin      │              │
│  └──────────┴────────────┴───────────────────────┘              │
│                                                                 │
│  Role Definitions:                                              │
│  ┌───────────────────────┬────────────────────────────────┐     │
│  │ Role                  │ Granted Actions                │     │
│  ├───────────────────────┼────────────────────────────────┤     │
│  │ resourceViewer        │ read ✅ (basic action)         │     │
│  │ reportingProjectOwner │ entity.read, entity.write,     │     │
│  │                       │ run.create, schedule.read ...   │     │
│  │                       │ (scoped actions only, NOT       │     │
│  │                       │  basic 'read'/'write'/'own')    │     │
│  │ taskAdmin             │ tasking.task.create, etc.       │     │
│  │ customFieldAdmin      │ entityCustomFieldDef.create...  │     │
│  └───────────────────────┴────────────────────────────────┘     │
│                                                                 │
│  entity-A: user has 'read' action via resourceViewer → ✅       │
│  entity-B: user has many scoped actions via multiple roles      │
│            but NONE of the 4 basic actions (deny/read/write/own)│
│            → all return false → 💥                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Root Cause Pinpointed

### The Real Root Cause: Backend Timeout Under High Volume

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                                                                   │  │
│  │  ROOT CAUSE: Policy Evaluator backend times out when             │  │
│  │  evaluating high-volume permission batches (large folders).      │  │
│  │  Returns all-false for entities it couldn't finish evaluating.   │  │
│  │                                                                   │  │
│  │  CRASH SITE: permission_actions.dart:46 — bang (!) operator      │  │
│  │  on the null return from entityPermissionFromAction(), which     │  │
│  │  was introduced by the null-safety migration (commit 6a286652b) │  │
│  │  and assumes the function never returns null.                    │  │
│  │                                                                   │  │
│  │  WHY NULL: entityPermissionFromAction() receives an empty        │  │
│  │  ContentEntityActions({}) because all 4 basic actions came back  │  │
│  │  false (timeout), falls through all if-branches, returns null.   │  │
│  │                                                                   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  The frontend's bang operator is the proximate cause of the crash,      │
│  but the deeper issue is the backend's inability to handle large        │
│  permission batches within its timeout window.                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Contributing Factor: Null-Safety Migration

| Commit | Date | What Changed |
|--------|------|-------------|
| `6a286652b` | Jan 23, 2026 | Changed `stateMap` from `Map<String, WDataEntityPermission?>` to non-nullable, added `!` |
| `b86cec9f9` | Feb 6, 2026 | Changed `UpdatePermissionsMap` parameter to non-nullable |

Before migration: `null` permission values were silently stored in the map.
After migration: `!` operator assumes `entityPermissionFromAction()` never returns null — wrong when the backend times out.

---

## Complete Request/Response Lifecycle

```
                    ┌──────────┐
                    │  Browser │
                    └────┬─────┘
                         │
    ┌────────────────────┼────────────────────────────────────────────────────┐
    │                    ▼                                                    │
    │   GetEntityPermissions(entityIds)                                       │
    │   permission_actions.dart:30                                            │
    │                    │                                                    │
    │   entityIds.map(getKeynameFromResourceId)                               │
    │   ─── base64-decodes resourceIds to keynames                           │
    │   ─── strips WFDataEntity prefix                                       │
    │                    │                                                    │
    │                    ▼                                                    │
    │   iamClient.canMany(                                                   │
    │     actions: [deny, read, write, own],                                 │
    │     resources: [keyname1, keyname2, ...]                               │
    │   )                                                                    │
    │                    │                                                    │
    │        ┌───────────┴───────────┐                                       │
    │        ▼                       ▼                                        │
    │   SessionCache              Cache MISS                                 │
    │   HIT → return               │                                         │
    │   cached map                  ▼                                         │
    │                    POST /api/v1/canMany                                 │
    │                    ──────────────────►  Policy Evaluator Service        │
    │                                        │                               │
    │                                        ▼                               │
    │                               ┌────────────────────┐                   │
    │                               │   IAM Database     │                   │
    │                               │                    │                   │
    │                               │ Look up:           │                   │
    │                               │ - Role assignments │                   │
    │                               │ - Policy rules     │                   │
    │                               │ - Action grants    │                   │
    │                               └────────┬───────────┘                   │
    │                                        │                               │
    │                    ◄───────────────────┘                                │
    │                    Response: {results: [...]}                           │
    │                    │                                                    │
    │                    ▼                                                    │
    │   Deserialize → Map<String, Map<String, bool>>                         │
    │   e.g. {"keyname1": {"deny":false,"read":false,"write":false,"own":false}}│
    │                    │                                                    │
    │                    ▼                                                    │
    │   permissions.forEach((key, actions) {                                 │
    │     // actions = {"deny":false,"read":false,"write":false,"own":false}  │
    │     // actions.isEmpty = false (has 4 entries!)                         │
    │                                                                        │
    │     actions.keys.where((k) => actions[k]!)                             │
    │     // Filters to keys where value == true                             │
    │     // Result: {} (EMPTY — no action is true)                          │
    │                                                                        │
    │     ContentEntityActions({})                                           │
    │     // Empty set → isOwner/isSharer/isEditor/isViewer/isDenied = false │
    │                                                                        │
    │     entityPermissionFromAction(ContentEntityActions({}))               │
    │     // Falls through ALL if-branches                                   │
    │     // Returns: NULL                                                   │
    │                                                                        │
    │     ...entityPermissionFromAction(...)!                                 │
    │     // 💥 Bang on NULL → NoSuchMethodError                             │
    │   })                                                                   │
    │                                                                        │
    └────────────────────────────────────────────────────────────────────────┘
```

---

## Why actions.isEmpty Check Doesn't Help

```dart
if (actions.isEmpty) {   // line 41
  return;                // line 42 — skip this entity
}                        // line 43
```

This check was meant to handle entities with no permission data. But the backend **always returns all 4 requested actions** with `allowed: true/false`. So `actions` is **never empty** — it always has 4 entries. The check only guards against the case where the backend returns zero action entries for a resource, which doesn't happen with `canMany`.

The problem is that `actions` is `{"deny":false, "read":false, "write":false, "own":false}` — it has 4 entries (not empty), but all values are `false`.

---

## Fix

In `permission_actions.dart`, remove the bang operator and default to `WDataEntityPermission.none`:

```dart
permissions.forEach((key, actions) {
  if (actions.isEmpty) {
    return;
  }

  final permission = cmu.entityPermissionFromAction(
      cmc.ContentEntityActions(
          actions.keys.where((key) => actions[key]!).toSet()));

  stateMap[utils.getResourceIdFromKeyname(key)] =
      permission ?? WDataEntityPermission.none;
});
```

### Why `none` instead of skipping

Using `?? WDataEntityPermission.none` instead of a null check + skip because:

1. **Prevents infinite refetch loops** — `folder_actions.dart:104` checks `permissionsMap.containsKey(folderResourceId)` and re-dispatches `GetEntityPermissions` if the key is missing. Skipping would leave the key absent, causing the same all-false `canMany` response forever.
2. **Semantically correct** — IAM already answered: all 4 basic actions are `false`. That **is** `none`.
3. **Downstream consumers expect it** — `entity_table.dart:113` and `selectors.dart:205` check for `WDataEntityPermission.none` to disable UI actions and set read-only state.

### What this fix does NOT solve

This is a **defensive frontend fix** — it prevents the crash and the refetch loop. The deeper issue (backend timing out under high-volume permission evaluation) remains and may need backend-side investigation (batching, pagination, or timeout tuning in the Policy Evaluator service).

### PR

https://github.com/Workiva/home/pull/18604

---

## Key Files Reference

| Layer | File | Role |
|-------|------|------|
| Frontend Action | `lib/src/redux/actions/permission_actions.dart:33-52` | Entry point + crash site (line 46) |
| Frontend Context | `lib/src/models/home_context.dart:154` | IamClient instantiation |
| Frontend Utils | `lib/src/utils/entity_utils.dart:22-61` | ResourceId ↔ Keyname conversion |
| IAM SDK Client | `iam_sdk-3.78.0/lib/src/iam_client.dart:144-166` | canMany() with caching |
| IAM SDK Auth | `iam_sdk-3.78.0/lib/src/dio_wdesk_interceptor.dart` | Bearer token injection |
| IAM SDK Cache | `iam_sdk-3.78.0/lib/src/session_cache.dart` | 10s TTL sessionStorage cache |
| HTTP Layer | `policy_evaluator-1.1.583/lib/src/api/default_api.dart` | POST /canMany HTTP call |
| Response Model | `policy_evaluator-1.1.583/lib/src/model/can_many_response.dart` | Response deserialization |
| Conversion | `content_migration_utils-1.59.47/lib/src/content_migration_utils.dart:109-126` | entityPermissionFromAction() — returns null |
| Permission Model | `content_management_client-1.31.63/lib/src/models/entities/content_entity_actions.dart` | ContentEntityActions — isOwner/isViewer etc. |
| IAM Constants | `iam_sdk-3.78.0/lib/src/iam_constants.dart` | 590+ IamActions, 4 basic ones used |
