# KQL operators: a quick reference

Source of truth: the [Kusto Query Language documentation](https://learn.microsoft.com/kusto/query/) (its source lives in the public [dataexplorer-docs](https://github.com/MicrosoftDocs/dataexplorer-docs) repository). This page is a short, practical summary of the operators you meet most often. The official reference has every option and edge case.

A KQL query is a data flow: start from a table, then pipe it through operators with `|`. Each operator takes a table and returns a table.

```kusto
StormEvents
| where State == "TEXAS" and StartTime > ago(30d)
| summarize events = count(), damage = sum(DamageProperty) by EventType
| top 5 by damage
```

## Filter and shape rows

| Operator | What it does | Example |
|---|---|---|
| `where` | Keeps rows that match a condition. | `\| where Level == "Error"` |
| `take` (also `limit`) | Returns any N rows. Fast way to peek at a table; the two names are equivalent. | `\| take 10` |
| `top N by col` | Returns the first N rows ordered by a column. | `\| top 5 by Duration desc` |
| `sort by` (also `order by`) | Orders rows. **The default is descending**; add `asc` for ascending. | `\| sort by Timestamp asc` |
| `distinct` | Unique combinations of the listed columns. | `\| distinct Computer, Level` |
| `sample N` | Returns N random rows. | `\| sample 100` |
| `count` | Returns the number of rows. | `\| count` |

## Choose and add columns

| Operator | What it does | Example |
|---|---|---|
| `project` | Keeps (and can compute) only the listed columns. | `\| project Timestamp, Level, Message` |
| `project-away` | Removes the listed columns. | `\| project-away _ResourceId` |
| `project-rename` | Renames columns. | `\| project-rename Time = Timestamp` |
| `project-reorder` | Changes column order. | `\| project-reorder Timestamp, Level` |
| `project-keep` | Keeps columns that match a name pattern. | `\| project-keep Event*` |
| `extend` | Adds computed columns, keeping the existing ones. | `\| extend DurationSec = Duration / 1000` |

## Aggregate

| Operator | What it does | Example |
|---|---|---|
| `summarize … by …` | Groups rows and aggregates, using functions like `count()`, `sum()`, `avg()`, `dcount()`, `max()`, `arg_max()`, `make_set()`. | `\| summarize count() by bin(Timestamp, 1h), Level` |
| `make-series` | Builds a regular time series, filling gaps. | `\| make-series n=count() on Timestamp step 1h` |
| `top-nested`, `top-hitters` | Nested top-N and approximate most-frequent values. | `\| top-hitters 10 of ClientIP by count()` |

`bin(value, size)` rounds a value down to a multiple of `size`, which is how you bucket times: `bin(Timestamp, 15m)`.

## Combine tables

| Operator | What it does | Example |
|---|---|---|
| `join` | Combines rows from two tables on matching columns. **The default kind is `innerunique`**, which removes duplicate rows on the left side first. Set `kind=inner`, `leftouter`, `leftanti` and so on when you want something else. | `A \| join kind=inner (B) on Id` |
| `lookup` | Enriches a large (fact) table from a small (dimension) table. Supports only `leftouter` (the default) and `inner`. | `Events \| lookup Users on UserId` |
| `union` | Stacks the rows of several tables. | `union Logs2025, Logs2026` |
| `evaluate` | Runs a plugin, for example `evaluate bag_unpack(Properties)`. | `\| evaluate bag_unpack(Properties)` |

## Work with arrays and text

| Operator | What it does | Example |
|---|---|---|
| `mv-expand` | Turns each element of an array or property bag into its own row. | `\| mv-expand Tags` |
| `mv-apply` | Applies a sub-query to each array element, then merges the results. | `\| mv-apply t = Tags to typeof(string) on (where t startswith "env")` |
| `parse` | Extracts values from a string using a pattern. | `\| parse Message with "user=" User " action=" Action` |
| `parse-where` | Like `parse`, but drops rows that do not match. | `\| parse-where Message with "id=" Id` |
| `parse-kv` | Extracts key-value pairs from a string. | `\| parse-kv Message as (user:string, ip:string) with (pair_delimiter=" ")` |
| `search` / `find` | Search across columns or tables without naming them. Convenient, but slower than `where` on a known column. | `search "timeout"` |

## String comparisons

Text operators are a common stumbling block. In short:

| Operator | Matches | Case |
|---|---|---|
| `==` | Equal | case-sensitive |
| `=~` | Equal | case-insensitive |
| `has` | A whole indexed term (terms are three or more characters) | case-insensitive |
| `contains` | A substring anywhere | case-insensitive |
| `startswith` / `endswith` | Prefix / suffix | case-insensitive |
| `matches regex` | A regular expression | case-sensitive |
| `in`, `!in` | One of several values | case-sensitive (use `in~` for case-insensitive) |

Prefer `has` over `contains` when you are looking for a whole word: it uses the index and is much faster on large tables. The `_cs` variants (`has_cs`, `contains_cs`) are case-sensitive.

## Create data and name things

| Operator | What it does | Example |
|---|---|---|
| `let` | Names a value, a table or a function for reuse in the query. | `let cutoff = ago(7d);` |
| `datatable` | Defines a small table inline. | `datatable(Id:int, Name:string) [1, "a", 2, "b"]` |
| `print` | Evaluates expressions into a one-row table. | `print now()` |
| `range` | Generates a sequence of values. | `range i from 1 to 10 step 1` |
| `as` | Names the current result so later parts of the query can refer to it. | `\| as T` |
| `getschema` | Lists the columns and types of the input. | `Logs \| getschema` |
| `render` | Tells the client to draw a chart (`timechart`, `barchart`, `piechart`, …). | `\| render timechart` |
| `serialize` | Marks a row order as fixed so row-number style functions work. | `\| serialize \| extend n = row_number()` |
| `externaldata` | Reads a file from storage as a table. | `externaldata(Id:int)[@"https://…/ids.csv"]` |

Also available: `consume`, `facet`, `fork`, `invoke`, `partition`, `reduce`, `scan`, `sample-distinct`, `macro-expand`, and a set of graph operators (`make-graph`, `graph-match`, `graph-shortest-paths`, `graph-to-table`, `graph-mark-components`).

---

Editing KQL in a JetBrains IDE? [Tabular Pipes](https://plugins.jetbrains.com/plugin/36123-tabular-pipes) adds highlighting, completion with short documentation, and a formatter that puts each pipe stage on its own line, all offline.
