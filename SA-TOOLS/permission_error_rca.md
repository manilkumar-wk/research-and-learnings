# Root Cause Analysis: NoSuchMethodError in GetEntityPermissions

## Error Summary

```
type: NoSuchMethodError: method not found: 'toString' on null
location: permission_actions.dart:46:118 — GetEntityPermissions.call
```

Appears as a frontend crash but the root cause is that **some entities have access only through modern granular/scoped roles** that don't include the 4 basic actions (`deny`, `read`, `write`, `own`) that Home's legacy permission probe asks for. The backend correctly returns all-false for these entities (no error, HTTP 200), and the frontend crashes because it doesn't handle this case.

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

## Why It Only Affects Certain Folders

This is **not a backend error** — the Policy Evaluator returns HTTP 200 with correct results. Backend logs confirm normal processing:

```
level: info — fielding HTTP permissions request
level: info — POST /s/policy-evaluator/api/v1/canMany HTTP/1.1
```

The issue is that **some entities have access only through modern granular/scoped roles** (e.g. `entity.read`, `file.write`, `tasking.task.create`) that don't include the 4 basic coarse actions (`deny`, `read`, `write`, `own`) that Home's legacy permission probe asks for. The backend correctly returns `allowed: false` for all 4 — the user genuinely doesn't have those basic grants.

**Why large folders surface it more:** More files in a folder = more entities in the batch = higher probability that at least one entity has only granular-role access. A small folder might not contain any such entities, so the crash never triggers.

```
┌──────────────────────────────────────────┬──────────────────────────────────────────┐
│   ENTITY WITH COARSE ROLE (Works ✅)      │   ENTITY WITH GRANULAR-ONLY ROLES        │
│                                          │   (Crashes ❌)                             │
├──────────────────────────────────────────┼──────────────────────────────────────────┤
│                                          │                                          │
│  User has "resourceViewer" role          │  User has "reportingProjectOwner",       │
│  which grants basic 'read' action        │  "taskAdmin", "customFieldAdmin" etc.    │
│                                          │  These grant scoped actions only:        │
│                                          │  entity.read, tasking.task.create, etc.  │
│                                          │  NOT the basic read/write/own/deny       │
│                                          │                                          │
│  Backend returns (HTTP 200, correct):    │  Backend returns (HTTP 200, correct):    │
│  {                                       │  {                                       │
│    deny: false,                          │    deny: false,                          │
│    read: TRUE,  ◄── coarse grant exists  │    read: false,  ◄── no coarse grant,   │
│    write: false,                         │    write: false,     access is through   │
│    own: false                            │    own: false        scoped actions only │
│  }                                       │  }                                       │
│                                          │                                          │
│  Filtered set: {"read"}                  │  Filtered set: {} (EMPTY)                │
│                                          │                                          │
│  ContentEntityActions({"read"})          │  ContentEntityActions({})                │
│    .isViewer = true ✅                    │    .isOwner = false                      │
│                                          │    .isEditor = false                     │
│  entityPermissionFromAction() returns:   │    .isSharer = false                     │
│    WDataEntityPermission.read ✅          │    .isViewer = false                     │
│                                          │    .isDenied = false                     │
│  Bang (!) succeeds                       │                                          │
│                                          │  entityPermissionFromAction() returns:   │
│                                          │    null ❌                                │
│                                          │                                          │
│                                          │  Bang (!) on null → 💥 CRASH             │
└──────────────────────────────────────────┴──────────────────────────────────────────┘
```

### Customer Workaround

Pull files out of the root of the affected folder to reduce the batch size, then re-attempt. Fewer files = fewer entities in the `canMany` batch = lower chance of hitting an entity with granular-only access.

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

### The Real Root Cause: Legacy Coarse-Action Probe vs. Modern Granular Roles

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                                                                   │  │
│  │  ROOT CAUSE: Home's legacy permission probe only asks for 4      │  │
│  │  coarse actions (deny/read/write/own). Modern granular roles     │  │
│  │  grant scoped actions (entity.read, file.write, etc.) that      │  │
│  │  don't include these 4. The backend correctly returns all-false  │  │
│  │  (HTTP 200, no error), but the frontend can't map this to a     │  │
│  │  WDataEntityPermission value.                                    │  │
│  │                                                                   │  │
│  │  CRASH SITE: permission_actions.dart:46 — bang (!) operator      │  │
│  │  on the null return from entityPermissionFromAction(), which     │  │
│  │  was introduced by the null-safety migration (commit 6a286652b) │  │
│  │  and assumes the function never returns null.                    │  │
│  │                                                                   │  │
│  │  WHY NULL: entityPermissionFromAction() receives an empty        │  │
│  │  ContentEntityActions({}) because all 4 basic actions came back  │  │
│  │  false (correctly), falls through all if-branches, returns null. │  │
│  │                                                                   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  The backend is behaving correctly. The mismatch is between Home's      │
│  legacy 4-action permission model and IAM's modern granular role        │
│  system. Large folders surface this more because they contain more      │
│  entities, increasing the chance of hitting one with granular-only      │
│  access.                                                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Contributing Factor: Null-Safety Migration

| Commit | Date | What Changed |
|--------|------|-------------|
| `6a286652b` | Jan 23, 2026 | Changed `stateMap` from `Map<String, WDataEntityPermission?>` to non-nullable, added `!` |
| `b86cec9f9` | Feb 6, 2026 | Changed `UpdatePermissionsMap` parameter to non-nullable |

Before migration: `null` permission values were silently stored in the map.
After migration: `!` operator assumes `entityPermissionFromAction()` never returns null — wrong when an entity has only granular roles.

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

This is a **defensive frontend fix** — it prevents the crash and the refetch loop. The deeper architectural issue is that Home's legacy permission model only understands 4 coarse actions, while IAM's modern role system grants access through hundreds of scoped actions. A longer-term fix would be to either extend the permission probe to account for granular roles, or migrate to the newer entity-level permission model (as suggested by the `@Deprecated('use permissions on the entity instead')` annotation on the class).

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
