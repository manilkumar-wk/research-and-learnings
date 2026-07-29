# Home: GetEntityPermissions NoSuchMethodError Analysis

**Status:** Root cause identified (FE crash on valid BE response)  
**Service / repo:** `home`  
**Symptom:** `NoSuchMethodError: method not found: 'toString' on null`  
**Stack:** `permission_actions.dart` → `GetEntityPermissions.call`  
**Observed when:** Higher-privilege / broader-access users (works for users with only lower grants like read)

---

## TL;DR (Slack-ready)

The **exception is thrown in Home FE**, not in policy-evaluator.

1. Home calls IAM `canMany` for `deny` / `read` / `write` / `own`.
2. Backend returns **HTTP 200** with all `allowed: false` (valid “no coarse grant” result).
3. Home only skips empty maps, not “all false”.
4. `entityPermissionFromAction({})` returns `null`.
5. Force unwrap (`!`) crashes → dart2js surfaces as `toString` on null.

**Fix belongs in Home FE** (null-safe mapping / treat all-false as `none`). Backend is behaving as designed unless product says that all-false payload is wrong for that user/file.

---

## Exception (as reported)

```text
message: Uncaught exception during experience runtime
type: NoSuchMethodError: method not found: 'toString' on null

stacktrace:
  permission_actions.dart 46:118
    GetEntityPermissions.call.<anonymous function>.<anonymous function>
  JsLinkedHashMap.forEach
  permission_actions.dart 40:7
    GetEntityPermissions.call.<anonymous function>
```

Column `46:118` is the trailing `!` on:

```dart
stateMap[utils.getResourceIdFromKeyname(key)] = cmu
    .entityPermissionFromAction(
      cmc.ContentEntityActions(
        actions.keys.where((key) => actions[key]!).toSet(),
      ),
    )!;
```

---

## Crash location: FE or BE?

| Layer | What happens |
|--------|----------------|
| **Home FE** | **Throws** — null assertion on unmappable `canMany` result |
| **policy-evaluator BE** | **Succeeds** — returns 200 with bools; no exception |
| **Trigger** | BE payload with **no** `true` among `deny` / `read` / `write` / `own` |

---

## End-to-end flow

```text
Home UI (context menu / folder select / post-permissions refresh)
  → GetEntityPermissions
  → iam_sdk IamClient.canMany(actions=[deny,read,write,own], resources=[keyname])
  → policy-evaluator POST /canMany
  → OPA workiva/can_many
  → 200 { results: [{ resource, actionsToAllowed: [...] }] }
  → Home maps true actions → WDataEntityPermission
  → if none true → null → ! → crash
```

### Call sites in Home that dispatch `GetEntityPermissions`

- Context menu when permission missing (`home_module.dart`)
- Folder select when folder not in `permissionsMap` (`folder_actions.dart`)
- ~15s refresh after permissions editor (`entity_permission.dart`) — IAM cache lag already noted in code comments

---

## Backend path (does not crash)

**Repo:** `Workiva/policy-evaluator`

**HTTP:** `POST /canMany` → `Server.CanMany` → `OPASdk.CanMany` → OPA path `workiva/can_many`

Resolver builds one `{ Action, Allowed }` per requested action per resource and returns HTTP 200.

---

## Backend response that triggers the FE crash

> Not captured from the specific prod incident; reconstructed from crash path. Confirm via Network tab / PE logs for `canMany`.

**Request (from Home):**

```json
{
  "actions": ["deny", "read", "write", "own"],
  "resources": ["<file-keyname>"]
}
```

**Response that kills FE (HTTP 200):**

```json
{
  "results": [
    {
      "resource": "<file-keyname>",
      "actionsToAllowed": [
        { "action": "deny",  "allowed": false },
        { "action": "read",  "allowed": false },
        { "action": "write", "allowed": false },
        { "action": "own",   "allowed": false }
      ]
    }
  ]
}
```

Parsed in FE as:

```dart
{
  "<file-keyname>": {
    "deny": false,
    "read": false,
    "write": false,
    "own": false
  }
}
```

### Why this blows up in Home

```dart
if (actions.isEmpty) return; // map is NOT empty — has 4 keys
// filtered true-actions → {}
entityPermissionFromAction({}) → null
null! → crash
```

`entityPermissionFromAction` only maps when the true-action set contains `own` / `write` / `read` / `deny`. Empty set → `null`.

---

## Why “works with less permissions”

| User | Typical `canMany` | Result |
|------|-------------------|--------|
| Lower privilege (e.g. read-only) | At least `read: true` | Maps to `read` — OK |
| Higher / broader access | Can open files they can *see* (admin / elevated / complex ACL) but coarse grants all `false` | Unmappable → crash |

So it is not that “more true actions break the mapper.” Higher-privilege users hit the **all-false** path that lower-privilege users rarely see.

---

## Working response (contrast)

```json
{
  "results": [
    {
      "resource": "<file-keyname>",
      "actionsToAllowed": [
        { "action": "deny",  "allowed": false },
        { "action": "read",  "allowed": true },
        { "action": "write", "allowed": false },
        { "action": "own",   "allowed": false }
      ]
    }
  ]
}
```

---

## Related code already null-safe elsewhere

`content_management_sdk` `_getFilePermissionActions` uses the same converter but **null-checks** instead of `!`:

```dart
final permission = cmu.entityPermissionFromAction(...);
if (permission != null) {
  stateMap[...] = permission;
}
```

Home never got that guard (null-safety migration added `!` in Jan 2026 review follow-ups).

Historical note: older Home mapped via `_actionPermissionMapping[entry.key]` on true entries only (no bang on converter).

---

## Recommended fix (Home FE)

In `lib/src/redux/actions/permission_actions.dart`:

```dart
final allowed = actions.entries
    .where((e) => e.value)
    .map((e) => e.key)
    .toSet();

if (allowed.isEmpty) {
  stateMap[utils.getResourceIdFromKeyname(key)] = WDataEntityPermission.none;
  return;
}

final permission = cmu.entityPermissionFromAction(
  cmc.ContentEntityActions(allowed),
);
if (permission != null) {
  stateMap[utils.getResourceIdFromKeyname(key)] = permission;
}
```

Add a unit test for all-false `canMany` payload.

Optional follow-up: stop relying on deprecated `GetEntityPermissions` / `permissionsMap` in favor of permissions on the entity (already marked `@Deprecated`).

---

## How to confirm in prod

1. Reproduce with high-privilege user.
2. Browser Network → filter `canMany`.
3. Expect all four `allowed: false` (or no true allows) for the failing resource.
4. Compare with low-privilege user on a file they can open → at least `read: true`.

---

## Slack copy-paste blurb

```text
Home crash analysis: NoSuchMethodError in GetEntityPermissions (permission_actions.dart:46)

- Crash is FE (null ! on entityPermissionFromAction), not policy-evaluator
- BE returns 200 canMany with deny/read/write/own all false
- Home only skips empty maps → maps {} → null → bang throws (dart2js: toString on null)
- Explains why low-permission users work (read:true) and higher-access users fail (can see file but coarse grants all false)
- Fix: null-safe map / treat all-false as none in home (same pattern already in content_management_sdk)
```
