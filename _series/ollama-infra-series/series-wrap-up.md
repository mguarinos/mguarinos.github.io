---
title: "Ollama infra series wrap-up: architecture, hardware lessons, and what to add next"
tags: [ollama, architecture, self-hosting, wrap-up]
series_part: 9
toc: true
description: "The full architecture diagram for the Ollama stack we built across 8 posts, hardware lessons from running it, and a prioritized list of what to tackle next."
---

This is the final post in the [Ollama infra series](/series/#ollama-infra-series). Across eight posts we went from a bare Ollama install to an observable, queue-backed inference stack with a chat UI, voice interface, Claude Code integration, and a set of Open WebUI extensions. Here is the full picture and honest lessons from running it.

---

## The full architecture

```
                        INTERNET
                            │
                   ┌────────▼────────┐
                   │   nginx (443)   │
                   │   TLS termination│
                   │   Auth header   │
                   └────┬───────┬───┘
                        │       │
           ┌────────────▼─┐   ┌─▼────────────────┐
           │  Open WebUI  │   │    Job API        │
           │  (Docker)    │   │  POST /jobs       │
           │  Port 8080   │   │  GET  /jobs/:id   │
           └──────┬───────┘   └────────┬──────────┘
                  │                    │
           ┌──────▼───────┐    ┌───────▼──────┐
           │   speaches   │    │  BullMQ      │
           │  STT + TTS   │    │  Queue       │
           │  Port 8081   │    └───────┬──────┘
           └──────────────┘            │
                               ┌───────▼──────┐
                               │  Worker(s)   │
                               │  concurrency │
                               └───────┬──────┘
                                       │
                  └──────────┬─────────┘
                             │
                    ┌────────▼────────┐
                    │     Ollama      │
                    │  127.0.0.1:11434│
                    │  model cache    │
                    └─────────────────┘

Supporting infrastructure:
  Redis  ──────── BullMQ backing store
  Prometheus ───── Metrics scrape target
  Grafana  ──────── Dashboard
  Metrics exporter ── custom /metrics endpoint
  Claude Code + MCP ── developer tool integration
```

---

## What each layer does and why it exists

**nginx**: TLS termination, routing between UI and API, access control. Without it, Ollama is unauthenticated on `0.0.0.0`.

**Open WebUI**: interactive chat for humans. Handles session management, conversation history, and model pulling UI. Communicates with Ollama directly — bypasses the queue because interactive use benefits from streaming, which the queue does not support.

**speaches**: voice layer sitting alongside Open WebUI. Whisper transcribes microphone input; the Kokoro TTS engine reads responses aloud. Both run in a single container, no audio leaves the machine.

**Job API**: async interface for programmatic callers. Returns a job ID immediately, decouples caller lifetime from inference duration. The right interface for automation, pipelines, and batch work.

**BullMQ + Redis**: backpressure and durability. Jobs survive worker restarts. Queue depth is a real-time signal of system load.

**Worker**: the only process that calls Ollama. Configurable concurrency. Emits metrics on completion.

**Ollama**: the inference runtime. Manages model loading, VRAM allocation, and quantization. Exposes a local HTTP API.

---

## Hardware lessons

**VRAM is the primary constraint**, not CPU. The model must fit in VRAM or performance degrades sharply (layers offloaded to RAM). Budget for the model size you want to run, not the model size that fits.

**Model loading takes real time**. A 7B model cold start is 5–15 seconds. The `OLLAMA_KEEP_ALIVE` setting is the most important tuning parameter if you have multiple models. Keep the most-used model loaded permanently, let others expire.

**Concurrency has a ceiling**. Running 4 parallel inferences on a single GPU does not give 4× the throughput — the GPU is already saturated. The optimal concurrency value is usually 1–2 for a single GPU. Higher values increase queueing latency without improving throughput.

**CPU fallback is viable for small models**. A 3B model on a modern CPU produces 5–15 tokens/second — slow but usable for background batch work. Do not rule out CPU-only machines for low-priority tasks.

**Quantization matters more than parameter count for memory**. A Q4_K_M quantized 7B model uses ~4.5 GB of VRAM. The same model at Q8_0 uses ~7 GB. Quality difference is small. Run Q4_K_M unless you have a reason not to.

---

## Limitations of what we built

**The queue does not support streaming**. Interactive applications that want token-by-token output cannot use BullMQ as-is. Open WebUI connects directly to Ollama for this reason. If you want streaming in your own app, call Ollama directly and manage connection lifetime yourself.

**The metrics exporter uses in-memory accumulators**. If the worker process restarts, counters reset. For production, use a proper Prometheus push gateway or a time-series database like InfluxDB.

**The shared secret is not suitable for production**. The `X-Ollama-Key` header is a static token shared across all callers — there is no per-caller identity, no revocation, and no audit trail. For production, replace it with short-lived JWT tokens per client, mutual TLS, or an API gateway that supports per-key rate limits and revocation.

**The job polling endpoint has no authorization**. Any caller with the API key can poll any job ID, and IDs are sequential integers — a caller can enumerate them and read results belonging to others. For multi-tenant use, associate each job with the submitting caller's identity and enforce ownership in the `GET /jobs/:id` handler before returning the result.

**No horizontal scaling**. The current setup is single-machine. Scaling out means either a single Ollama instance behind a load balancer (multiple workers, one Ollama), or multiple Ollama instances with a smarter routing layer that tracks per-instance queue depth and loaded models.