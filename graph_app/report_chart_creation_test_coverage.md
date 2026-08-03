# Report chart creation — test coverage

**Feature:** the `Chart` dropdown on the Report toolbar (Bar / Pie / Scatter).
**Repo:** [Workiva/graph_app](https://github.com/Workiva/graph_app)
**Audience:** QA and dev working on the reports experience.
**Prepared:** 2026-08-03.

> **Note on percentages.** Numbers in this document are **feature/scenario
> coverage** (what portion of the user-facing capability has a corresponding
> test), not line coverage. See section 8 for how to generate real line
> coverage locally.

---

## 1. TL;DR

| Layer                                | Bar | Pie | Scatter | Layer % |
| ------------------------------------ | :-: | :-: | :-----: | :-----: |
| Toolbar dropdown click (unit)        |  ✅  |  ✅  |    ❌    |   67 %  |
| Chart loader factory routing (unit)  |  ✅  |  ✅  |    ✅    |  100 %  |
| Renderer / loader deep unit tests    |  ✅  |  ✅  |    ✅    |  100 %  |
| Chart writer unit tests              |  ✅  |  ✅  |    ✅    |  100 %  |
| Functional page-object helper        |  ✅  |  ❌  |    ❌    |   33 %  |
| End-to-end functional scenario       |  ✅  |  ❌  |    ❌    |   33 %  |
| **Row coverage**                     | **6/6 · 100 %** | **4/6 · 67 %** | **3/6 · 50 %** | — |

**Overall scenario coverage: 13 / 18 = 72 %.**

Gaps concentrated in **Scatter** at the unit-click layer and in **Pie /
Scatter** at the functional layer.

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

## 3. Unit tests — create flow

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
  (§4).
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

## 4. Unit tests — chart loader (per-type render config)

### 4.1 `test/unit/common/chart/chart_module/loader/bar_loader_test.dart`

| Test           | What it covers |
| -------------- | -------------- |
| `vertical`     | Verifies the full Highcharts bar `Bar` config for a vertical bar chart (series, category axis, value axis, colors, labels, sorting). |
| `horizontal`   | Same, but with the horizontal orientation swap applied. |

### 4.2 `test/unit/common/chart/chart_module/loader/pie_chart_loader_test.dart`

| Test    | What it covers |
| ------- | -------------- |
| `load`  | Given `store.sliceDataItems` and a `chartFormat` map, verifies the built `pie.Pie` config: pie size, animation, ellipsis overflow, root label distance/separator/connector, name/value/percent label flags, slice id/name/value/fillColor, legend enabled + placement. |

### 4.3 `test/unit/common/chart/chart_module/loader/scatter_chart_loader_test.dart`

| Test                                            | What it covers |
| ----------------------------------------------- | -------------- |
| `load`                                          | X-axis/Y-axis min/max/step/title, grid enabled flags, plot marker size, per-point id/name/x/y. |
| `default gradient`                              | `BackgroundFill.threeColorGradient` populates default hex triplet. |
| `no color`                                      | Empty `plotFillColors` list is passed through. |
| `truncates colors to match gradient`            | Two-color gradient trims a three-color input. |
| `add defaults to fill in two color gradient`    | One user color + default fallback fills a two-color gradient. |
| `add defaults to fill in three color gradient`  | One user color + two default fallbacks fills a three-color gradient. |

---

## 5. Unit tests — chart writer (chart-type-specific setters)

These exercise the `ChartWriter` subclasses that translate user format
changes into Highcharts option mutations.

### 5.1 `test/unit/common/chart/chart_module/writer/bar_writer_test.dart`

Groups: `vertical`, `horizontal`, plus shared. Per group:
`setCategoryAxisTitle`, `setCategoryAxisTitleEnabled`, `setValueAxisTitle`,
`setValueAxisTitleEnabled`. Shared: `setSeriesColor`, `setShowLabels`,
`setShowTotalLabels`.

### 5.2 `test/unit/common/chart/chart_module/writer/pie_writer_test.dart`

`setSliceColor`, `setShowLabels`, `setShowName`, `setShowValue`,
`setShowPercent`.

### 5.3 `test/unit/common/chart/chart_module/writer/scatter_writer_test.dart`

Axis setters (`setHideXAxisTitle`, `setHideYAxisTitle`,
`set{X,Y}AxisMinValue`, `set{X,Y}AxisMaxValue`, `set{X,Y}AxisStepValue`,
`set{X,Y}AxisGridEnabled`), data-label setters, marker setters, and
gradient / fill setters (`setDataLabelsPlacement`, `setDataLabelsEnabled`,
`setMarkersEnabled`, `setGradientColors`, `setGradientDirection`,
`setMarkerSize`, `setMarkerFillColor`, `setMarkerShape`).

### 5.4 `test/unit/common/chart/chart_module/writer/legend_writer_test.dart`

`setShowLegend`, `setPlacement`. Shared across chart types.

---

## 6. Unit tests — shared chart surface (post-create)

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

## 7. Functional (browser / Puppeteer) tests

### 7.1 Page objects

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

### 7.2 End-to-end scenarios

`dart_functional/test/ci/reports_dashboards/report_experience_test.dart`

The only scenario that exercises chart creation:

| Test                              | Type | What it verifies |
| --------------------------------- | :--: | ---------------- |
| `Create Chart and Apply Filter`   | Bar  | Navigate to a report → open Chart dropdown → click Bar → set X-axis (`Test Status`) + Y-axis (`Tester`) → assert draft view visible → save as new view (`Test Status Chart`) → re-open view → assert X/Y axis titles + X-axis labels → apply filter (`In Progress`, `Not Started`) → assert filtered X-axis labels. |
| *(missing)* Pie scenario          | Pie  | ❌ **GAP** |
| *(missing)* Scatter scenario      | Scatter | ❌ **GAP** |

---

## 8. How to generate real line-coverage numbers

Feature-level percentages above tell you which scenarios exist. If the QA
team wants **line coverage** for the chart create surface:

### 8.1 Unit-test line coverage

From the repo root:

```bash
dart pub global activate coverage

dart test --coverage=coverage \
  test/unit/common/experiences/graph_report/report_toolbar_test.dart \
  test/unit/common/chart/

dart pub global run coverage:format_coverage \
  --lcov --in=coverage --out=coverage/lcov.info \
  --packages=.dart_tool/package_config.json \
  --report-on=lib

# Narrow the report to the chart create surface:
lcov --extract coverage/lcov.info \
  'lib/src/reports/report_rich/report_toolbar.dart' \
  'lib/src/reports/chart/chart_module/models/plot_type.dart' \
  'lib/src/reports/chart/chart_module/loader/*' \
  'lib/src/reports/chart/chart_module/writer/*' \
  -o coverage/chart-create.info

genhtml coverage/chart-create.info -o coverage/html
open coverage/html/index.html
```

### 8.2 Functional-test coverage

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
```

---

## 9. Gaps and suggested new tests

| # | Gap | Suggested fix | Effort |
| :-: | ---- | ---- | :-: |
| 1 | No unit test for **Scatter** dropdown click. | Add one `test('Create scatter chart button click', …)` in `report_toolbar_test.dart` mirroring the bar/pie tests. | ~15 min |
| 2 | No **render scatter chart** test in `chart_content_view_test.dart`. | Add a companion to `render pie chart` / `render bar chart`. | ~30 min |
| 3 | No **scatter data pane** test in `chart_edit_panel_test.dart`. | Mirror the existing `pie chart data pane` / `bar chart data pane` tests. | ~30 min |
| 4 | Page-object `createBarChart` is Bar-only despite its docstring. | Refactor into a generic `createChart({required String chartType, required String xAxis, required String yAxis})` or add `createPieChart` / `createScatterChart` siblings. | ~30 min |
| 5 | No **Pie** end-to-end scenario. | Add `Create Pie Chart` scenario in `report_experience_test.dart` reusing the existing Bar fixture and pie-friendly assertions (slice labels, legend). | ~1 hr |
| 6 | No **Scatter** end-to-end scenario. | Add `Create Scatter Chart` scenario in the same file, asserting axis min/max/step/title after creation. | ~1 hr |

### 9.1 Code snippet for gap #1

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

---

## 10. Suggested reading order

- [ ] Read section 2 to learn the automation test-ids exposed by the
      toolbar. These are the primary anchors for any new automation.
- [ ] Skim section 3 — this is the click-flow contract. Any regression in
      the create dropdown will show here first.
- [ ] Skim section 4 — this is the render-config contract. Any Highcharts
      version bump should be validated against these tests.
- [ ] Run the local commands in section 8.1 once to make sure your
      environment can generate line coverage.
- [ ] Review section 9. The six gaps make a good first PR series for
      whoever picks them up.
- [ ] After the six gaps are closed, all three chart types should be at
      100 % scenario coverage across all six layers.

---

## 11. Reference files (quick jump)

**Production**

- [`lib/src/reports/report_rich/report_toolbar.dart`](https://github.com/Workiva/graph_app/blob/master/lib/src/reports/report_rich/report_toolbar.dart)
- [`lib/src/reports/chart/chart_module/models/plot_type.dart`](https://github.com/Workiva/graph_app/blob/master/lib/src/reports/chart/chart_module/models/plot_type.dart)
- [`lib/src/reports/chart/chart_module/loader/`](https://github.com/Workiva/graph_app/tree/master/lib/src/reports/chart/chart_module/loader)
- [`lib/src/reports/chart/chart_module/writer/`](https://github.com/Workiva/graph_app/tree/master/lib/src/reports/chart/chart_module/writer)

**Unit tests**

- [`test/unit/common/experiences/graph_report/report_toolbar_test.dart`](https://github.com/Workiva/graph_app/blob/master/test/unit/common/experiences/graph_report/report_toolbar_test.dart)
- [`test/unit/common/chart/`](https://github.com/Workiva/graph_app/tree/master/test/unit/common/chart)

**Functional tests**

- [`dart_functional/framework/page_objects/reports/report_toolbar.dart`](https://github.com/Workiva/graph_app/blob/master/dart_functional/framework/page_objects/reports/report_toolbar.dart)
- [`dart_functional/framework/page_objects/chart.dart`](https://github.com/Workiva/graph_app/blob/master/dart_functional/framework/page_objects/chart.dart)
- [`dart_functional/test/ci/reports_dashboards/report_experience_test.dart`](https://github.com/Workiva/graph_app/blob/master/dart_functional/test/ci/reports_dashboards/report_experience_test.dart)
