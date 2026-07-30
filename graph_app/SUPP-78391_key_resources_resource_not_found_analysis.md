# SUPP-78391 — Key Resources `<Resource not found>` on Landing Pages

**Date:** 2026-07-30
**Tickets:** [SUPP-78391](https://jira.atl.workiva.net/browse/SUPP-78391),
prior [INTRISK-86141](https://jira.atl.workiva.net/browse/INTRISK-86141),
platform mitigations [AS-5155](https://jira.atl.workiva.net/browse/AS-5155) /
[AS-5135](https://jira.atl.workiva.net/browse/AS-5135)
**Customer:** Lincoln International LLC — Investment Reporting
(`QWNjb3VudB8xMTU0MTUwMjAzMA`), user Joe Rufolo; Landing Pages board with many
Key Resources folder widgets

---

## Verdict

`<Resource not found>` is a **catch-all UI label** in graph_app, not a
literal “object missing” or “no permission” signal. For this customer it
most likely means **folder name enrichment failed** (Content Management
`getFiles` miss / error / rate pressure), not that Joe lacks access.

This is **unlikely to be caused by null-safety**. The cited commit
[`4a8ead9`](https://github.com/Workiva/graph_app/commit/4a8ead937f9f4a333ff0f8a0ae35dad38c9e2a7f)
only NNBD-migrated **tests** for Key Resources; production client
behavior for the error string was already intentional.

There **is** a graph_app problem worth GRC attention: folder enrichment
after INTRISK-86141 can **amplify** `getFiles` traffic (and may paginate
incorrectly). Rate-limit bumps help partially (BC Capital Partners
appeared after Tony’s increase) but do not fix the client request
pattern. Hard to repro with fewer folders is expected.

---

## Where `<Resource not found>` comes from

String: `WSoxIntl.resourceNotFoundInBrackets` → `'<Resource not found>'`.

### Path A — RIS (non-`files` resources)

`_getResourceName` maps **any** `ResourceData.err` (including
`TOO_MUCH_LOAD` and cast failures) to that string:

```435:448:lib/src/landing_page_widgets/key_resources/key_resources_widget_client.dart
  String? _getResourceName(ResourceData data) {
    if (data.err != null) {
      // ...
      _logger.severe('Error getting resource name', riError);
      return WSoxIntl.resourceNotFoundInBrackets;
    }
    return data.data;
  }
```

Unit test explicitly expects this for `TOO_MUCH_LOAD` and non-RI errors.

### Path B — UI empty name (folders often hit this)

List rendering treats empty/null `displayName` the same way:

```41:52:lib/src/landing_page_widgets/key_resources/components/key_resources_widget_list.dart
    ReactElement _renderListItemText(KeyResource? resource) {
      final isNameEmpty = resource?.displayName?.isEmpty ?? true;
      // ...
            isNameEmpty ? (WSoxIntl.resourceNotFoundInBrackets) : resource!.displayName,
```

### Path C — Content Management folders (`service == 'files'`)

Folders are **not** resolved via RIS. `_enrichKeyResourcesFromCM` calls
`ContentService.getFiles`. If the folder is never found after pagination,
it returns the resource **unchanged** (no `displayName`) → Path B UI
label. True lack of access can look identical, which matches PSE’s
permission repro — but that is a **shared symptom**, not proof of the
customer root cause (access was verified; rate-limit increase made BC
Capital Partners appear).

---

## Why “not too many permission checks globally” can still be right

github-ded’s read is correct for the **mapping**: one failed
per-resource name lookup (RIS error / null / empty) → same label.

What that does **not** rule out: **mass CM/RIS load** causing those
per-resource lookups to fail. Splunk already showed ~694
`GET /api/v1/files/` authenticated requests from one user in ~10s
(~1/3 of the 60s quota). Absence of the exact
`Request quota has been exhausted` string now may mean softer failures,
different status/body after AS-5155, or failures earlier in pagination
that still leave `displayName` empty.

---

## Folder enrichment: request amplifier (graph_app)

```387:430:lib/src/landing_page_widgets/key_resources/key_resources_widget_client.dart
  Future<List<KeyResource>> _enrichKeyResourcesFromCM(...) async {
    final allFileData = await _contentService.getFiles(GetFilesRequest()..kindFilter = {'Folder'});
    String? nextRequestCursor = allFileData.nextRequest?.cursor;

    final enrichedResources = keyResources.map((keyResource) async {
      // ...
      fileData = await findFileByIdWithCursor(fileData, folderId, nextRequestCursor);
      if (fileData == null) {
        return keyResource; // empty name → <Resource not found>
      }
      // ...
    });
    return await Future.wait(enrichedResources);
  }

  Future<ContentEntity?> findFileByIdWithCursor(...) async {
    while (fileData == null && nextRequestCursor != null) {
      final nextSetFileData = await _contentService.getFiles(GetFilesRequest()
        ..kindFilter = {folderId}   // suspicious: initial call used {'Folder'}
        ..cursor = nextRequestCursor);
      // ...
    }
  }
```

Problems:

1. **Concurrent per-folder cursor walks** (`Future.wait` over every
   missing folder) — each folder can paginate independently from the
   same starting cursor → O(folders × pages) `getFiles` calls. Matches
   Andrew’s note and the Splunk burst. Landing page not deduping across
   same-type widgets multiplies this further.

2. **`kindFilter = {folderId}` on continuation** — initial page uses
   `{'Folder'}`; cursor pages use the folder UUID as `kindFilter`. That
   looks incorrect vs INTRISK-86141’s intent (keep listing folders via
   `nextRequest`). If wrong, pagination never finds the folder → empty
   name forever (except lucky first-page hits).

3. **INTRISK-86141** closed the “>1000 folders, add link shows resource
   not found” gap with this cursor logic; customer scale + many widgets
   may be exposing remaining flaws / amplification.

---

## Explaining the odd observations

| Observation | Likely explanation |
| --- | --- |
| Can’t repro with many folders locally | Needs large workspace folder catalog + many KR widgets + rate pressure; first page may always contain your test folders |
| Repro by removing access | Same UI label for empty/error name; permission is one cause, not the only one |
| “Stops in alphabetical order” (A → then C after limit bump) | Resources are **sorted by `displayName` after enrich**; partial success + sort makes the failure frontier look alphabetical. Limit bump → more names resolve → frontier moves (A… → C…) |
| BC Capital Partners fixed after rate limit increase | Transient CM capacity / quota; not permissions (history unchanged) |
| Quota message gone but still broken | Still failing lookups; logging string may differ; or failures are empty results rather than explicit quota text |
| NNBD / commit `4a8ead9` | Test-only NNBD for KR client tests — **not** a smoking gun for prod |

---

## Null-safety?

Unlikely primary cause. NNBD batches touched Key Resources source
earlier (INTRISK-95126 … 101359); the cited June commit is tests.
Behavior “map any enrich failure → `<Resource not found>`” predates
that. Focus on CM load + folder enrich design, not a speculative NNBD
regression, unless someone finds a concrete null-coalesce that drops
names on the success path (none obvious in `_enrichKeyResourcesFromCM`
success branch).

---

## Who to bring in / what to look at

**Yes — bring GRC (graph_app Key Resources) in**, alongside Files /
content-management / Landing Page owners already on the thread.

### GRC / graph_app checklist

1. Fix folder enrich to **paginate folders once** (`kindFilter:
   {'Folder'}` + shared cursor), build `id → name` map, then resolve
   all KR folder wurls — no per-folder parallel cursor storms.
2. Confirm whether `kindFilter = {folderId}` is valid CM API usage; if
   not, treat as bug from INTRISK-86141.
3. Differentiate UX: permission denied vs not found vs temporary load
   failure (today all look the same).
4. Log `RIError.code` / CM failures with resource id (severe logs exist
   for RIS; CM miss is silent empty name).
5. Consider board-level dedupe / caching of `getFiles` across widgets.

### Platform / CM checklist (already partly done)

1. Correlate remaining failures with `getFiles` 429 / errors after
   AS-5155 (even if message text ≠ old quota string).
2. Whether folder widgets also fan out into **children file** fetches
   elsewhere in Landing Page (outside this enrich method).
3. Guidance on KR widgets at this scale until client fix ships.

### Quick customer / debug asks

1. Count Key Resources **folder** entries across the board vs file /
   GRC object entries.
2. Browser network: volume of `/api/v1/files/` on board load; any 429 /
   5xx; whether cursor pages use weird kind filters.
3. Console/`KeyResourcesClient` severe logs for RIS errors on non-folder
   items.
4. Whether failures are mostly `files/folder/...` wurls (points at CM
   path) vs mixed types (RIS path too).

---

## Suggested reply

> Agree this is worth GRC looking at in graph_app. `<Resource not
> found>` is a catch-all when name enrichment fails (RIS error **or**
> empty display name). Lack of access can produce it, but so can CM
> `getFiles` misses under load — which fits verified perms, partial
> recovery after rate-limit bumps, and alphabetical “frontier” after
> sort.
>
> The June NNBD commit called out only migrated Key Resources **tests**,
> not this mapping. Stronger lead: folder enrich from INTRISK-86141
> concurrently paginates per folder (and may use a bad `kindFilter` on
> cursor pages), which can explode `/api/v1/files/` traffic — consistent
> with Splunk and with widgets still failing after quota message
> disappeared.
>
> Rate limits are a bandage; next step is fix/dedupe folder name
> resolution in Key Resources + keep CM owners for remaining quota
> behavior.
