# Heat map report — Option B implementation

Custom Over React 5×5 risk heat map for ERM/ORM reports. Replaces
overlapping scatter points with box-based cells. Highcharts is **not** used
for rendering.

**Target:** graph_app chart module, behind a LaunchDarkly flag, before
September Amplify.

**Why custom UI:** The design needs in-cell risk pills, `+N more`, rich Impact
axis labels, Inherent/Residual toggle, and a detail side panel. Highcharts
heat maps only support colored value cells and do not fit this UX.

---

## 1. Architecture

```text
ChartModule
  ChartStore  ── builds HeatMapModel from report rows
  ChartContentView
    └─ HeatMapContentView          ← NEW (when PlotType.heatmap + LD on)
         ├─ HeatMapToolbar         Inherent | Residual
         ├─ HeatMapGrid            5×5 cells + axis labels
         │    └─ HeatMapCell       severity label, pills, +N more
         └─ HeatMapDetailPanel     selected cell list
```

When the flag is off or the plot type is not heatmap, existing scatter / pie /
bar paths are unchanged.

---

## 2. Data model (new)

```dart
class HeatMapAxisLevel {
  final int value;          // 1..5
  final String label;       // "High - 4"
  final String? description;// "$1M-$5M, 1-3 days..."
}

class HeatMapRiskItem {
  final String id;          // vertex / row id
  final String displayId;   // "ERM 031"
  final String title;       // "Competitive Pressure"
}

class HeatMapCell {
  final int impact;         // 1..5 (y)
  final int likelihood;     // 1..5 (x)
  final int score;          // impact * likelihood (or configured)
  final String severityLabel; // "Very High - Escalate to ELT - 12"
  final String colorHex;
  final List<HeatMapRiskItem> items;
}

class HeatMapModel {
  final List<HeatMapAxisLevel> impactLevels;     // top→bottom: 5..1
  final List<HeatMapAxisLevel> likelihoodLevels; // left→right: 1..5
  final List<HeatMapCell> cells; // always 25
  final HeatMapMode mode; // inherent | residual
}
```

### Bucketing (in `ChartStore`)

1. Resolve Impact / Likelihood fields for the current mode.
2. For each report row, parse discrete ints (clamp or drop outside 1–5 —
   product decision).
3. Group into `Map<(impact, likelihood), List<HeatMapRiskItem>>`.
4. Always emit **25 cells** (empty cells still get severity color/label).
5. Map score → severity label + color from preset bands.

---

## 3. Feature flag and plot type

**Flag:** `grc-enable-heatmap-report` (same pattern as
`enable-uce-bar-in-graph-app`).

**Files:**

- `lib/src/feature_flags.dart` — helper getter
- `lib/src/reports/chart/chart_module/models/plot_type.dart` — add `heatmap`
- Chart type picker / data pane — show Heat map only if flag is on
- `chart_content_view.dart` — branch to `HeatMapContentView` instead of
  UCE / Highcharts renderer
- `ChartLoader.forChartType` — `NoopChartLoader` for heatmap (no Highcharts
  load)

```dart
// chart_content_view.dart (concept)
if (store.plotType == PlotType.heatmap && isHeatmapEnabled()) {
  return HeatMapContentView()..store = store..actions = actions;
}
```

---

## 4. Component breakdown

| Component | Responsibility |
| --- | --- |
| `HeatMapContentView` | Layout: toolbar + grid + optional panel; wires store |
| `HeatMapToolbar` | Inherent / Residual toggle → `actions.setHeatMapMode` |
| `HeatMapGrid` | CSS grid: axis column + 5×5; maps `HeatMapModel.cells` |
| `HeatMapAxisLabel` | Impact text + description (left); Likelihood labels (bottom) |
| `HeatMapCell` | Background color, severity header, up to N pills, `+N more`, selected border |
| `HeatMapDetailPanel` | Header (severity + score), Impact/Likelihood line, accordion list; close clears selection |

**Suggested paths:**
`lib/src/reports/chart/components/heatmap/`

**Layout sketch:**

```text
[ Inherent | Residual ]
┌──────────┬─────────────────────────────┬─────────────┐
│ Impact   │  5×5 HeatMapCell grid       │ DetailPanel │
│ labels   │                             │ (if selected│
│          │  Likelihood labels ↓        │  cell)      │
└──────────┴─────────────────────────────┴─────────────┘
```

Use CSS Grid / flex (`web_skin_dart` Block/VBlock). Give cells a min-height so
pills do not crush empty cells.

---

## 5. Store and actions

**New on `ChartActions`:**

- `setHeatMapMode(HeatMapMode)`
- `selectHeatMapCell(HeatMapCell?)`
- `openHeatMapRisk(HeatMapRiskItem)` (or reuse drill-down)

**New on `ChartStore`:**

- `heatMapMode`, `selectedHeatMapCell`, `heatMapModel`
- `_buildHeatMapModel()` after data refresh (alongside
  `_buildDataItemsForPlotType`)
- Selection does **not** call Highcharts drill-down with a single point
  index; cell selection filters by Impact + Likelihood values

**Cell → detail panel:** set `selectedHeatMapCell`, `trigger()`.

**Risk row click:**

- Prefer existing vertex navigation / `triggerSelectVertexList` / report
  drill-down with filters:
  `Impact == cell.impact` AND `Likelihood == cell.likelihood` (and
  optionally risk id).
- If `items.length > 50`: panel shows first 50 + “View all” → full filtered
  list page (per kickoff meeting).

**Pill limit in cell:** e.g. show 2–3 pills, then `+N more` (click selects
cell / opens panel). Exact N = design.

---

## 6. Format / config persistence

Extend `chartFormat` / report view format map (same as scatter):

```dart
{
  'type': 'heatmap',
  'heatMapMode': 'residual',          // or inherent
  'impactFieldInherent': '...',
  'likelihoodFieldInherent': '...',
  'impactFieldResidual': '...',
  'likelihoodFieldResidual': '...',
  'labelField': '...',                // ERM id / name for pills
  'titleField': '...',
  'gridSize': 5,                      // fixed for v1
  'axisLevels': { ... },              // optional custom labels/descriptions
  'severityBands': [                   // color + label presets
    { 'minScore': 1,  'maxScore': 4,  'label': 'Very Low', 'color': '#...' },
    { 'minScore': 5,  'maxScore': 9,  'label': 'Low - Monitor', 'color': '#...' },
    // ...
    { 'minScore': 20, 'maxScore': 25, 'label': 'Critical - Board ATTN', 'color': '#...' },
  ],
}
```

Edit panel: when `PlotType.heatmap`, show heatmap-specific format UI (fields,
bands, axis copy) instead of scatter marker options. UCE format panel can stay
unused (`Noop` / hide).

---

## 7. Inherent vs Residual

Toggle only switches **which field pair** is used for X/Y (and maybe title),
then rebuilds `HeatMapModel`. Same 5×5 chrome.

Wire in data pane:

- Required: Impact + Likelihood for active mode
- Optional: display id + title fields for pills

`missingChartInputs()` should require those fields for heatmap.

---

## 8. Styling / colors

- Cell background from severity band (diagonal green → yellow → red → maroon
  in mockup).
- Selected: blue border (design token if available).
- Pills: white chips, truncate long names.
- Empty cells: still colored by score band, no pills.
- SCSS under existing chart/report styles — scoped class names like
  `.heat-map-grid`, `.heat-map-cell--selected`.

Score default: `impact * likelihood` (1–25). Band thresholds configurable via
format presets.

---

## 9. What not to use from Highcharts

| Existing piece | Heatmap behavior |
| --- | --- |
| `ScatterChartLoader` / `loadScatter` | Skip — `NoopChartLoader` |
| `_chartModule.components.content` | Don’t use as renderer |
| PNG via `_chartModule.components.contentSvg` | Separate later (html2canvas / defer export) |
| Point-index `drillDown` | Replace with cell filter + risk id |

---

## 10. Suggested delivery slices

1. **Flag + `PlotType.heatmap` + empty grid shell** in `ChartContentView`
2. **Bucketing + 25 colored cells** + severity labels (no pills yet)
3. **Pills + `+N more`**
4. **Selection + detail panel**
5. **Inherent/Residual toggle + field mapping**
6. **Format presets** (colors/labels/axis text) + persist
7. **Risk click → navigate/drill-down** + >50 “View all”
8. **Tests:** store bucketing, band colors, panel list; smoke on ERM workspace

---

## 11. Key files to touch

| Area | Files |
| --- | --- |
| Type / flag | `plot_type.dart`, `feature_flags.dart` |
| Data | `chart_store.dart`, `chart_actions.dart`, new `heatmap_models.dart` |
| Render | `chart_content_view.dart`, new `components/heatmap/*` |
| Edit UX | `chart_data_pane.dart`, `chart_format_pane.dart` / edit panel |
| Loader | `chart_loader.dart` → noop for heatmap |
| Format keys | `chart_format.dart` (heatmap constants) |
| Tests | `test/unit/common/chart/...` |

---

## 12. Decisions to lock before coding

1. Outside 1–5: drop vs clamp vs “Other” row/col (mockup implies drop/clamp
   only).
2. Max pills per cell before `+N more`.
3. Panel list hard cap 50 — confirm.
4. Axis descriptions: hardcoded EN preset vs fully configurable in format.
5. PNG export in v1 or later.

---

## Summary

Treat heatmap as a new chart view parallel to scatter, backed by the same
report data pipeline, with all matrix UX in Over React. Highcharts stays out
of the render path.
