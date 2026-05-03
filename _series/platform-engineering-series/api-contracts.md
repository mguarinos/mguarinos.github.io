---
title: "API contracts and the dependency graph: making service relationships visible"
tags: [platform-engineering, backstage, openapi, service-catalog, api, dependency-graph]
series_part: 8
toc: true
description: "When every service defines its APIs and dependencies in catalog-info.yaml, Backstage builds a live dependency graph of your entire estate. Here's how to make that graph accurate and useful."
---

This is part 8 of the [Building a Golden Path with Backstage series](/series/#platform-engineering-series).

---

## Why APIs are first-class entities

In part 4 we covered the catalog's entity kinds. The `API` kind is the one most teams skip, and it's the one that makes the catalog genuinely useful for answering questions like: "if I change the response shape of this endpoint, what breaks?"

When a component declares `providesApis: [payments-api]` and another declares `consumesApis: [payments-api]`, Backstage builds that edge in the graph. The catalog page for `payments-api` shows every consumer. Before you change the API, you know exactly who to coordinate with.

---

## Defining an API entity

An `API` entity points to an OpenAPI spec — either inline or by URL. The skeleton services ship with a `openapi.yaml` at the repo root, and the `catalog-info.yaml` references it:

```yaml
# catalog-info.yaml (in the service repo)
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: payments-api
  description: Payment processing and refund API
  tags: [rest, payments, critical]
spec:
  type: openapi
  lifecycle: production
  owner: payments-team
  definition:
    $text: ./openapi.yaml
```

Register this alongside the Component entity. One repo can contain both — the Component (the service itself) and the API (the contract it exposes).

The `$text` reference tells Backstage to fetch the file from the same repo. It renders the spec in the Backstage API tab using Swagger UI, no external hosting needed.

> 📷 **Screenshot** — The API entity page in Backstage: the Swagger UI rendering the OpenAPI spec, with endpoints, request/response schemas, and the "Consumers" section listing which components consume this API.

---

## Wiring consumers and producers in catalog-info.yaml

**The service that exposes the API:**

```yaml
spec:
  type: service
  lifecycle: production
  owner: payments-team
  system: checkout-platform
  providesApis:
    - payments-api
  dependsOn:
    - resource:payments-db
    - resource:redis-cache
```

**The service that consumes it:**

```yaml
spec:
  type: service
  lifecycle: production
  owner: orders-team
  system: checkout-platform
  consumesApis:
    - payments-api
  dependsOn:
    - component:inventory-service
```

After both catalog-info files are ingested, Backstage renders the full relationship graph on each entity page.

> 📷 **Screenshot** — The "Relations" graph on a component page: nodes for Component, API, Resource, connected by labelled directed edges (providesApis, consumesApis, dependsOn).

---

## The OpenAPI spec skeleton

The skeleton services ship with a minimal but valid `openapi.yaml`:

```yaml
openapi: "3.0.3"
info:
  title: ${{ values.name }}
  version: "1.0.0"
  description: ${{ values.description }}
paths:
  /health/live:
    get:
      summary: Liveness probe
      responses:
        "200": { description: Service is alive }
  /health/ready:
    get:
      summary: Readiness probe
      responses:
        "200": { description: Service is ready }
        "503": { description: Dependency unavailable }
```

The developer extends this as they add routes. Because the spec lives in the repo and Backstage fetches it on each sync, the catalog always reflects the current API shape — no separate documentation pipeline needed.

---

## Enforcing contracts in CI

The CI pipeline validates the OpenAPI spec on every PR:

```yaml
- name: Validate OpenAPI spec
  run: npx @redocly/cli lint openapi.yaml
```

And runs contract tests against it:

```yaml
- name: Contract tests
  run: npx @schemathesis/cli run openapi.yaml --url http://localhost:8000
```

[Schemathesis](https://schemathesis.io/) generates test cases from the spec and runs them against the live service in CI. It catches cases where the implementation drifts from the spec — before the PR merges, not after a consumer breaks.

---

## The dependency graph in practice

Once a handful of services are in the catalog with APIs and dependencies declared, the graph becomes genuinely useful:

- **Impact analysis**: "payments-service is degraded — which upstream services are affected?" Follow `consumesApis` edges upward.
- **Deprecation planning**: "we want to shut down v1 of the inventory API — who still uses it?" Filter by `consumesApis: inventory-api-v1`.
- **Onboarding**: a new engineer opens the system page for `checkout-platform` and sees every service, API, database, and their relationships in one view.

The graph is only as accurate as the `catalog-info.yaml` files. The key is having the Software Template generate an accurate file from day one — then developers update it as the service evolves, because it is in the repo alongside the code.

---

## Adding an API kind to the Software Template

Add a second entity registration step to the `new-python-service` and `new-node-service` templates:

```yaml
- id: register-api
  name: Register API in catalog
  action: catalog:register
  input:
    repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
    catalogInfoPath: /catalog-api.yaml
```

Where `catalog-api.yaml` in the skeleton defines the `API` kind entity. The developer gets both the Component and the API entity registered in one template run.

Part 9 covers hardening Backstage itself for production — switching to PostgreSQL, adding authentication, and pre-building TechDocs to S3.
