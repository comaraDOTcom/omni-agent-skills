# Building Dashboards People Can Act On

The [Design defaults](../SKILL.md#design-defaults) make a dashboard readable. This reference makes an operational or exec dashboard *actionable*: a viewer should finish it knowing what to do today and with whom. Same rule as the defaults: direction from the user, an existing dashboard's conventions, or an org theme wins.

Field-tested on a customer build (CLI 1.4.0). Where a result is "observed", it was seen on a live instance but is not documented behavior; re-check it with a PNG render.

## Contents

- [Order by what can still change](#order-by-what-can-still-change)
- [Tables: five columns, no truncated headers](#tables-five-columns-no-truncated-headers)
- [List rows in a markdown tile](#list-rows-in-a-markdown-tile)
- [Every view drills](#every-view-drills)
- [Relative dates everywhere](#relative-dates-everywhere)
- [Click path: overview → breakdown → deep dive](#click-path-overview--breakdown--deep-dive)
- [Colour has one meaning](#colour-has-one-meaning)
- [Comparing a value with its outcome](#comparing-a-value-with-its-outcome)
- [Shares that sum to 100%](#shares-that-sum-to-100)
- [Legends and axis formats](#legends-and-axis-formats)
- [Titles, subtitles, and KPI height](#titles-subtitles-and-kpi-height)
- [Scope filters vs viewer controls](#scope-filters-vs-viewer-controls)
- [Checklist before publishing](#checklist-before-publishing)

## Order by what can still change

- **Still recoverable before already lost.** Put the "at risk" list above or left of the "over / lost" list. A viewer can act on the first; the second is a post-mortem.
- **Every list row names an owner** (account manager, team lead, project owner). A row without an owner answers "what", not "with whom".
- **Use a live window, not lifetime totals.** For example, open items with activity in the last 90 days. Lifetime numbers are dominated by stale records. Say the window in the subtitle.
- **Move data-quality outliers to their own tile.** A tiny denominator (a 2-hour budget, a £10 target) produces ratios like 77,214% that top every ranked list. Filter them out of the main lists (`budget > 10`) and show them in a "Check these" tile.

## Tables: five columns, no truncated headers

- **At most 5 columns.** If a row needs more context, use a [list row](#list-rows-in-a-markdown-tile).
- **Long measure labels truncate column headers** ("Time Value at Cli…") on a narrow tile. Shorten the labels (`resultConfig` column labels or a shorter measure `label`), widen the tile, or switch to a list row. Never ship a table whose headers truncate.
- **Size the row `limit` to what the tile shows** without scrolling, and say so in the subtitle ("the 7 closest to budget"). Check it in a PNG render ([downloads.md](downloads.md)); the API cannot tell you a header truncated.

## List rows in a markdown tile

A `{{#result}}…{{/result}}` section repeats once per result row. Inside it, field tokens are relative to the row: `{{view.field.value}}`, not `{{result._first.view.field.value}}`. One row = a bold title, one muted context line, one number, and an optional bar against a target:

```html
<style>
.r{display:flex;gap:12px;align-items:center;padding:6px 0;border-bottom:1px solid #E5E7EB}
.r .t{flex:1;min-width:0} .r .c{font-size:12px;color:#6B7280}
.r .n{font-weight:700;white-space:nowrap}
.bar{position:relative;height:6px;background:#F1F5F9;border-radius:3px;margin-top:4px}
.bar .f{height:6px;background:#D97706;border-radius:3px}
.bar .m{position:absolute;left:66.67%;top:-3px;width:2px;height:12px;background:#334155}
</style>
{{#result}}
<div class="r">
  <div class="t"><b>{{projects.name.value}}</b>
    <div class="c">{{projects.owner.value}} · {{projects.client.value_static}}</div>
    <div class="bar"><div class="f" style="width:min(100%, calc({{projects.pct_of_budget.raw}} / 1.5 * 100%))"></div><div class="m"></div></div>
  </div>
  <div class="n">{{projects.pct_of_budget.value}}</div>
</div>
{{/result}}
```

The bar's scale is 0–150%, so the 100% target marker sits at 66.67%. Clamp with `min(100%, …)` so overruns don't spill. The tile needs `automaticVis: false` ([visConfig.md](visConfig.md)); sort and `limit` the query, since the section renders every row returned.

## Every view drills

In markdown tiles, use **`.value`** for names and key numbers: it renders a drillable, interactive element. **`.value_static`** is display-only text; use it for secondary context. **`.raw`** belongs only inside CSS or HTML attributes (`width:calc(…)`, `class="…"`). A dashboard whose markdown uses only `.value_static` has no drill path. Token reference: [mustache.md](mustache.md).

## Relative dates everywhere

Use a relative window on the dashboard date control **and** on every tile-level date filter, so the dashboard never goes stale:

```jsonc
{ "type": "date", "kind": "TIME_FOR_INTERVAL_DURATION", "ui_type": "PAST",
  "left_side": "12 complete months ago", "right_side": "12 months" }
```

- **Month over month:** `"left_side": "2 complete months ago", "right_side": "2 months"`, labelled "last month vs previous month".
- **Never hard-code a month name** in a tile title, subtitle, series title, or calculation label ("September vs August"). It is wrong the day the month turns. Use relative words, or `{{filters.<view.field>.summary}}` in a markdown tile ([mustache.md](mustache.md)).

## Click path: overview → breakdown → deep dive

Three pages, each answering the next question:

1. **Overview** — the KPI that says whether there is a problem.
2. **Breakdown** — where it is: a `heatmap` of department × business unit ([visConfig.md](visConfig.md#config-object-heatmap)), clickable to drill.
3. **Deep dive** — "pick a team": a single-select string filter placed **in the page**, not the filter bar, and a "View by" switcher **inside the chart tile** to swap the breakdown dimension.

```jsonc
// controls.data
"team": { "config": {
  "type": "string", "kind": "EQUALS", "fieldName": "time.team", "label": "Team",
  "values": ["Design"], "filterControlType": "singleValueEquals" }, "map": {} },
"view_by": { "config": {
  "id": "view_by", "type": "FIELD_SELECTION", "kind": "FIELD", "display": "BUTTON_TOGGLE",
  "label": "View by", "field": "time.person",
  "options": [ { "label": "Person", "value": "time.person" },
               { "label": "Client", "value": "time.client" },
               { "label": "Task",   "value": "time.task_type" } ] },
  "map": {} }

// page-3 grid: the filter as its own grid item, above the chart
{ "type": "filter", "id": "team", "instanceKey": "p3-team",
  "appearance": { "control": "buttonToggle", "display": "inline" },
  "gridPosition": { "x": 0, "y": 7, "w": 24, "h": 6 } }

// the chart tile stack's children: chart, then the switcher
{ "as": "chart", "id": "9", "type": "query", "instanceKey": "tile-viz-9" },
{ "type": "control", "id": "view_by", "instanceKey": "tile-9-view-by" }
```

A page-placed filter still applies to every tile it maps to, on every page. Scope it with `map` ([controls.md](controls.md#map--per-tile-scoping)). Page and placement mechanics: [containers.md](containers.md).

## Colour has one meaning

- **One meaning per colour across the page:** red = over / lost, amber = at risk, neutral slate or grey = context. Don't reuse red for a series that only means "this year".
- **Always pair colour with a text label** ("At risk", "Over") so the meaning survives greyscale and colour-blind viewing.
- **Set single-series colours explicitly** with `series[].mark._mark_color` + `manual: true` ([visConfig.md](visConfig.md#series-array)). An `automaticVis` chart draws in the default blue, which clashes with a red/amber page.

## Comparing a value with its outcome

To show a value against what it produced (time value vs recognised revenue, budget vs spend), don't overlay or group two bars and make the reader estimate the gap. Either:

- **Stack the parts:** "recovered" + "not recovered", where "not recovered" is a difference measure (`value - outcome`). The bar total is the value and the red segment is the gap. Use `barStacked` / `columnStacked` and title the series with `series[].title.value`.
- **Plot the ratio** (`outcome / value`) as a line on its own chart.

One value axis only; never put the two on `y` and `y2`.

## Shares that sum to 100%

Observed: `barStackedPercentage` did **not** normalise raw-hour measures; the axis showed 1,600,000%. Model the shares instead:

1. Define one share measure per segment series (`billable_hours / total_hours`, `non_billable_hours / total_hours`, …), so the series sum to 1 on every bar.
2. Chart the share measures with `barStacked` and pin the value axis:

```jsonc
"x": { "axis": { "domain": { "min": 0, "max": 1 },
                 "label": { "format": { "format": "percent_0" } } } }
```

Check the axis reads 0–100% in a PNG render.

## Legends and axis formats

- **Legend at the bottom:** `config.color.legendPosition: "bottom"` (lowercase). Observed: `"BOTTOM"` rendered the legend over the bars in a PNG export.
- **Rename legend entries** with `series[].title.value`; there is no `series[].label` ([visConfig.md](visConfig.md#series-array)).
- **Axis number format:** `<axis>.axis.label.format.format`, e.g. `"number_0"`, `"gbpcurrency_0"`, `"percent_0"`.

## Titles, subtitles, and KPI height

- **Titles are questions** ("Which teams are absorbing it?", "Is it getting better?").
- **The subtitle is the definition and the window, in one line** ("Hours over budget ÷ budget, open jobs, last 90 days").
- **A KPI tile with a two-line subtitle needs `h` ≥ 22–24.** At `h: 16` the number clipped.

## Scope filters vs viewer controls

- **Fixed scope** (one business unit, open status only) goes in each tile's `query.filters`, not the filter bar. Nobody changes it, so it shouldn't take a control.
- **At most 5 viewer-facing controls**, each one a viewer would change.
- **A tile-level scope filter on a field that also has a dashboard control is cancelled** when the control is blank, because two filters on one field OR together ([controls.md](controls.md#map--per-tile-scoping)). Scope through a different field that has no dashboard control.

## Checklist before publishing

- [ ] At-risk before lost; an owner on every row
- [ ] No table over 5 columns; no truncated headers in the PNG
- [ ] Every row limit fits its tile and is named in the subtitle
- [ ] Names and key numbers in markdown use `.value`
- [ ] Every date filter is relative; no month names in titles or labels
- [ ] Each colour means one thing and has a text label
- [ ] Outliers sit in a check tile, not the main list
- [ ] Fixed scope in tile filters; 5 or fewer viewer controls
