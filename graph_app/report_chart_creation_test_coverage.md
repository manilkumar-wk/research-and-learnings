# Report chart creation — test coverage

**Feature:** the `Chart` dropdown on the Report toolbar (Bar / Pie / Scatter).
**Repo:** [Workiva/graph_app](https://github.com/Workiva/graph_app)
**Audience:** QA and dev working on the reports experience.
**Prepared:** 2026-08-03.

> **Note on percentages.** Numbers in this document are **feature/scenario
> coverage** (what portion of the user-facing capability has a corresponding
> test), not line coverage. See section 10 for how to generate real line
> coverage locally.

---

## Executive Summary

> **Q:** *The Pres team recently helped to create the Builder Visual
> options for Charts in GRC Workspaces. What type of test coverage
> (unit, functional, etc.) currently exists for these features?*

We have **four layers** of test coverage for the chart creation features
today:

### 1. Unit Tests — Strong coverage
- **Toolbar dropdown click** — Tests that clicking Bar/Pie in the Report
  toolbar dispatches the correct `PlotType`. Scatter click test is
  **missing**.
- **Report store integration** — Tests that creating a chart from a
  table copies columns, filters, and assigns the correct `PlotType` for
  **all 3 types** (bar, pie, scatter).
- **Chart loader factory** — Tests that `ChartLoader.forChartType()`
  routes to the correct renderer for **all 3 types**.
- **Per-type loader & writer tests** — Deep config tests for **all 3
  types** (Highcharts options, axis setters, color/gradient, labels,
  legend).
- **Insert widget (dashboard)** — Tests that the dashboard
  "Insert Widget" chart-type dropdown dispatches correctly for **all 3
  types**.
- **Shared chart surface** — 60+ tests covering chart store, edit panel,
  format pane, preview, image export, event listeners, and content view
  rendering (bar and pie only — **scatter render test missing**).

### 2. Functional (E2E / Puppeteer) Tests — Limited
- **Report chart creation** — One scenario: create a **Bar** chart, set
  axes, save as view, apply filter. **No Pie or Scatter scenario.**
- **Dashboard chart widget** — One scenario: insert a new **Bar** chart
  widget. **No Pie or Scatter scenario.**

### 3. What's NOT Covered
- **No dedicated tests** for: chart context menus (right-click
  Export/Drill Down), `HighchartContentView` (the Highcharts rendering
  wrapper), `ChartLabelOptions`, `ChartDataPane`, chart undo/redo
  operations (`SetPlotTypeOp`/`SetDataSourceOp`), or `panel_options`.
- **No signal tests** specific to chart creation were found.

### 4. At a Glance

| Test type | Bar | Pie | Scatter |
|-----------|:---:|:---:|:-------:|
| Unit — toolbar click | ✅ | ✅ | ❌ |
| Unit — store / loader / writer | ✅ | ✅ | ✅ |
| Unit — component render | ✅ | ✅ | ❌ |
| Unit — insert widget | ✅ | ✅ | ✅ |
| Functional E2E (report) | ✅ | ❌ | ❌ |
| Functional E2E (dashboard) | ✅ | ❌ | ❌ |
| Context menu / undo-redo | ❌ | ❌ | ❌ |

**Bottom line:** Unit-level coverage is solid for the core chart pipeline
(loaders, writers, store) across all three types. The main gaps are
(1) Scatter at the toolbar-click and component-render layers, (2) Pie
and Scatter in functional E2E tests, and (3) no dedicated tests for the
context menu, rendering wrapper, or undo/redo operations. We've
documented **13 specific gaps** with prioritized effort estimates in
[section 12](#12-gaps-and-suggested-new-tests) below.

---

## 1. TL;DR

| Layer                                         | Bar | Pie | Scatter | Layer % |
| --------------------------------------------- | :-: | :-: | :-----: | :-----: |
| Toolbar dropdown click (unit)                 |  ✅  |  ✅  |    ❌    |   67 %  |
| Report store — create from table (unit)       |  ✅  |  ✅  |    ✅    |  100 %  |
| Report store — create draft views (unit)      |  ✅  |  ❌  |    ❌    |   33 %  |
| Insert widget — chart type selection (unit)   |  ✅  |  ✅  |    ✅    |  100 %  |
| Chart loader factory routing (unit)           |  ✅  |  ✅  |    ✅    |  100 %  |
| Renderer / loader deep unit tests             |  ✅  |  ✅  |    ✅    |  100 %  |
| Chart writer unit tests                       |  ✅  |  ✅  |    ✅    |  100 %  |
| Chart content view render (unit)              |  ✅  |  ✅  |    ❌    |   67 %  |
| Chart edit panel data pane (unit)             |  ✅  |  ✅  |    ❌    |   67 %  |
| Chart context menu (unit)                     |  ❌  |  ❌  |    ❌    |    0 %  |
| Functional page-object helper                 |  ✅  |  ❌  |    ❌    |   33 %  |
| End-to-end functional scenario (report)       |  ✅  |  ❌  |    ❌    |   33 %  |
| End-to-end functional scenario (dashboard)    |  ✅  |  ❌  |    ❌    |   33 %  |
| **Row coverage**                              | **13/13 · 100 %** | **7/13 · 54 %** | **4/13 · 31 %** | — |

**Overall scenario coverage: 24 / 39 = 62 %.**

Gaps concentrated in **Scatter** at toolbar-click and component layers,
in **Pie / Scatter** at the functional layer, and in **Chart context
menu** across all types.

---

## 2. Feature under test

The Report toolbar exposes a `Chart` dropdown with three menu items:

- **Bar**  → dispatches `createNewChart(PlotType.bar)`
- **Pie**  → dispatches `createNewChart(PlotType.pie)`
- **Scatter** → dispatches `createNewChart(PlotType.scatter)`

Production code lives under
[`lib/src/reports/report_rich/report_toolbar.dart`](https://github.com/Workiva/graph_app/blob/master/lib/src/reports/report_rich/report_toolbar.dart)
and
[`lib/src/reports/chart/`](https://github.com/Workiva/graph_app/tree/master/lib/src/reports/chart).

Enum type used to distinguish chart types
([`lib/src/reports/chart/chart_module/models/plot_type.dart`](https://github.com/Workiva/graph_app/blob/master/lib/src/reports/chart/chart_module/models/plot_type.dart)):

```dart
enum PlotType { bar, pie, scatter }
```

Automation test-ids advertised by the toolbar (used by every unit /
functional test in this document):

| Constant                                       | DOM `data-test-id`                                       |
| ---------------------------------------------- | -------------------------------------------------------- |
| `createChartDropdownTestId`                    | `graph.components.report-toolbar.create-chart-dropdown`  |
| `createBarChartDropdownOptionTestId`           | `…create-bar-chart-option`                               |
| `createPieChartDropdownOptionTestId`           | `…create-pie-chart-option`                               |
| `createScatterPlotDropdownOptionTestId`        | `…create-scatter-plot-option`                            |

---

## 3. Unit tests — toolbar create flow

### 3.1 `test/unit/common/experiences/graph_report/report_toolbar_test.dart`

Shared helper used by every chart-type click test:

```dart
testCreateChartClick(String testId, PlotType type) async {
  var createChartDropdown = wsd_utils.getByTestId(
    renderedToolbar, ReportToolbarComponent.createChartDropdownTestId);
  var createChartDropdownDom =
    wsd_utils.getComponentRootDomByTestId(createChartDropdown, 'wsd.hitarea');
  expect(createChartDropdown, isNotNull);

  wsd_utils.click(createChartDropdownDom);         // open dropdown
  await nextTick();

  var createChartMenuItem = wsd_utils.getByTestId(renderedToolbar, testId);
  expect(createChartMenuItem, isNotNull);

  Completer actionListener = Completer();
  await actions.createNewChart.dispose();
  actions.createNewChart.listen(actionListener.complete);

  wsd_utils.click(                                 // click menu item
    wsd_utils.getComponentRootDomByTestId(createChartMenuItem, 'wsd.hitarea'));

  expect(await actionListener.future, type);       // payload check
}
```

| Test name                            | PlotType asserted | Coverage |
| ------------------------------------ | :---------------: | :------: |
| `Create bar chart button click`      | `PlotType.bar`    | ✅ |
| `Create pie chart button click`      | `PlotType.pie`    | ✅ |
| *(missing)* `Create scatter chart button click` | `PlotType.scatter` | ❌ **GAP** |

**What each test asserts:**

- The `create-chart-dropdown` button renders and is clickable.
- After opening the dropdown, the correct menu item renders.
- Clicking the menu item dispatches `actions.createNewChart` exactly once
  with the correct `PlotType` payload.

**What is NOT asserted at this layer:**

- Whether the correct chart type is subsequently created / rendered — that
  is covered by `ChartLoader.forChartType` (§3.2) and the loader tests
  (§5).
- Whether the toolbar is disabled when the report is dirty — separate
  suite `with store.isReportDirty == true` covers save/discard flows but
  does not re-test the chart dropdown for each type.

### 3.2 `test/unit/common/chart/chart_module/loader/chart_loader_test.dart`

Verifies the factory that selects the concrete renderer for a given
`PlotType`:

```dart
group('ChartLoader', () {
  test('forChartType', () {
    when(() => store.plotType).thenReturn(PlotType.pie);
    expect(ChartLoader.forChartType(api: api, store: store),
      isA<PieChartLoader>());

    when(() => store.plotType).thenReturn(PlotType.scatter);
    expect(ChartLoader.forChartType(api: api, store: store),
      isA<ScatterChartLoader>());

    when(() => store.plotType).thenReturn(PlotType.bar);
    expect(ChartLoader.forChartType(api: api, store: store),
      isA<NoopChartLoader>());
  });
});
```

Bar routes to `NoopChartLoader` because bar rendering uses the default
Highcharts pipeline; pie and scatter each have their own loaders.

---

## 4. Unit tests — report store chart creation

### 4.1 `test/unit/common/experiences/graph_report/report_store_test.dart`

These tests exercise the **store-level** chart creation flow — the layer
between the toolbar click and the chart module. They verify that the
report store correctly creates draft views, copies table columns/filters
to the chart, and assigns the correct `PlotType`.

| Test name                        | PlotType(s)  | What it covers |
| -------------------------------- | :----------: | -------------- |
| `create draft views`             | Bar only     | Dispatches `createNewChart(PlotType.bar)` twice interleaved with table creates. Asserts 4 drafts created with correct `viewType` (`chart:bar` / `table`), `isDirty`, and `isDraftView` flags. Tests switching between drafts via `setReportViewId`, discarding a draft, and saving a draft layout via `saveLayoutAs`. |
| `create bar chart from table`    | Bar          | Calls `chartFromLayout(CreateChartFromLayoutParams(viewId, PlotType.bar))`. Asserts table outputs and filters are copied to chart `priorityColumnsMap` and `displayFilters`, and `plotType` is `PlotType.bar`. |
| `create pie chart from table`    | Pie          | Same as above with `PlotType.pie`. |
| `create scatter plot from table` | Scatter      | Same as above with `PlotType.scatter`. |

**Shared helper:**

```dart
Future<void> _testCreateChartFromTable(PlotType plotType) async {
  await setUpReportStore();
  await actions.setReportViewId(reportView1VertexId);
  // ... sets up table with 3 TraversalOutputs ...
  await actions.chartFromLayout(
    CreateChartFromLayoutParams(reportView1VertexId, plotType));

  expect(store!.outlineInfo.drafts, hasLength(1));
  await actions.setReportViewId(store!.outlineInfo.drafts.first.id!);

  expect(store!.chart.store.priorityColumnsMap.length, 3);
  expect(store!.chart.store.displayFilters!.length, 1);
  expect(store!.chart.store.plotType, plotType);
}
```

**Coverage note:** The `create draft views` test only creates `PlotType.bar`
charts. Pie and scatter draft creation at the store level rely on the
same code path, but there is **no explicit test asserting `viewType`
equals `chart:pie` or `chart:scatter`** for drafts.

---

## 5. Unit tests — chart loader (per-type render config)

### 5.1 `test/unit/common/chart/chart_module/loader/bar_loader_test.dart`

| Test           | What it covers |
| -------------- | -------------- |
| `vertical`     | Verifies the full Highcharts bar `Bar` config for a vertical bar chart (series, category axis, value axis, colors, labels, sorting). |
| `horizontal`   | Same, but with the horizontal orientation swap applied. |

### 5.2 `test/unit/common/chart/chart_module/loader/pie_chart_loader_test.dart`

| Test    | What it covers |
| ------- | -------------- |
| `load`  | Given `store.sliceDataItems` and a `chartFormat` map, verifies the built `pie.Pie` config: pie size, animation, ellipsis overflow, root label distance/separator/connector, name/value/percent label flags, slice id/name/value/fillColor, legend enabled + placement. |

### 5.3 `test/unit/common/chart/chart_module/loader/scatter_chart_loader_test.dart`

| Test                                            | What it covers |
| ----------------------------------------------- | -------------- |
| `load`                                          | X-axis/Y-axis min/max/step/title, grid enabled flags, plot marker size, per-point id/name/x/y. |
| `default gradient`                              | `BackgroundFill.threeColorGradient` populates default hex triplet. |
| `no color`                                      | Empty `plotFillColors` list is passed through. |
| `truncates colors to match gradient`            | Two-color gradient trims a three-color input. |
| `add defaults to fill in two color gradient`    | One user color + default fallback fills a two-color gradient. |
| `add defaults to fill in three color gradient`  | One user color + two default fallbacks fills a three-color gradient. |

---

## 6. Unit tests — chart writer (chart-type-specific setters)

These exercise the `ChartWriter` subclasses that translate user format
changes into Highcharts option mutations.

### 6.1 `test/unit/common/chart/chart_module/writer/bar_writer_test.dart`

Groups: `vertical`, `horizontal`, plus shared. Per group:
`setCategoryAxisTitle`, `setCategoryAxisTitleEnabled`, `setValueAxisTitle`,
`setValueAxisTitleEnabled`. Shared: `setSeriesColor`, `setShowLabels`,
`setShowTotalLabels`.

### 6.2 `test/unit/common/chart/chart_module/writer/pie_writer_test.dart`

`setSliceColor`, `setShowLabels`, `setShowName`, `setShowValue`,
`setShowPercent`.

### 6.3 `test/unit/common/chart/chart_module/writer/scatter_writer_test.dart`

Axis setters (`setHideXAxisTitle`, `setHideYAxisTitle`,
`set{X,Y}AxisMinValue`, `set{X,Y}AxisMaxValue`, `set{X,Y}AxisStepValue`,
`set{X,Y}AxisGridEnabled`), data-label setters, marker setters, and
gradient / fill setters (`setDataLabelsPlacement`, `setDataLabelsEnabled`,
`setMarkersEnabled`, `setGradientColors`, `setGradientDirection`,
`setMarkerSize`, `setMarkerFillColor`, `setMarkerShape`).

### 6.4 `test/unit/common/chart/chart_module/writer/legend_writer_test.dart`

`setShowLegend`, `setPlacement`. Shared across chart types.

---

## 7. Unit tests — shared chart surface (post-create)

These do not exercise the **Create** dropdown but cover the chart *after*
it exists (rendering, edit panel, data pane, events, image export). They
apply to all three chart types via the shared code paths.

| File | Purpose |
| ---- | ------- |
| `test/unit/common/chart/chart_store_test.dart` | The workhorse: 60+ tests covering query construction, filter application, pivoting, format map (`makeFormatMap: bar / scatter / pie`), `applyFormat`, drill-down, environment filters, and dozens of individual setters (`setDataSource`, `setPlotType`, `setHideLegend`, `set{X,Y,T,Fill,Shape,Size}Mapping`, etc.). |
| `test/unit/common/chart/chart_module_test.dart` | Module lifecycle (`suspend / resume`, `components exist`), api setters, default graph chart model. |
| `test/unit/common/chart/chart_util_test.dart` | `exportChartAsPNG`. |
| `test/unit/common/chart/chart_module/chart_event_listener_test.dart` | Event routing: legend, pie label / fill, bar axis + labels + colors. |
| `test/unit/common/chart/components/chart_content_view_test.dart` | Renders pie chart, renders bar chart, missing data state, missing report state, loading state, null result state. |
| `test/unit/common/chart/components/chart_edit_panel_test.dart` | Edit panel render, pie data pane, bar data pane, bar type switch, priority-columns handling. |
| `test/unit/common/chart/components/chart_data_view_test.dart` | Data-view component renders. |
| `test/unit/common/chart/components/chart_table_view_test.dart` | Table-view component renders. |
| `test/unit/common/chart/components/chart_preview_test.dart` | Preview modal renders (default + debug). |
| `test/unit/common/chart/components/chart_format_pane_test.dart` | Format pane renders, axis title options. |
| `test/unit/common/chart/components/chart_image_modal_test.dart` | Image modal renders. |

**Coverage note:** Scatter is under-covered in this shared surface too —
`chart_content_view_test.dart` has explicit `render pie chart` and
`render bar chart` tests but **no** `render scatter chart` test.
`chart_edit_panel_test.dart` covers pie and bar data panes but not scatter.

---

## 8. Unit tests — insert widget chart creation (dashboard context)

### 8.1 `test/unit/common/insert_widget/insert_widget_view_grpc_test.dart`

These tests exercise the **dashboard Insert Widget** flow, which lets
users create a new chart widget from the dashboard. The insert widget
dialog has a chart tab with a chart-type dropdown (Bar / Pie / Scatter).

| Test name                                              | PlotType asserted | Coverage |
| ------------------------------------------------------ | :---------------: | :------: |
| `default rendering`                                    | —                 | ✅ Asserts chart tab, table tab, chart types menu, insert-chart-list, cancel button, and disabled insert button all render. |
| `onCreateNew called on create bar chart selection`     | `PlotType.bar`    | ✅ |
| `onCreateNew called on create pie chart selection`     | `PlotType.pie`    | ✅ |
| `onCreateNew called on create scatter plot selection`  | `PlotType.scatter` | ✅ |

**What each chart-type test asserts:**

- Sets up a list of reports and a `dataSourceId`.
- Opens the chart-types dropdown via `chartDropdown.showDropdownMenu()`.
- Clicks the specific chart-type option (e.g.
  `insert-widget-create-bar-chart-option`).
- Asserts `onCreateNew` callback was called with `isChart: true`, the
  correct `dataSourceId`, and the correct `PlotType`.

**What is NOT asserted:**

- Post-creation rendering of the chart in the dashboard widget — that is
  covered by the dashboard experience functional test (§9.2) and the
  chart content view unit tests (§7).

---

## 9. Functional (browser / Puppeteer) tests

### 9.1 Page objects

`dart_functional/framework/page_objects/reports/report_toolbar.dart`

```dart
/// Create a chart from the current table view.
///
/// [chartType] should be one of 'Bar', 'Pie', or 'Scatter'.
/// [xAxis] and [yAxis] will correspond to the columns in the current report.
Future<void> createBarChart({required String xAxis, required String yAxis}) async {
  _logger.log('Create Bar chart from current table view');
  await _chartDropdown.select();
  await _chartDropdown.selectOptionByText('Bar');

  final chartPanel = ChartPanel(_page, _logger);
  await chartPanel.setXAxis(xAxis);
  await chartPanel.setYAxis(yAxis);
}
```

**Coverage note:** the docstring implies Bar / Pie / Scatter are all
supported but only Bar is implemented. Pie and Scatter callers do not
exist because there is no helper for them.

`dart_functional/framework/page_objects/chart.dart` — `ChartPanel` has
generic `setXAxis` / `setYAxis` helpers used post-create.

### 9.2 End-to-end scenarios — report charts

`dart_functional/test/ci/reports_dashboards/report_experience_test.dart`

The only scenario that exercises chart creation in reports:

| Test                              | Type | What it verifies |
| --------------------------------- | :--: | ---------------- |
| `Create Chart and Apply Filter`   | Bar  | Navigate to a report → open Chart dropdown → click Bar → set X-axis (`Test Status`) + Y-axis (`Tester`) → assert draft view visible → save as new view (`Test Status Chart`) → re-open view → assert X/Y axis titles + X-axis labels → apply filter (`In Progress`, `Not Started`) → assert filtered X-axis labels. |
| *(missing)* Pie scenario          | Pie  | ❌ **GAP** |
| *(missing)* Scatter scenario      | Scatter | ❌ **GAP** |

### 9.3 End-to-end scenarios — dashboard chart widgets

`dart_functional/test/ci/reports_dashboards/dashboard_experience_test.dart`

Tests chart widget creation and insertion in a dashboard context:

| Test                                     | Type | What it verifies |
| ---------------------------------------- | :--: | ---------------- |
| `Insert chart widget from existing view` | —    | Navigates to a dashboard → inserts an existing chart (`Issue Severity`) → asserts chart widget count increases to 1. |
| `Create new chart widget`                | Bar  | Creates a new dashboard → inserts a new bar chart via `insertNewBarChart()` with `chartName: 'By Process'` → asserts chart widget count is 1. |
| *(missing)* Pie dashboard scenario       | Pie  | ❌ **GAP** |
| *(missing)* Scatter dashboard scenario   | Scatter | ❌ **GAP** |

---

## 10. Source files with no dedicated test coverage

The following production source files in `lib/src/reports/chart/` have
**no dedicated test file**. Some are indirectly exercised by other tests
(noted below), but untested behaviour in these files represents a
coverage risk.

| Source file | Purpose | Indirect coverage |
| ----------- | ------- | ----------------- |
| `chart_module/chart_context_menu.dart` | `ChartContextMenuListener` — handles right-click context menus on chart elements (Export as PNG, Drill Down). Listens to pie slice, scatter item, and bar item context-menu/click/double-click events. | Partially by `chart_event_listener_test.dart` (event routing only, not context-menu item creation or `showContextMenu` call). |
| `chart_operations.dart` | `SetDataSourceOp` and `SetPlotTypeOp` — undo/redo operation classes for changing chart data source and plot type. | Indirectly by `chart_store_test.dart` (the store calls these ops, but the undo path is not directly asserted). |
| `components/chart_data_pane.dart` | `ChartDataPaneComponent` — renders the chart axis/mapping selection UI with plot-type icon, bar orientation buttons, and column mappings. | Partially by `chart_edit_panel_test.dart` (tests `pie chart data pane` and `bar chart data pane` groups, but no `scatter chart data pane` group). |
| `components/chart_label_options.dart` | `ChartLabelOptionsComponent` — manages slice label field selection (add/remove/reorder) for pie charts. | None found. |
| `components/highchart_content_view.dart` | `HighchartContentViewComponent` — the actual Highcharts rendering wrapper that receives plot type, data items, axis config, colors, and format options and renders the chart DOM. | Indirectly by `chart_content_view_test.dart` (renders `ChartContentView` which internally uses this, but the Highcharts interaction is not asserted). |
| `components/chart_edit_group.dart` | `ChartEditGroupComponent` — renders drag-and-drop mapping groups (X, Y, Fill, etc.) in the chart edit panel. | Partially by `chart_edit_panel_test.dart`. |
| `components/chart_edit_group_item.dart` | `ChartEditGroupItemComponent` — individual draggable items within an edit group. | Partially by `chart_edit_panel_test.dart`. |
| `chart_module/chart_format.dart` | Chart format constants and default format map builders. | Indirectly by `chart_store_test.dart` (`makeFormatMap` tests). |
| `chart_module/panel_options.dart` | Chart panel options configuration for the Highcharts module. | None found. |

---

## 11. How to generate real line-coverage numbers

Feature-level percentages above tell you which scenarios exist. If the QA
team wants **line coverage** for the chart create surface:

### 11.1 Unit-test line coverage

From the repo root:

```bash
dart pub global activate coverage

dart test --coverage=coverage \
  test/unit/common/experiences/graph_report/report_toolbar_test.dart \
  test/unit/common/experiences/graph_report/report_store_test.dart \
  test/unit/common/chart/ \
  test/unit/common/insert_widget/insert_widget_view_grpc_test.dart

dart pub global run coverage:format_coverage \
  --lcov --in=coverage --out=coverage/lcov.info \
  --packages=.dart_tool/package_config.json \
  --report-on=lib

# Narrow the report to the chart create surface:
lcov --extract coverage/lcov.info \
  'lib/src/reports/report_rich/report_toolbar.dart' \
  'lib/src/reports/report_rich/report_store.dart' \
  'lib/src/reports/chart/chart_module/models/plot_type.dart' \
  'lib/src/reports/chart/chart_module/loader/*' \
  'lib/src/reports/chart/chart_module/writer/*' \
  'lib/src/reports/chart/chart_module/chart_context_menu.dart' \
  'lib/src/reports/chart/chart_operations.dart' \
  'lib/src/reports/chart/components/*' \
  -o coverage/chart-create.info

genhtml coverage/chart-create.info -o coverage/html
open coverage/html/index.html
```

### 11.2 Functional-test coverage

Functional tests run in a real browser via Puppeteer; standard Dart
coverage does not apply. Options:

- Wire Chrome V8 coverage inside the fixture via
  `page.coverage.startJSCoverage()`. Not currently instrumented in this
  repo.
- Track scenario coverage instead: number of user paths covered vs. total
  user paths.

Local run:

```bash
cd dart_functional
dart run test test/ci/reports_dashboards/report_experience_test.dart
dart run test test/ci/reports_dashboards/dashboard_experience_test.dart
```

---

## 12. Gaps and suggested new tests

| # | Gap | Suggested fix | Effort |
| :-: | ---- | ---- | :-: |
| 1 | No unit test for **Scatter** toolbar dropdown click. | Add one `test('Create scatter chart button click', …)` in `report_toolbar_test.dart` mirroring the bar/pie tests. | ~15 min |
| 2 | No **render scatter chart** test in `chart_content_view_test.dart`. | Add a companion to `render pie chart` / `render bar chart`. | ~30 min |
| 3 | No **scatter data pane** test in `chart_edit_panel_test.dart`. | Mirror the existing `pie chart data pane` / `bar chart data pane` tests. | ~30 min |
| 4 | No dedicated test for **`ChartContextMenuListener`** (`chart_context_menu.dart`). | Create `test/unit/common/chart/chart_module/chart_context_menu_test.dart`. Should test: (a) right-click on pie slice shows "Export as PNG" + "Drill Down" menu items when `canSaveAs` and `canDrilldown` return true; (b) only "Export as PNG" shows when `canDrilldown` is false; (c) only "Drill Down" shows when `canSaveAs` is false; (d) double-click on scatter item triggers `onDrilldown` with correct `itemId`; (e) single-click on pie slice triggers `onDrilldown` with `isDrilldown: false`; (f) context menu on bar item triggers correctly. | ~1–2 hr |
| 5 | No dedicated test for **`ChartLabelOptionsComponent`** (`chart_label_options.dart`). | Create `test/unit/common/chart/components/chart_label_options_test.dart`. Test: renders label fields, add/remove label field, drag reorder. | ~1 hr |
| 6 | No dedicated test for **`HighchartContentViewComponent`** (`highchart_content_view.dart`). | Create `test/unit/common/chart/components/highchart_content_view_test.dart`. Test: renders with bar/pie/scatter plotType, passes axis titles and colors to Highcharts config, handles context menu blocking, handles drill-down blocking. | ~2 hr |
| 7 | No dedicated test for **`SetPlotTypeOp` / `SetDataSourceOp`** undo path (`chart_operations.dart`). | Add operation-specific tests in `chart_store_test.dart` or a new `chart_operations_test.dart`. Test: `doOperation` returns false for no-op (same type), `undo` restores previous type and mappings, `doOperation` clears mappings on data-source change. | ~1 hr |
| 8 | `create draft views` test only creates **Bar** drafts — no assertion for Pie/Scatter `viewType`. | Add `create pie chart draft view` and `create scatter chart draft view` tests in `report_store_test.dart` asserting `viewType` is `chart:pie` and `chart:scatter` respectively. | ~30 min |
| 9 | Page-object `createBarChart` is Bar-only despite its docstring. | Refactor into a generic `createChart({required String chartType, required String xAxis, required String yAxis})` or add `createPieChart` / `createScatterChart` siblings. | ~30 min |
| 10 | No **Pie** end-to-end report scenario. | Add `Create Pie Chart` scenario in `report_experience_test.dart` reusing the existing Bar fixture and pie-friendly assertions (slice labels, legend). | ~1 hr |
| 11 | No **Scatter** end-to-end report scenario. | Add `Create Scatter Chart` scenario in the same file, asserting axis min/max/step/title after creation. | ~1 hr |
| 12 | No **Pie / Scatter** end-to-end dashboard scenario. | Add scenarios in `dashboard_experience_test.dart` using `insertNewPieChart` / `insertNewScatterChart` page-object helpers (requires gap #9 first). | ~1 hr |
| 13 | No dedicated test for **`panel_options.dart`**. | Create `test/unit/common/chart/chart_module/panel_options_test.dart` or add coverage in `chart_module_test.dart`. | ~30 min |

### 12.1 Code snippet for gap #1

```dart
test('Create scatter chart button click', () async {
  await testCreateChartClick(
    ReportToolbarComponent.createScatterPlotDropdownOptionTestId,
    PlotType.scatter,
  );
});
```

Insert directly after the existing `Create pie chart button click` test
in `test/unit/common/experiences/graph_report/report_toolbar_test.dart`.

### 12.2 Prioritized effort estimate

| Priority | Gaps | Total effort | Rationale |
| :------: | :--: | :----------: | --------- |
| P1 — Quick wins | #1, #2, #3, #8 | ~1.5 hr | Low effort, closes the most visible scatter / draft gaps. |
| P2 — Functional parity | #9, #10, #11, #12 | ~3.5 hr | Extends E2E coverage to Pie and Scatter for both reports and dashboards. |
| P3 — Deeper unit coverage | #4, #5, #6, #7, #13 | ~5.5 hr | Adds dedicated tests for previously untested source files. |

---

## 13. Suggested reading order

- [ ] Read section 2 to learn the automation test-ids exposed by the
      toolbar. These are the primary anchors for any new automation.
- [ ] Skim section 3 — this is the click-flow contract. Any regression in
      the create dropdown will show here first.
- [ ] Read section 4 — new: report store integration tests covering the
      `chartFromLayout` path for all three chart types.
- [ ] Skim section 5 — this is the render-config contract. Any Highcharts
      version bump should be validated against these tests.
- [ ] Read section 8 — new: insert widget tests that exercise the
      dashboard chart-creation flow for all three types.
- [ ] Read section 10 — new: identifies 9 source files with no dedicated
      test coverage.
- [ ] Run the local commands in section 11.1 once to make sure your
      environment can generate line coverage.
- [ ] Review section 12. The 13 gaps make a good backlog for the team
      to pick up, with P1 items achievable in a single sprint.
- [ ] After all gaps are closed, every chart type should be at 100 %
      scenario coverage across all layers.

---

## 14. Reference files (quick jump)

**Production**

- [`lib/src/reports/report_rich/report_toolbar.dart`](https://github.com/Workiva/graph_app/blob/master/lib/src/reports/report_rich/report_toolbar.dart)
- [`lib/src/reports/report_rich/report_store.dart`](https://github.com/Workiva/graph_app/blob/master/lib/src/reports/report_rich/report_store.dart)
- [`lib/src/reports/chart/chart_module/models/plot_type.dart`](https://github.com/Workiva/graph_app/blob/master/lib/src/reports/chart/chart_module/models/plot_type.dart)
- [`lib/src/reports/chart/chart_module/loader/`](https://github.com/Workiva/graph_app/tree/master/lib/src/reports/chart/chart_module/loader)
- [`lib/src/reports/chart/chart_module/writer/`](https://github.com/Workiva/graph_app/tree/master/lib/src/reports/chart/chart_module/writer)
- [`lib/src/reports/chart/chart_module/chart_context_menu.dart`](https://github.com/Workiva/graph_app/blob/master/lib/src/reports/chart/chart_module/chart_context_menu.dart)
- [`lib/src/reports/chart/chart_operations.dart`](https://github.com/Workiva/graph_app/blob/master/lib/src/reports/chart/chart_operations.dart)
- [`lib/src/reports/chart/components/`](https://github.com/Workiva/graph_app/tree/master/lib/src/reports/chart/components)

**Unit tests**

- [`test/unit/common/experiences/graph_report/report_toolbar_test.dart`](https://github.com/Workiva/graph_app/blob/master/test/unit/common/experiences/graph_report/report_toolbar_test.dart)
- [`test/unit/common/experiences/graph_report/report_store_test.dart`](https://github.com/Workiva/graph_app/blob/master/test/unit/common/experiences/graph_report/report_store_test.dart)
- [`test/unit/common/chart/`](https://github.com/Workiva/graph_app/tree/master/test/unit/common/chart)
- [`test/unit/common/insert_widget/insert_widget_view_grpc_test.dart`](https://github.com/Workiva/graph_app/blob/master/test/unit/common/insert_widget/insert_widget_view_grpc_test.dart)

**Functional tests**

- [`dart_functional/framework/page_objects/reports/report_toolbar.dart`](https://github.com/Workiva/graph_app/blob/master/dart_functional/framework/page_objects/reports/report_toolbar.dart)
- [`dart_functional/framework/page_objects/chart.dart`](https://github.com/Workiva/graph_app/blob/master/dart_functional/framework/page_objects/chart.dart)
- [`dart_functional/test/ci/reports_dashboards/report_experience_test.dart`](https://github.com/Workiva/graph_app/blob/master/dart_functional/test/ci/reports_dashboards/report_experience_test.dart)
- [`dart_functional/test/ci/reports_dashboards/dashboard_experience_test.dart`](https://github.com/Workiva/graph_app/blob/master/dart_functional/test/ci/reports_dashboards/dashboard_experience_test.dart)
