# KQL recipes: common query patterns

Source of truth: the [Kusto Query Language documentation](https://learn.microsoft.com/kusto/query/). This page collects patterns that come up often. Table and column names are examples (`Events`, `Requests`, `Users`); substitute your own. For the operator list see [KQL operators](kql-operators).

## Time filters and buckets

Rows from the last 24 hours:

```kusto
Events
| where Timestamp > ago(24h)
```

Count per hour, as a chart:

```kusto
Events
| where Timestamp > ago(1d)
| summarize count() by bin(Timestamp, 1h)
| render timechart
```

A fixed window instead of a relative one:

```kusto
Events
| where Timestamp between (datetime(2026-01-01) .. datetime(2026-02-01))
```

## Grouping and top lists

Top 10 error messages:

```kusto
Events
| where Level == "Error"
| summarize Count = count() by Message
| top 10 by Count
```

Several aggregates at once, with distinct users:

```kusto
Requests
| summarize Requests = count(), Users = dcount(UserId), P95 = percentile(DurationMs, 95) by Route
| order by Requests desc
```

The latest row per key with `arg_max`:

```kusto
Events
| summarize arg_max(Timestamp, *) by DeviceId
```

## Joining tables

`join` defaults to `kind=innerunique`, which removes duplicate left-side rows on the join key first. State the kind when you want ordinary inner-join behaviour or to keep unmatched rows:

```kusto
Requests
| join kind=leftouter (Users | project UserId, Plan) on UserId
```

Rows with no match on the other side:

```kusto
Requests
| join kind=leftanti (Users) on UserId
```

For a small lookup table, `lookup` is simpler:

```kusto
Requests
| lookup Users on UserId
```

## Working with text

Extract fields from a message:

```kusto
Events
| parse Message with "user=" User " action=" Action
```

Prefer `has` to `contains` when you are matching a whole word, because it uses the term index and is faster on large tables:

```kusto
Events
| where Message has "timeout"
```

Regular-expression extraction:

```kusto
Events
| extend Code = extract(@"code=(\d+)", 1, Message)
```

## Arrays and dynamic values

Turn each element of an array into its own row:

```kusto
Events
| mv-expand Tag = Tags
| summarize count() by tostring(Tag)
```

Read a property from a dynamic column:

```kusto
Events
| extend Region = tostring(Properties.region)
```

## Reusable pieces

Name a value or a sub-query with `let`:

```kusto
let cutoff = ago(7d);
let Errors = Events | where Level == "Error" and Timestamp > cutoff;
Errors
| summarize count() by bin(Timestamp, 1d)
```

Small inline test data with `datatable`:

```kusto
datatable(Id:int, Name:string) [1, "a", 2, "b"]
```

## Habits that avoid slow or wrong queries

- Filter early, and filter on time first. Put `where` before `join` and `summarize`.
- Use `project` to keep only the columns you need before heavy operators.
- `==` and `has_cs` are case-sensitive; `=~`, `has` and `contains` are not.
- `take` returns any rows, not the first rows. Use `top` or `sort by` when order matters.
- `sort by` defaults to descending.

---

Editing KQL in a JetBrains IDE? [Tabular Pipes](https://plugins.jetbrains.com/plugin/36123-tabular-pipes) adds highlighting, completion with short documentation, and a formatter that puts each pipe stage on its own line, all offline.
