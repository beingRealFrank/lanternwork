# PromQL recipes: common queries and alert patterns

Source of truth: the [Prometheus querying documentation](https://prometheus.io/docs/prometheus/latest/querying/basics/) and the [alerting rules guide](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/). This page collects patterns that come up again and again. Metric names are examples; substitute your own. For the function list see [PromQL functions](promql-functions).

## Request rate, errors and latency

Requests per second, per job, over the last 5 minutes:

```
sum by (job) (rate(http_requests_total[5m]))
```

Error ratio (share of requests that failed):

```
sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
/
sum by (job) (rate(http_requests_total[5m]))
```

95th percentile latency from a classic histogram. Keep the `le` label in the `by` clause, or `histogram_quantile` has nothing to work with:

```
histogram_quantile(0.95, sum by (le, job) (rate(http_request_duration_seconds_bucket[5m])))
```

Average latency from a histogram or summary's sum and count:

```
rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m])
```

## Resources

Fraction of memory in use (node exporter metric names):

```
1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
```

CPU busy fraction per instance:

```
1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
```

Disk predicted to fill within 4 hours, based on the last 6 hours:

```
predict_linear(node_filesystem_avail_bytes{fstype!="tmpfs"}[6h], 4 * 3600) < 0
```

## Top and bottom lists

Five busiest instances by request rate:

```
topk(5, sum by (instance) (rate(http_requests_total[5m])))
```

## Is it up?

Targets that failed their last scrape:

```
up == 0
```

A job with no `up` series at all (for example, it was removed from the config):

```
absent(up{job="api"})
```

## Comparing with the past

Request rate now compared with the same time a week ago, using the `offset` modifier:

```
sum(rate(http_requests_total[5m]))
/
sum(rate(http_requests_total[5m] offset 1w))
```

## Matching two sides of a binary operation

When both sides carry different label sets, tell Prometheus how to pair them. `on (labels)` matches only on the listed labels; `ignoring (labels)` matches on everything except them. When one side has several series per match, add `group_left` or `group_right`:

```
rate(http_requests_total[5m])
* on (instance) group_left (version)
app_build_info
```

## Alerting rules

A rule file groups rules. `for` makes the condition hold for a period before the alert fires, which avoids alerts on brief spikes:

```yaml
groups:
  - name: example
    rules:
      - alert: HighErrorRatio
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
            / sum by (job) (rate(http_requests_total[5m])) > 0.05
        for: 10m
        labels:
          severity: page
        annotations:
          summary: "More than 5% errors for {{ $labels.job }}"
```

Recording rules precompute expensive expressions. A common naming convention is `level:metric:operations`:

```yaml
groups:
  - name: recordings
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
```

## Habits that avoid wrong answers

- Apply `rate` or `increase` to counters, not gauges, and take the `rate` first, then `sum`. Summing counters first hides resets.
- Make the range window at least four times the scrape interval so `rate` always has enough samples.
- Quote regex matchers with `=~` and `!~`; they are fully anchored, so `status=~"5.."` matches exactly three characters.
- `histogram_quantile` results are estimates; their accuracy depends on your bucket boundaries.

---

Editing PromQL or Prometheus rule files in a JetBrains IDE? [Ruleglass](https://plugins.jetbrains.com/plugin/36121-ruleglass) adds highlighting, completion and function documentation, including inside rule files.
