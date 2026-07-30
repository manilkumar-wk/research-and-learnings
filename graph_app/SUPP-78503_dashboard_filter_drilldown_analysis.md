# SUPP-78503 — Dashboard filter missing in drill-down

**Date:** 2026-07-30
**Tickets:** [SUPP-78503](https://jira.atl.workiva.net/browse/SUPP-78503),
related [INTRISK-101358](https://jira.atl.workiva.net/browse/INTRISK-101358)
(shipped in graph_app 10.4.12 / [INTRISK-101923](https://jira.atl.workiva.net/browse/INTRISK-101923))
**Customer:** ASML — dashboard
`1a3a053e-839f-473a-848a-f4dbb679ca7d`, report
"ICRC Assessments Reporting | Used for dashboards", column
"Country Reporting (Country Reporting)"

---

## Verdict

This is **not primarily a regression of INTRISK-101358**, and **not**
caused by brackets in the column name. It looks like a **longstanding
asymmetry** between how dashboard (environment) filters are applied to
the **chart query** vs how they are re-applied when building
**drill-down / Data View** outputs.

The bar-count vs Data View list mismatch can be the **same class of
bug** as INTRISK-101358 (env filter not fully applied on the
non-chart path), but the dominant SUPP symptom — column visible in
drill-down **with no filter chip**, showing other countries — points at
`applyEnvFilter` skipping outputs whose `annotationValues` is `null`.

---

## Symptoms (from PSE / SUPP)

| Surface | Observed |
| --- | --- |
| Dashboard filter chip | Shows "Country Reporting (Country Reporting)" |
| Chart bars | Count respects the dashboard filter |
| Drill-down filter bar | Other dashboard filters appear; **Country does not** |
| Drill-down table | Country column is present but **unfiltered** (other countries) |
| Same filter on report (data) | Works |
| Repro with same name / brackets | PSE could **not** reproduce |

Secondary: bar hover/count vs Data View row list can disagree (similar
to INTRISK-101358). Hover saying `1` while opening the `2` bar is
likely a hover/selection mismatch in the recording, not the core bug.

---

## How the two paths differ

### Chart (works)

`ChartStore._makeTraversalOutputs` **injects** dashboard env filters
that are not already chart pivots/aggs: it looks up the column in
`_dsColumnsMap`, then adds `hidden` + `filter` annotations before the
chart query runs.

```1521:1541:lib/src/reports/chart/chart_store.dart
    for (DisplayFilter? filter in (_environmentFilter?.propFilters
            ?.where((pf) => pf.dataSourceId == dataSourceId)
            .map((pf) => pf.displayFilter)
            .toList() ??
        [])
      ..addAll(filtersList)) {
      String key = '${filter!.stepLabel}.${filter.propertyName}';
      if (usedOutputs.contains(key)) {
        continue;
      }
      gdt.TraversalOutput? output = _dsColumnsMap[key]?.output;
      if (output == null) {
        inactiveFilters.add(filter);
        continue;
      }

      output = addAnnotation(output, ColAnnotation.hidden);
      output = addAnnotation(output, ColAnnotation.filter, filter.values);

      newOutputs.add(output);
    }
```

`getQueries()` then also calls `applyEnvFilter` on those outputs. By
then `annotationValues` is already populated, so the second pass can
intersect/apply.

### Drill-down (broken for this column)

`triggerDrillDown` rebuilds outputs from **raw** `_dsColumnsMap`
(click/pivot filters + display filters only), then relies on
`applyEnvFilter` alone — it does **not** re-run the injection loop
above.

```2063:2095:lib/src/reports/chart/chart_store.dart
  void triggerDrillDown(Map<String, List<gdt.Value>?> filteredCols, Set<String> pivots) {
    List<gdt.TraversalOutput>? outputs = _dsColumnsMap.values.map((ColumnInfo ci) {
      // ... add click / display filters via addAnnotation ...
      return o;
    }).toList();
    // ...
    outputs = applyEnvFilter(environmentFilter, dataSourceId, outputs);
    _events.drillDownTriggered(DrillDownPayload(_title, pivots, dataSourceId, outputs, _dsQuery), dispatchKey);
  }
```

### The smoking gun in `applyEnvFilter`

```74:91:lib/src/reports/dashboard/rich/filter/filter_util.dart
  for (EnvPropertyFilter filter in propFilters) {
    for (int i = 0; i < outputsLength; i++) {
      gdt.TraversalOutput o = newOutputs[i];
      if (o.annotationValues == null) continue;

      if (o.stepLabel == filter.displayFilter?.stepLabel && o.propertyName == filter.displayFilter?.propertyName) {
        List<gdt.Value>? filterValues = o.annotationValues![ColAnnotation.filter];
        if (filterValues == null) {
          filterValues = filter.displayFilter?.values;
        } else {
          filterValues = filterValues.where((v) => filter.displayFilter?.values?.contains(v) == true).toList();
        }

        if (filterValues?.isNotEmpty == true) {
          newOutputs[i] = addAnnotation(o, ColAnnotation.filter, filterValues);
        }
      }
    }
  }
```

If a matching column has `annotationValues == null`, the env filter is
**skipped entirely**. The column still appears in drill-down (it came
from `_dsColumnsMap`), but **without** a filter annotation — exactly
the SUPP screenshots.

That null check was added in 2019 ("Fix some run time errors") to
avoid NPE when reading `annotationValues[...]`. It accidentally became
a behavioral gate: env filters only apply to columns that already have
an `annotationValues` map.

- Empty map `{}` → check passes → filter applied.
- `null` → skipped → bug.

That alone explains hard-to-reproduce behavior: repro often works if
the column already had any annotation history (sort, prior filter,
agg remnant, etc.).

---

## Relation to INTRISK-101358

INTRISK-101358: dashboard + chart selection filters updated the **bar
count**, but Data View **list** ignored the extra dashboard layer
until only one filter layer remained.

**Fix (PR #95167, graph_app 10.4.12):**
`triggerSelectVertexList` stopped restricting outputs to
`activeFields ∪ displayFilteredFields` and included all columns so
`applyEnvFilter` could see the dashboard-filtered field.

That fix addresses "column not in outputs at all." It does **not**
fix "column in outputs but `annotationValues == null`."

SUPP-78503 primary symptom is the latter (column present, filter
missing). Secondary count-vs-list mismatch is the same family as
101358 and can still happen when env filter application on the Data
View / drill-down path is incomplete.

**Conclusion:** related failure mode (env filter not fully transferred
off the chart path), not a clean regression of the 101358 change.
Brackets in the display label are a red herring; matching uses
`stepLabel` + `propertyName`, while UI labels use
`title (reportTitle)` from `OutputOption`.

---

## What else to check (repro / confirm)

1. **On the customer chart**, is "Country Reporting" a pivot/axis
   field, or only a dashboard filter on a non-mapped column?
   - Only-dashboard-filter + never annotated → highest risk.

2. **In DevTools / breakpoint** on drill-down open:
   - `environmentFilter.propFilters` entry for Country:
     `dataSourceId`, `stepLabel`, `propertyName`, values.
   - Matching output in the payload: is
     `annotationValues == null` before `applyEnvFilter`?
   - After `applyEnvFilter`: still no `filter` annotation?

3. **Compare a "working" dashboard filter** on the same drill-down:
   does that column already have `annotationValues` (even `{}`) or was
   it also a bar-click pivot (got `addAnnotation` from
   `filteredCols`)?

4. **Confirm report-level vs dashboard-level:**
   - Report filter → `displayFilters` path → `addAnnotation` → works.
   - Dashboard-only → depends on `applyEnvFilter` → fails when null.

5. **Data View (list) vs Drill-down:** both call into chart store
   paths that end in `applyEnvFilter`. If Country is missing from
   active chart fields, Data View may still need the 101358 "include
   all columns" behavior **and** a correct `applyEnvFilter`.

6. **Two widgets / two reports on the dashboard:** ticket notes one
   chart works and one does not — confirm Country filter's
   `dataSourceId` matches the broken widget's report id
   (`applyEnvFilter` filters by `dataSourceId`).

7. **Do not over-index on brackets** in
   "Country Reporting (Country Reporting)" — that is normal
   `title (reportTitle)` (or type/property naming), not the matcher.

---

## Likely fix direction (for a follow-up bug)

1. **Fix `applyEnvFilter`:** remove the `annotationValues == null`
   continue; treat null like "no existing filter values" and call
   `addAnnotation` (null-safe read via `?.`).

2. **Align drill-down with chart:** in `triggerDrillDown` (and Data
   View path), inject env-filter columns the same way
   `_makeTraversalOutputs` does when the column is missing from the
   output list — do not rely on `applyEnvFilter` alone.

3. **Unit tests:** env filter on a column with `annotationValues ==
   null`; env filter on a column absent from active chart fields;
   multi-filter dashboard + bar click → drill-down shows all filter
   chips and filtered rows.

---

## Suggested reply to PSE

> Thanks for the thorough isolation. This does not look like a
> brackets/naming issue, and it is only loosely related to
> INTRISK-101358 (same area: dashboard filters not fully applied off
> the chart path).
>
> Chart rendering **injects** dashboard filters into the query;
> drill-down rebuilds columns and runs `applyEnvFilter`, which
> **skips** any matching column whose `annotationValues` is null. That
> yields: filter chip on dashboard + correct bars, but drill-down
> shows the Country column with **no** filter and other countries
> visible. Report-level filters work because they go through
> `displayFilters` / `addAnnotation`.
>
> To confirm on the customer dashboard: when opening drill-down,
> check whether the Country output has `annotationValues == null`
> while other working dashboard-filter columns do not. Repro tip: use
> a column that is **only** a dashboard filter (not a chart
> axis/pivot) and has never had a report filter/sort applied.
