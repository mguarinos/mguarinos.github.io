---
title: "Service skeletons: Python and Node.js golden path in practice"
tags: [platform-engineering, backstage, python, nodejs, opentelemetry, docker, graceful-shutdown]
series_part: 6
toc: true
description: "The skeleton is what your Software Template scaffolds — a service that already handles signals, exposes health endpoints, ships structured logs with trace IDs, and builds to a minimal Docker image from commit one."
---

This is part 6 of the [Building a Golden Path with Backstage series](/series/#platform-engineering-series).

The companion repository for this series is at [github.com/mguarinos/backstage-golden-path](https://github.com/mguarinos/backstage-golden-path). The skeletons described here live in `skeletons/python-service/` and `skeletons/node-service/`.

---

## What a skeleton is responsible for

A skeleton is the template directory the Backstage scaffolder renders when a developer creates a new service. It is not a stub — it is a working, deployable service that already has everything production requires before the first line of business logic is written:

- Graceful shutdown on SIGTERM
- Liveness and readiness health endpoints
- Structured JSON logs with trace and span IDs injected
- OpenTelemetry instrumentation wired and ready
- A multi-stage Dockerfile that produces a minimal runtime image
- A CI pipeline that builds, tests, and pushes to ECR on merge

The developer adds their domain logic on top of a baseline that already passes security review.

---

## Graceful shutdown

This is the most commonly skipped piece and the most likely to cause incidents during deployments. When ECS (or Kubernetes) stops a task, it sends SIGTERM first. If the process does not shut down within the termination grace period (30 seconds by default), it receives SIGKILL — and in-flight requests are dropped.

**Python (FastAPI + uvicorn)**

```python
import asyncio
import signal
import uvicorn

class GracefulServer:
    def __init__(self):
        self.server = None
        self._shutdown = asyncio.Event()

    async def run(self):
        config = uvicorn.Config(
            app="main:app",
            host="0.0.0.0",
            port=int(os.getenv("PORT", 8000)),
        )
        self.server = uvicorn.Server(config)

        loop = asyncio.get_event_loop()
        loop.add_signal_handler(signal.SIGTERM, self._handle_shutdown)
        loop.add_signal_handler(signal.SIGINT, self._handle_shutdown)

        await self.server.serve()

    def _handle_shutdown(self):
        self.server.should_exit = True
```

Uvicorn drains in-flight requests when `should_exit` is set. The key is that SIGTERM must reach the Python process — not just the container. In Docker, use `CMD ["python", "main.py"]` (exec form), not a shell script, or the signal will be swallowed by the shell.

**Node.js (Express)**

```javascript
const server = app.listen(PORT, () => {
  console.log(JSON.stringify({ level: 'info', msg: `listening on ${PORT}` }));
});

const shutdown = async (signal) => {
  server.close(async () => {
    await db.end();   // close connection pool
    process.exit(0);
  });

  // force exit if drain takes too long
  setTimeout(() => process.exit(1), 25_000);
};

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT',  () => shutdown('SIGINT'));
```

---

## Health endpoints

Two endpoints with different semantics. The container orchestrator uses both.

| Endpoint | Fails when | Used by |
|---|---|---|
| `GET /health/live` | Process is stuck or deadlocked | Liveness probe — restarts the container |
| `GET /health/ready` | Database unreachable, dependency down | Readiness probe — removes from load balancer |

Never put dependency checks on `/health/live`. A flapping database would restart all your pods simultaneously.

```python
@app.get("/health/live")
async def liveness():
    return {"status": "ok"}

@app.get("/health/ready")
async def readiness(db: AsyncSession = Depends(get_db)):
    try:
        await db.execute(text("SELECT 1"))
        return {"status": "ok"}
    except Exception:
        raise HTTPException(status_code=503, detail="database unavailable")
```

---

## OpenTelemetry instrumentation

The skeleton uses OTel auto-instrumentation as a baseline and adds manual spans for business-critical paths.

**Python setup (`telemetry.py`)**

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

def setup_telemetry(app):
    provider = TracerProvider()
    provider.add_span_processor(
        BatchSpanProcessor(OTLPSpanExporter())  # reads OTEL_EXPORTER_OTLP_ENDPOINT
    )
    trace.set_tracer_provider(provider)

    FastAPIInstrumentor.instrument_app(app)
    HTTPXClientInstrumentor().instrument()
    SQLAlchemyInstrumentor().instrument()
```

The OTLP endpoint is configured via `OTEL_EXPORTER_OTLP_ENDPOINT`. Locally this points to an OTel Collector or directly to Tempo. In production it points to whatever backend the platform team runs.

**Adding a manual span**

```python
tracer = trace.get_tracer(__name__)

async def process_order(order_id: str):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        # business logic here
```

**Structured logging with trace context**

```python
import logging
import json
from opentelemetry import trace

class OtelJsonFormatter(logging.Formatter):
    def format(self, record):
        span = trace.get_current_span()
        ctx = span.get_span_context()
        return json.dumps({
            "level": record.levelname.lower(),
            "msg": record.getMessage(),
            "trace_id": format(ctx.trace_id, "032x") if ctx.is_valid else None,
            "span_id":  format(ctx.span_id,  "016x") if ctx.is_valid else None,
            "service": os.getenv("OTEL_SERVICE_NAME"),
        })
```

This means every log line carries the trace ID. In Loki + Grafana you can jump from a log line directly to the trace in Tempo.

---

## Multi-stage Dockerfile

```dockerfile
# ── build stage ──────────────────────────────────────────────────────────────
FROM python:3.12-slim AS build

WORKDIR /app
RUN pip install uv

COPY pyproject.toml .
RUN uv pip install --system --no-cache-dir .

COPY . .
RUN python -m pytest tests/ -q

# ── runtime stage ─────────────────────────────────────────────────────────────
FROM python:3.12-slim AS runtime

RUN adduser --disabled-password --no-create-home app
WORKDIR /app

COPY --from=build /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=build /app/src ./src

USER app
EXPOSE 8000

CMD ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

The test suite runs inside the build stage. If tests fail, the image does not build. No separate "run tests then build" step needed in CI — the Dockerfile is the unit of verification.

The Node.js Dockerfile follows the same pattern: `node:20-alpine` as build, `node:20-alpine` with only production deps in runtime, non-root user, exec-form CMD.

---

## Environment variables

Every skeleton reads configuration exclusively from environment variables. No config files, no defaults that hide missing values in production.

| Variable | Purpose | Required |
|---|---|---|
| `PORT` | HTTP server port | No (default 8000/3000) |
| `DATABASE_URL` | PostgreSQL connection string | Yes |
| `OTEL_SERVICE_NAME` | Service name in traces and logs | Yes |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTel Collector or backend endpoint | Yes |
| `LOG_LEVEL` | debug / info / warn / error | No (default info) |

In production these come from ECS task definition environment overrides, sourced from SSM Parameter Store via the task execution role. The Terraform `service-infra` module provisions the SSM parameters — the next post covers this.

---

## What the scaffolder produces

When a developer runs the `new-python-service` or `new-node-service` template in Backstage, they get a GitHub repository that:

1. Passes `docker build` immediately
2. Has a working CI pipeline on first push (builds, tests, pushes to ECR on merge to main)
3. Is registered in the Backstage catalog with TechDocs wired
4. Has a `docs/runbook.md` with deployment, rollback, and alert descriptions filled in with the service name

Part 7 shows how the template also provisions the AWS infrastructure the service will run on.
