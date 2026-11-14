# PromQL functions: a quick reference

Source of truth: the [Prometheus querying functions documentation](https://prometheus.io/docs/prometheus/latest/querying/functions/). This page is a short, practical summary written for quick lookup. Check the official page for edge cases and for functions marked experimental.

A few terms: an **instant vector** is one value per series at one moment; a **range vector** is a window of samples per series, written with a duration such as `[5m]`; a **scalar** is a single number.

## Rates and counters

Use these on counters (values that only go up, apart from resets).

| Function | Takes → returns | What it does | Example |
|---|---|---|---|
| `rate(v range-vector)` | range → instant | Per-second average increase over the window, adjusted for counter resets. The usual choice for graphs and alerts. | `rate(http_requests_total[5m])` |
| `irate(v range-vector)` | range → instant | Per-second rate from the last two samples only. Reacts fast, but is noisy. | `irate(http_requests_total[1m])` |
| `increase(v range-vector)` | range → instant | Total increase over the window (like `rate` times the window length). | `increase(errors_total[1h])` |
| `resets(v range-vector)` | range → instant | How many times a counter reset in the window. | `resets(process_start_total[1d])` |

## Gauges: change over time

| Function | What it does | Example |
|---|---|---|
| `delta(v range-vector)` | Difference between first and last value in the window. For gauges. | `delta(node_memory_free_bytes[1h])` |
| `idelta(v range-vector)` | Difference between the last two samples. | `idelta(queue_depth[5m])` |
| `deriv(v range-vector)` | Per-second slope of a gauge, by linear regression. | `deriv(disk_used_bytes[30m])` |
| `predict_linear(v range-vector, t scalar)` | Extrapolates the value `t` seconds ahead using linear regression. | `predict_linear(disk_free_bytes[6h], 4 * 3600) < 0` |
| `changes(v range-vector)` | Number of times the value changed in the window. | `changes(config_hash[1d])` |
| `double_exponential_smoothing(v range-vector, sf scalar, tf scalar)` | Smoothed value for slowly moving gauges. `sf` is the smoothing factor and `tf` the trend factor, both between 0 and 1. **Experimental:** it only works when Prometheus runs with `--enable-feature=promql-experimental-functions`. | `double_exponential_smoothing(temp[10m], 0.5, 0.1)` |

## Aggregating over time

The `<aggregation>_over_time` family collapses each series' window into one value: `avg_over_time`, `min_over_time`, `max_over_time`, `sum_over_time`, `count_over_time`, `quantile_over_time(φ, v)`, `stddev_over_time`, `stdvar_over_time`, `last_over_time`, `present_over_time` and `absent_over_time`.

```
max_over_time(cpu_usage[1h])
quantile_over_time(0.95, request_latency_seconds[1d])
```

## Math and rounding

| Function | What it does | Example |
|---|---|---|
| `abs(v)` | Absolute value. | `abs(delta(temp[5m]))` |
| `ceil(v)`, `floor(v)` | Round up or down to a whole number. | `ceil(queue_depth / 10)` |
| `round(v, to_nearest=1)` | Round to the nearest multiple of `to_nearest`. | `round(cpu_usage, 0.1)` |
| `sqrt(v)`, `exp(v)` | Square root; e to the power v. | `sqrt(variance)` |
| `ln(v)`, `log2(v)`, `log10(v)` | Logarithms. | `log10(bytes_total)` |
| `clamp(v, min, max)` | Keep values within `[min, max]`. | `clamp(ratio, 0, 1)` |
| `clamp_min(v, min)`, `clamp_max(v, max)` | Clamp on one side only. | `clamp_min(free_bytes, 0)` |
| `sgn(v)` | −1, 0 or 1 by sign. | `sgn(delta(temp[10m]))` |

The trigonometric functions (`sin`, `cos`, `tan`, `asin`, `acos`, `atan`, `sinh`, `cosh`, `tanh`, plus `deg` and `rad`) are also available.

## Missing data and conversions

| Function | What it does | Example |
|---|---|---|
| `absent(v instant-vector)` | Returns a single series with value 1 if the input has no samples; otherwise empty. Handy for "metric disappeared" alerts. | `absent(up{job="api"})` |
| `absent_over_time(v range-vector)` | Same, but over a window. | `absent_over_time(up{job="api"}[10m])` |
| `scalar(v)` | Turns a single-element vector into a scalar (NaN if there are not exactly one). | `scalar(sum(up))` |
| `vector(s scalar)` | Turns a scalar into a one-element vector. | `vector(1)` |
| `time()` | Current evaluation time, in seconds since the Unix epoch. | `time() - process_start_time_seconds` |
| `timestamp(v)` | Timestamp of each sample, in epoch seconds. | `time() - timestamp(last_backup)` |

## Sorting

`sort(v)` and `sort_desc(v)` order by value. `sort_by_label(v, label…)` and `sort_by_label_desc(v, label…)` order by label values. Sorting only has an effect on instant queries.

## Labels

| Function | What it does | Example |
|---|---|---|
| `label_replace(v, dst, replacement, src, regex)` | Writes a new label `dst` using a regex over label `src`. | `label_replace(up, "host", "$1", "instance", "(.*):.*")` |
| `label_join(v, dst, separator, src…)` | Joins several label values into `dst`. | `label_join(up, "id", "-", "job", "instance")` |

## Date and time parts

`year()`, `month()`, `day_of_month()`, `day_of_week()`, `day_of_year()`, `days_in_month()`, `hour()` and `minute()` return that part of a timestamp, in UTC. With no argument they use the current time.

```
hour() >= 9 and hour() < 17 and day_of_week() > 0 and day_of_week() < 6
```

## Histograms

| Function | What it does | Example |
|---|---|---|
| `histogram_quantile(φ, b)` | Estimates the φ-quantile (0 to 1) from histogram buckets (`le` label). | `histogram_quantile(0.99, sum by (le) (rate(request_duration_bucket[5m])))` |
| `histogram_count(v)`, `histogram_sum(v)` | Count and sum of observations in a native histogram. | `histogram_sum(rate(latency[5m]))` |
| `histogram_avg(v)` | Average observation of a native histogram. | `histogram_avg(rate(latency[5m]))` |
| `histogram_fraction(lower, upper, v)` | Fraction of observations between two values. | `histogram_fraction(0, 0.2, rate(latency[5m]))` |

Some newer histogram functions are marked experimental in the official docs; check there before relying on them.

## Aggregation operators (not functions)

`sum`, `min`, `max`, `avg`, `count`, `group`, `stddev`, `stdvar`, `topk`, `bottomk`, `quantile` and `count_values` combine series. Narrow the result with `by (labels)` or `without (labels)`:

```
sum by (job) (rate(http_requests_total[5m]))
topk(3, sum by (instance) (rate(cpu_seconds_total[5m])))
```

---

Editing PromQL in a JetBrains IDE? [Ruleglass](https://plugins.jetbrains.com/plugin/36121-ruleglass) adds highlighting, completion and function documentation, including inside Prometheus rule files.
