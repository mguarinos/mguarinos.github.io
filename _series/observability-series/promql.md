---
title: "PromQL: writing queries that mean something"
tags: [prometheus, promql, metrics, rate, histogram_quantile, recording-rules]
series_part: 4
toc: true
description: "The PromQL mental model: instant vs range vectors, rate() vs irate(), histogram_quantile() for percentiles, aggregation operators, and recording rules that precompute expensive queries."
---

This is part 4 of the [Production Observability series](/series/#observability-series). We have metrics flowing from [part 3](/posts/observability-series/instrumentation/).

---

## Instant vectors and range vectors

Every PromQL query returns one of two types.

An **instant vector** is a set of time series, each with a single sample at the query evaluation time:

```promql
http_requests_total
```

Returns the current value of every time series matching `http_requests_total`. Each series appears once with one value.

A **range vector** is a set of time series, each with a *range* of samples over a time window:

```promql
http_requests_total[5m]
```

Returns the last 5 minutes of samples for each time series. You cannot graph a range vector directly — it is an intermediate type used as input to functions like `rate()`.

---

## Selectors and label matching

Narrow a query by adding label matchers in `{}`:

```promql
# Exact match
http_requests_total{status="200"}

# Not equal
http_requests_total{status!="200"}

# Regex match (anchored)
http_requests_total{status=~"5.."}

# Regex not-match
http_requests_total{status!~"2.."}
```

`=~` uses RE2 syntax and is anchored — `"5.."` matches `500`, `503`, etc. Multiple matchers are ANDed together:

```promql
http_requests_total{job="order-service", status=~"5.."}
```

---

## rate() vs irate()

`rate()` computes the per-second average rate of increase over a range window. It handles counter resets automatically (a restart that resets a counter to zero does not produce a negative spike).

```promql
rate(http_requests_total[5m])
```

`irate()` computes the instantaneous rate using only the last two data points in the window. It is more sensitive to spikes but noisier.

Use `rate()` for dashboards and alerts — it is smooth and reliable. Use `irate()` when you specifically need to catch short-lived spikes that `rate()` would average away.

The window duration in `rate()` affects sensitivity. A `[1m]` window catches rapid changes but is noisy. A `[5m]` window is smooth but slow to react. For dashboards: `[5m]`. For fast-firing alerts: `[2m]`.

A common mistake: using too short a window relative to the scrape interval. If Prometheus scrapes every 15 seconds, a `[30s]` window often has only two data points. Use at least 4× the scrape interval as your rate window.

---

## increase()

`increase()` is `rate()` multiplied by the window duration — it gives you the total number of increases over the window rather than per-second rate. Use it when you want counts ("how many errors in the last hour") rather than rates ("errors per second").

```promql
# Errors in the last 1 hour
increase(http_requests_total{status=~"5.."}[1h])
```

---

## histogram_quantile()

Histograms are stored as bucket counts, not as pre-computed percentiles. To get a percentile, use `histogram_quantile()`:

```promql
histogram_quantile(
  0.99,
  rate(http_request_duration_seconds_bucket[5m])
)
```

This computes the 99th percentile latency over the last 5 minutes. The `rate()` wrapper on the bucket metric is required — you want the rate of observations into each bucket, not the cumulative total. Without `rate()`, the quantile computation uses all historical data since the process started, which is usually not what you want.

The accuracy depends on bucket placement. If the true p99 falls between buckets at 0.25s and 0.5s, Prometheus linearly interpolates within that bucket. Narrow buckets around your SLO threshold give more accurate results.

---

## Aggregation operators

PromQL aggregation collapses a set of time series into a smaller set by applying a function:

```promql
# Sum across all instances (one result)
sum(rate(http_requests_total[5m]))

# Sum, keeping the endpoint label
sum(rate(http_requests_total[5m])) by (endpoint)

# Sum, dropping the instance label (keep everything else)
sum(rate(http_requests_total[5m])) without (instance)
```

`by` keeps only the listed labels. `without` keeps everything except the listed labels. Use `by` when you know which dimensions you want; use `without` when you want to drop a single dimension like `instance` and preserve the rest.

**The histogram aggregation pattern** — to compute a service-wide percentile across multiple instances:

```promql
histogram_quantile(
  0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint)
)
```

The `sum(...) by (le, endpoint)` step first aggregates the bucket counts across all instances. Only after that does `histogram_quantile` compute the percentile. This is correct. If you compute `histogram_quantile` per instance first and then average the results, you get a meaningless number — percentiles cannot be averaged.

---

## Practical queries for the order service

Error rate as a fraction of total traffic:

```promql
sum(rate(http_requests_total{job="order-service", status=~"5.."}[5m]))
/
sum(rate(http_requests_total{job="order-service"}[5m]))
```

p50, p95, p99 latency for `POST /orders`:

```promql
histogram_quantile(0.50,
  sum(rate(http_request_duration_seconds_bucket{
    job="order-service", endpoint="/orders", method="POST"
  }[5m])) by (le)
)
```

Replace `0.50` with `0.95` or `0.99` for the other percentiles. Grafana makes it easy to parameterize this with a variable.

Orders created per minute:

```promql
sum(rate(orders_total{status="created"}[5m])) * 60
```

Failed order fraction:

```promql
sum(rate(orders_total{status="failed"}[5m]))
/
sum(rate(orders_total[5m]))
```

Current in-flight orders:

```promql
orders_in_flight
```

## Recording rules

Recording rules precompute expensive or frequently-used queries and store the results as new time series. They serve two purposes:

1. **Performance** — Grafana and alert evaluations re-run the same queries on every refresh. Precomputing `rate(http_requests_total[5m])` once per evaluation interval is cheaper than computing it fresh for every dashboard panel and every alert rule.

2. **Composability** — recording rule results are regular time series. You can use them in other rules, other dashboards, and for long-range queries against historical data.

`prometheus/rules/recording.yml`:

```yaml
groups:
  - name: order_service
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job, endpoint)

      - record: job:http_errors:rate5m
        expr: >
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job, endpoint)

      - record: job:http_request_duration_p99:5m
        expr: >
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job, endpoint)
          )

      - record: job:http_request_duration_p95:5m
        expr: >
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job, endpoint)
          )
```

The naming convention `level:metric:operation` (e.g., `job:http_requests:rate5m`) is the Prometheus community standard. The level (`job`) identifies the aggregation dimension, the metric (`http_requests`) is what is being measured, and the operation (`rate5m`) describes the computation.

Apply recording rules without restarting Prometheus:

```bash
curl -X POST http://localhost:9090/-/reload
```

Then verify in Status → Rules that the rules loaded without errors, and in the expression browser that you can query `job:http_requests:rate5m` directly.

---

## What you have now

- A mental model for instant vs range vectors and why the distinction matters
- `rate()` for dashboard and alert queries, `irate()` for instantaneous spikes
- `histogram_quantile()` with the correct `sum by (le)` aggregation pattern
- Aggregation operators that collapse across instances correctly
- Recording rules precomputing the expensive queries that alert rules and dashboards share

In the next post we take these queries into Grafana and build dashboards.