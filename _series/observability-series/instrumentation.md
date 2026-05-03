---
title: "Instrumenting a Python service: the RED method in practice"
tags: [prometheus, python, fastapi, instrumentation, metrics, prometheus-client]
series_part: 3
toc: true
description: "Add Prometheus metrics to the order service using prometheus_client. Capture rate, error rate, and duration (RED) with ASGI middleware, expose business metrics, and scrape the new target from Prometheus."
---

This is part 3 of the [Production Observability series](/series/#observability-series). In [part 2](/posts/observability-series/prometheus/) we set up Prometheus with infrastructure exporters. Now we instrument the application itself — the part no exporter can do for you.

---

## The prometheus_client library

Python's `prometheus_client` library maintains a global metrics registry, serializes it to the Prometheus text exposition format, and handles thread safety internally.

```bash
pip install prometheus-client
```

The four Python classes map to Prometheus types directly:

```python
from prometheus_client import Counter, Gauge, Histogram, Summary
```

One rule that matters: **define metrics at module level, not inside request handlers**. Metrics are instantiated once and registered globally on import. If you define a `Counter` inside a function body, you create a new metric object on every call — which either errors (duplicate registration) or leaks memory. Define them at the top of a module and import that module everywhere that needs to observe values.

---

## Defining metrics

`app/metrics.py`:

```python
from prometheus_client import Counter, Gauge, Histogram

REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status"],
)

REQUEST_LATENCY = Histogram(
    "http_request_duration_seconds",
    "HTTP request latency in seconds",
    ["method", "endpoint"],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
)

ORDERS_TOTAL = Counter(
    "orders_total",
    "Total orders by outcome",
    ["status"],
)

ORDERS_IN_FLIGHT = Gauge(
    "orders_in_flight",
    "Orders currently being processed",
)
```

The bucket list in `REQUEST_LATENCY` is chosen around expected latency targets for the order service. The baseline order processing is 50–100ms, with slow orders at 300ms+. Placing bucket boundaries at 0.1, 0.25, 0.5 gives good resolution around those values. If your p99 SLO were 50ms, you would want more buckets below 0.1.

---

## Middleware: capturing RED automatically

Rather than adding instrumentation to each endpoint function, we use ASGI middleware to observe every request in one place.

`app/main.py`:

```python
import time
from fastapi import FastAPI, Request
from fastapi.responses import Response
from prometheus_client import REGISTRY
from prometheus_client.openmetrics.exposition import generate_latest
from metrics import REQUEST_COUNT, REQUEST_LATENCY

app = FastAPI()

@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start

    endpoint = request.url.path
    # Normalize dynamic path segments to avoid high cardinality
    if endpoint.startswith("/orders/") and len(endpoint) > len("/orders/"):
        endpoint = "/orders/{id}"

    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=endpoint,
        status=str(response.status_code),
    ).inc()
    REQUEST_LATENCY.labels(
        method=request.method,
        endpoint=endpoint,
    ).observe(duration)

    return response
```

The path normalization at line 17 is not optional. Without it, every unique order ID becomes its own label value — `/orders/abc-123`, `/orders/def-456`, and so on — and you have the cardinality explosion described in post 2. Normalize dynamic segments to their route pattern before using them as label values.

Note that `REQUEST_LATENCY` does not include `status` as a label. This is intentional: for latency we care about the distribution *per endpoint*, not split by status code. Splitting by status would also fragment the histogram's observation counts across labels, making quantile estimates less accurate.

---

## The /metrics endpoint

```python
@app.get("/metrics")
def metrics():
    return Response(generate_latest(REGISTRY), media_type="application/openmetrics-text; version=1.0.0; charset=utf-8")
```

The OpenMetrics content type is required for Prometheus to receive and store exemplars (added in part 10). The metrics content itself is identical to the standard Prometheus text format, with a `# EOF` line appended at the end.

Verify it works (using `order-service` on port 8001):

```bash
curl http://localhost:8001/metrics
```

You should see output like:

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{endpoint="/metrics",method="GET",status="200"} 1.0
# HELP http_request_duration_seconds HTTP request latency in seconds
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{endpoint="/metrics",le="0.005",method="GET"} 1.0
...
# EOF
```

---

## Business metrics

Alongside the generic RED metrics, track domain-specific values. Business metrics are what let you correlate technical signals with user-visible behaviour.

In the order creation handler:

```python
from metrics import ORDERS_TOTAL, ORDERS_IN_FLIGHT

@app.post("/orders", status_code=201)
def create_order(req: OrderRequest):
    ORDERS_IN_FLIGHT.inc()
    try:
        # ... processing ...
        ORDERS_TOTAL.labels(status="created").inc()
        return order
    except HTTPException:
        ORDERS_TOTAL.labels(status="failed").inc()
        raise
    finally:
        ORDERS_IN_FLIGHT.dec()
```

`ORDERS_IN_FLIGHT` is a gauge that increases at the start of each order creation and decreases when it finishes — whether successfully or not. The `finally` block guarantees the gauge is always decremented. If this gauge grows unboundedly, workers are falling behind incoming requests. It is a leading saturation indicator before the queue backs up.

`ORDERS_TOTAL` with a `status` label gives you the rate of each outcome independently. You can graph the failed order rate separately from the HTTP 5xx rate — sometimes an application-level failure results in a 200 response with an error body, and you would miss it in the HTTP metrics alone.

---

## Wiring Prometheus

The same `metrics.py` module and middleware are added to all four services. Each service exposes the same `http_requests_total` and `http_request_duration_seconds` metrics; the `job` label from the Prometheus scrape config tells them apart.

Add all four services as scrape targets in `prometheus/prometheus.yml`:

```yaml
scrape_configs:
  # ... existing targets ...

  - job_name: order-service
    static_configs:
      - targets: ["order-service:8001"]

  - job_name: inventory-service
    static_configs:
      - targets: ["inventory-service:8002"]

  - job_name: payment-service
    static_configs:
      - targets: ["payment-service:8003"]

  - job_name: notification-service
    static_configs:
      - targets: ["notification-service:8004"]
```

Restart:

```bash
docker compose up -d
```

In Prometheus → Status → Targets, all four jobs should appear as **UP**. The `job` label makes it easy to query a single service or aggregate across all of them — `sum(rate(http_requests_total[5m])) by (job)` gives you per-service request rates in one query.

---

## Sending test traffic

Generate some requests to populate the metrics:

```bash
# Send a mix of small and large orders through the full service chain
for i in $(seq 1 30); do
  qty=$((RANDOM % 8 + 1))
  curl -s -X POST http://localhost:8001/orders \
    -H "Content-Type: application/json" \
    -d "{\"item\": \"book\", \"quantity\": $qty, \"price\": 9.99}" > /dev/null
done
```

Each `POST /orders` call drives downstream calls to `inventory-service`, `payment-service`, and `notification-service`, so all four services accumulate metrics from this single script.

Then in the Prometheus expression browser:

```promql
# Request rate per service
sum(rate(http_requests_total[5m])) by (job)

# Error fraction for order-service
rate(http_requests_total{job="order-service",status=~"5.."}[5m])
  /
rate(http_requests_total{job="order-service"}[5m])

# p99 latency for POST /orders on order-service
histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket{
    job="order-service",
    endpoint="/orders",
    method="POST"
  }[5m])
)

# p99 latency for inventory-service stock checks
histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket{
    job="inventory-service",
    endpoint="/stock/{item}"
  }[5m])
)
```

The p99 query will show the elevated latency from large-quantity orders bleeding into the tail. We will turn these into proper PromQL queries in post 4 and dashboard panels in post 5.

---

## What you have now

- **Rate**: `http_requests_total` — every request labelled by method, endpoint (normalized), and status
- **Errors**: the `status=~"5.."` subset of `http_requests_total` — no extra code, already in the middleware
- **Duration**: `http_request_duration_seconds` histogram with buckets tuned for the order service's expected latency range
- **Business metric**: `orders_total` (outcome counts) and `orders_in_flight` (concurrency gauge)
- Path normalization preventing cardinality explosion from dynamic route segments

In the next post we write PromQL queries that make these metrics useful: rates, quantiles, aggregations, and recording rules that precompute the expensive ones.
