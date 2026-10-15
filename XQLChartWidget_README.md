# XQLChartWidget

Runs an XQL query over a time frame (default 90 days) and shows the result as a line, pie, column, or number widget on a Cortex XSIAM dashboard.

## Core requirements (read this first)

A search must have these parts, or the widget will not work. The script checks for them and tells you what is missing.

| Chart | The query must have | Set these arguments |
|---|---|---|
| **All charts** | `dataset = <name>` at the start, and a `comp` stage that names its result: `comp count() as count` | `value_field` must match the `as` name (default `count`) |
| **Line / column** | `bin _time span = 1d` **and** `comp ... by _time` | Nothing else (`x_field` defaults to `_time`) |
| **Line / column, several lines** | `bin _time span = 1d` **and** `comp ... by _time, <field>` | `series_field=<field>` |
| **Pie** | `comp count() as count by <field>` (no `bin` needed) | `category_field=<field>` |
| **Number** | `comp count() as count` (no `by` needed) | Nothing else |

- Field names are case-sensitive and must match the dataset exactly. Run the widget with `chart_type=raw` to see the real field names.
- Do not set the time range inside the query. Use the `time_frame` argument.
- Long field names such as `xdm.case.severity` work, but the chart legend reads better if you rename them first: `| alter severity = xdm.case.severity`.

Copy-paste examples (all tested on the `cases` dataset):

```
Line:    dataset = cases | bin _time span = 1d | comp count() as count by _time
Pie:     dataset = cases | alter severity = xdm.case.severity | comp count() as count by severity
Number:  dataset = cases | comp count() as count
```

## Before you start

- The **XQL Query Engine** integration must have an instance configured.
- The script must have the `widget` tag. The YAML already includes it.

## Set up a widget

1. Go to **Investigation & Response > Automation > Scripts** and upload `XQLChartWidget.yml`.
2. Go to **Dashboards & Reports > Widget Library > Create custom widget > Script** and select `XQLChartWidget`.
3. Set the arguments, e.g. `chart_type`, `query` and `time_frame`.
4. In the Chart Editor, pick the same chart type as `chart_type` (**Line**, **Pie**, **Column**, or **Single Value** for `number`), then save.

Create one widget for each chart. You can use the same script for all of them.

## Arguments

| Argument | Default | What it does |
|---|---|---|
| `chart_type` | `line` | The chart to build: `line`, `pie`, `column`, `number`, or `raw` (a table of results, for checking a query). |
| `query` | daily count of cases | The XQL query. See Core requirements. |
| `time_frame` | `90 days` | The time range, counted back from now, e.g. `10 days`, `30 days`, `90 days`, `12 hours`, `2 weeks`, or an exact range: `between 2026-01-01T00:00:00Z and 2026-04-01T00:00:00Z`. |
| `x_field` | `_time` | The X axis field for line and column charts. |
| `value_field` | `count` | The number to plot. Must match the `as` name in `comp`. |
| `series_field` | empty | Splits a line or column chart into one line per value, e.g. `severity`. |
| `category_field` | empty | The field for pie slices, e.g. `status`. If empty, the pie shows totals per month. |
| `date_format` | automatic | The X axis label format, e.g. `%d %b`. If empty, it uses `%Y-%m-%d` for days and weeks, or `%Y-%m-%d %H:00` for hours. |
| `fill_missing_days` | `true` | Adds zeros for days (or hours, or weeks) with no results, so the line covers the whole time frame. |
| `validate_query` | `true` | Checks the query for missing or misspelled parts before it runs. Set to `false` for advanced queries the check doesn't understand. |
| `output_style` | `docs` | The entry format. Use `widget` only if the widget preview is blank. |
| `polling_interval` | `10` | The number of seconds between checks for results. |
| `query_timeout_duration` | `120` | The maximum number of seconds to wait for results. Use 180-240 for more than 30 days. |
| `xql_query_instance` | empty | The XQL Query Engine instance to use, if it isn't the default one. |

## Search recipes

These are all tested against the dev tenant (see `tests/recipes.py`).

| Goal | `query` | Other arguments |
|---|---|---|
| Cases per day | `dataset = cases \| bin _time span = 1d \| comp count() as count by _time` | |
| Cases per week (long ranges) | `dataset = cases \| bin _time span = 1w \| comp count() as count by _time` | `time_frame=180 days` |
| One line per severity | `dataset = cases \| alter severity = xdm.case.severity \| bin _time span = 1d \| comp count() as count by _time, severity` | `series_field=severity` |
| Stacked columns by severity | Same query as the line above | `chart_type=column series_field=severity` (Chart Editor: Column > Stacked) |
| Pie by severity | `dataset = cases \| alter severity = xdm.case.severity \| comp count() as count by severity` | `chart_type=pie category_field=severity` |
| Pie by status | `dataset = cases \| alter status = xdm.case.status_progress \| comp count() as count by status` | `chart_type=pie category_field=status` |
| Total number | `dataset = cases \| comp count() as count` | `chart_type=number` |

Tips for a good search:

- **Always aggregate with `comp`.** The widget needs a few summary rows, not thousands of raw events.
- **Filter early.** Put `filter` stages right after `dataset`, e.g. `dataset = cases | filter xdm.case.severity = "HIGH" | ...`. Values are case-sensitive. In `cases`, severity is `CRITICAL`, `HIGH`, `MEDIUM` or `LOW`, and `status_progress` is `NEW` or `RESOLVED`. Run `chart_type=raw` to see the values in other datasets.
- **Keep the result small.** The widget reads up to 1,000 rows. Rows = time buckets x series, so 90 days x 5 severities = 450 rows is fine.

## Time frames

| `time_frame` | Use this bin | Points on the line |
|---|---|---|
| `10 days` | `bin _time span = 1d` (or `1h` for detail) | 11 (or about 240 with `1h`) |
| `30 days` | `bin _time span = 1d` | 31 |
| `90 days` | `bin _time span = 1d`, or `1w` for a smoother line | 91 (or 13-14 with `1w`) |
| `180 days` or more | `bin _time span = 1w` | About 26 per 180 days |
| An exact range | Any of the above | `between 2026-01-01T00:00:00Z and 2026-04-01T00:00:00Z` |

For queries over 30 days:

- They take longer. Set `query_timeout_duration` to 180-240.
- They use more XQL compute units. Weekly bins (`1w`) keep the result small.
- Small typos are fixed for you: `30day`, `90 dyas` and `7d` all work. Words like `ninety days` do not.

## Troubleshooting

| Message or symptom | Fix |
|---|---|
| `Did you mean '...'?` | A value or field is misspelled. Use the suggested spelling. |
| `... is not in the 'by' list of the comp stage` | Add the field to `comp ... by`, or fix its spelling. |
| `The comp stage names its result 'cnt' but value_field is 'count'` | Change the query to `as count`, or set `value_field=cnt`. |
| `Line and column charts need a 'bin _time span = 1d' stage` | Add `| bin _time span = 1d` before `comp`. |
| `The query returned 0 rows` | Nothing matched. Try a longer `time_frame`, or run `chart_type=raw`. |
| `Unable to get query results within 120 seconds` | Raise `query_timeout_duration`, or use a shorter `time_frame`. |
| `Unsupported command` or no instance found | Configure the XQL Query Engine integration, or set `xql_query_instance`. |
| The widget preview is blank | Check the Chart Editor type matches `chart_type`, then try `output_style=widget`. |
