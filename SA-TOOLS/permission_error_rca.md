# Root Cause Analysis: NoSuchMethodError in GetEntityPermissions

## Error Summary

```
type: NoSuchMethodError: method not found: 'toString' on null
location: permission_actions.dart:46:118 — GetEntityPermissions.call
```

Appears as a frontend crash but the root cause is a **mismatch between the backend response and frontend assumptions** during the null-safety migration.

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

## Comparison: Few Permissions vs. More Permissions

```
┌─────────────────────────────────────────┬──────────────────────────────────────────┐
│      FEW PERMISSIONS (Works ✅)          │      MORE/COMPLEX PERMISSIONS (Crashes ❌)│
├─────────────────────────────────────────┼──────────────────────────────────────────┤
│                                         │                                          │
│  User has "viewer" role on entity       │  User has scoped/granular roles only     │
│                                         │  (e.g., entity.read, file.write,         │
│                                         │   filesystem.files.list, etc.)           │
│                                         │                                          │
│  Backend returns:                       │  Backend returns:                        │
│  {                                      │  {                                       │
│    deny: false,                         │    deny: false,                          │
│    read: TRUE,  ◄── at least one TRUE   │    read: false,                          │
│    write: false,                        │    write: false,                         │
│    own: false                           │    own: false  ◄── ALL FALSE             │
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

### Where: The Conversion Layer (Frontend)

```
permission_actions.dart:46 — the bang (!) operator
```

### Why: Assumption Violated During Null-Safety Migration

| Commit | Date | What Changed |
|--------|------|-------------|
| `6a286652b` | Jan 23, 2026 | Changed `stateMap` from `Map<String, WDataEntityPermission?>` to non-nullable, added `!` |
| `b86cec9f9` | Feb 6, 2026 | Changed `UpdatePermissionsMap` parameter to non-nullable |

Before migration: `null` permission values were silently stored in the map.
After migration: `!` operator assumes `entityPermissionFromAction()` never returns null — **wrong**.

### The Backend's Role

The backend is **behaving correctly** — it returns `allowed: false` for actions the user doesn't have. The problem is that:

1. The **frontend only asks about 4 basic actions** (`deny`, `read`, `write`, `own`)
2. Many roles grant **scoped actions** (like `entity.read`, `file.write`, `filesystem.files.list`) that don't include these 4 basic ones
3. When a user has **only scoped/granular roles** on an entity, all 4 basic actions return `false`
4. The frontend has **no fallback** for this case

### The Real CRA (Critical Root Area)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  entityPermissionFromAction()  in content_migration_utils               │
│  content_migration_utils.dart:109-126                                    │
│                                                                         │
│  This function can ONLY map to 4 enum values:                           │
│    own / share / read / none                                            │
│                                                                         │
│  It returns NULL when none of the 4 basic actions match.                │
│  The IAM system has 590+ possible actions (IamActions class),           │
│  but this function only recognizes 4.                                   │
│                                                                         │
│  The FRONTEND (permission_actions.dart:46) then applies ! on the null.  │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  PRIMARY: entityPermissionFromAction() returning null             │  │
│  │           when ContentEntityActions has empty actions set         │  │
│  │                                                                   │  │
│  │  SECONDARY: permission_actions.dart:46 using ! instead of        │  │
│  │             null-safe handling — introduced by null-safety        │  │
│  │             migration commit 6a286652b                           │  │
│  │                                                                   │  │
│  │  ROOT: Mismatch between IAM's granular permission model          │  │
│  │        (590+ actions) and the frontend's simplified model        │  │
│  │        (4 basic actions)                                         │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

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

## Suggested Fix

In `permission_actions.dart`, replace the bang operator with a null check:

```dart
permissions.forEach((key, actions) {
  if (actions.isEmpty) {
    return;
  }

  final permission = cmu.entityPermissionFromAction(
      cmc.ContentEntityActions(
          actions.keys.where((key) => actions[key]!).toSet()));

  if (permission != null) {
    stateMap[utils.getResourceIdFromKeyname(key)] = permission;
  }
  // If null, skip — user has no basic permission level on this entity
});
```

This treats unrecognized permission combinations as "no basic permission info" and skips them instead of crashing.

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
