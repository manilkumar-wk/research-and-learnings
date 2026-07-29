# Root Cause Analysis: NoSuchMethodError in GetEntityPermissions

## Error Summary

```
type: NoSuchMethodError: method not found: 'toString' on null
location: permission_actions.dart:46:118 — GetEntityPermissions.call
```

Appears as a frontend crash but the root cause is a backend response the frontend doesn't handle.

---

## Error Flow

### 1. The IAM Backend Call

`permission_actions.dart:34-37` calls `iamClient.canMany()` with 4 actions from `_actionPermissionMapping`:

```dart
Map<String, WDataEntityPermission> _actionPermissionMapping = {
  IamActions.deny: WDataEntityPermission.none,   // 'deny'
  IamActions.read: WDataEntityPermission.read,   // 'read'
  IamActions.write: WDataEntityPermission.share,  // 'write'
  IamActions.own: WDataEntityPermission.own       // 'own'
};
```

### 2. The Backend Response

`canMany()` returns `Map<String, Map<String, bool>>` — for each entity keyname, a map of `action -> bool` indicating whether the user has that permission.

### 3. The Frontend Processing (Crash Site)

```dart
permissions.forEach((key, actions) {                              // line 40
  if (actions.isEmpty) { return; }                                // line 41-43

  stateMap[utils.getResourceIdFromKeyname(key)] = cmu
      .entityPermissionFromAction(
          cmc.ContentEntityActions(
              actions.keys.where((key) => actions[key]!).toSet()
          ))!;                                                    // line 45-46 — bang (!) on null
});
```

`actions.keys.where((key) => actions[key]!)` filters to only the actions that are `true` (granted), then wraps them in a `ContentEntityActions` and passes to `entityPermissionFromAction()`.

### 4. The Conversion Function (Returns Null)

In `content_migration_utils` (`content_migration_utils.dart:109-126`):

```dart
WDataEntityPermission? entityPermissionFromAction(ContentEntityActions? action) {
  if (action == null) return null;

  if (action.isOwner) return WDataEntityPermission.own;                        // checks for 'own'
  else if (action.isSharer || action.isEditor) return WDataEntityPermission.share; // checks for 'write'
  else if (action.isViewer) return WDataEntityPermission.read;                 // checks for 'read'
  else if (action.isDenied) return WDataEntityPermission.none;                 // checks for 'deny'

  return null;  // <-- FALLS THROUGH — no recognized action matched
}
```

`ContentEntityActions` checks (`content_entity_actions.dart:34-38`):

```dart
bool get isOwner  => actions.intersection({'own'}).isNotEmpty;
bool get isEditor => actions.intersection({'write'}).isNotEmpty;
bool get isSharer => actions.intersection({'write'}).isNotEmpty;
bool get isViewer => actions.intersection({'read'}).isNotEmpty;
bool get isDenied => actions.intersection({'deny'}).isNotEmpty;
```

### 5. The Crash

Back at line 46, the `!` (bang operator) force-unwraps the `null` return, causing:

```
NoSuchMethodError: method not found: 'toString' on null
```

---

## Why "Only Files With More Permissions"

The code queries only 4 basic actions: `deny`, `read`, `write`, `own`.

- **Simple permissions**: At least one of these 4 is `true`, so `entityPermissionFromAction` returns a valid `WDataEntityPermission`. No crash.

- **Complex/granular permissions**: The IAM backend returns `{deny: false, read: false, write: false, own: false}` for the entity. The user's access may be managed through more granular or scoped actions (e.g., `entity.read`, `file.write`, etc.) rather than the 4 basic ones. The `where` filter produces an **empty set** (no basic actions are `true`). `entityPermissionFromAction` receives a `ContentEntityActions({})`, none of the `isOwner/isEditor/isSharer/isViewer/isDenied` checks match, and it **returns `null`**.

---

## Why It Looks Like a Frontend Error But Is a Backend Issue

| Layer | What happens |
|-------|-------------|
| **IAM Backend** | Returns entities where none of the 4 basic actions (`deny/read/write/own`) are granted — the user's access is through other action types |
| **Frontend** | Assumes every entity will match at least one of the 4 basic actions — introduced by null-safety migration (commit `6a286652b`) which added the `!` bang operator |

The stacktrace points entirely to Dart/frontend code, but the underlying problem is that the IAM backend can return a permission state the frontend was never designed to handle.

---

## Historical Context (Null-Safety Migration)

| Commit | Date | Change |
|--------|------|--------|
| `6a286652b` | Jan 23, 2026 | Changed `stateMap` from `Map<String, WDataEntityPermission?>` to `Map<String, WDataEntityPermission>` and added `!` bang operator |
| `b86cec9f9` | Feb 6, 2026 | Changed `UpdatePermissionsMap` parameter type from nullable to non-nullable |

Before the migration, `stateMap` accepted nullable values, so a `null` from `entityPermissionFromAction` was silently stored. The migration forced non-nullable values and papered over the issue with `!` instead of handling the null case.

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
  // If null, skip this entity — it has no recognized basic permission level
});
```

This treats unrecognized permission combinations as "no permission info available" and skips them rather than crashing. Entities with only granular/scoped permissions will simply not appear in the permissions map instead of causing a runtime exception.

---

## Key Files

| File | Role |
|------|------|
| `lib/src/redux/actions/permission_actions.dart` | Crash site — lines 40-47 |
| `content_migration_utils/lib/src/content_migration_utils.dart:109-126` | `entityPermissionFromAction()` — returns null |
| `content_management_client/lib/src/models/entities/content_entity_actions.dart` | `ContentEntityActions` — permission check logic |
| `iam_sdk/lib/src/iam_constants.dart` | `IamActions` — the 4 basic actions + hundreds of granular ones |
| `lib/src/utils/entity_utils.dart` | `getResourceIdFromKeyname` / `getKeynameFromResourceId` |
