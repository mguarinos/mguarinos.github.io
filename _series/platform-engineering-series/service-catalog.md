---
title: "The service catalog: making your entire estate discoverable"
tags: [platform-engineering, backstage, service-catalog, catalog-info, developer-experience]
series_part: 4
toc: true
description: "Backstage's service catalog is a living inventory of every service, library, website, and pipeline in your organisation — discoverable, linked to ownership, and queryable without opening GitHub."
---

This is part 4 of the [Building a Golden Path with Backstage series](/series/#platform-engineering-series).

---

## The problem the catalog solves

In most organisations, the answer to "who owns this service?" is a Slack message. The answer to "what does this service depend on?" is reading the code. The answer to "is there a runbook?" is checking three different places.

The Backstage catalog is a machine-readable inventory of your entire software estate. Every service, library, API, pipeline, and team is an entity. Entities are linked — a component has an owner, depends on APIs, is part of a system. The result is a graph you can query without opening GitHub or asking anyone.

---

## The `catalog-info.yaml` file

Every entity is described by a `catalog-info.yaml` at the root of its repository. This is what the Software Template we built in part 3 generates automatically:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payments-service
  description: Handles payment processing and refunds
  annotations:
    github.com/project-slug: your-org/payments-service
    backstage.io/techdocs-ref: dir:.
  tags:
    - node
    - payments
    - critical
spec:
  type: service
  lifecycle: production
  owner: payments-team
  system: checkout-platform
  dependsOn:
    - component:orders-service
    - resource:payments-postgres
  providesApis:
    - payments-api
```

Backstage ingests this file and builds the graph. Change the file and the catalog updates automatically.

---

## Entity kinds

The catalog models your estate with a small set of kinds:

| Kind | What it represents |
|---|---|
| `Component` | A service, library, website, or CLI |
| `API` | An OpenAPI, GraphQL, or Async API spec |
| `Resource` | Databases, queues, S3 buckets, external services |
| `System` | A collection of components that deliver a capability |
| `Domain` | A grouping of related systems (mirrors DDD domains) |
| `Group` | A team or org unit |
| `User` | An individual engineer |

The relationships between these kinds — `dependsOn`, `partOf`, `ownedBy` — are what makes the catalog a graph rather than a list.

---

## Populating the catalog

There are two approaches.

**Pull from GitHub**: configure a discovery provider that scans your GitHub org for `catalog-info.yaml` files. Every repo that has one gets registered automatically. New repos created by the Software Template already have the file.

```yaml
catalog:
  providers:
    github:
      my-org:
        organization: your-org
        catalogPath: /catalog-info.yaml
        schedule:
          frequency: { minutes: 30 }
          timeout: { minutes: 3 }
```

**Manual registration**: for existing services, engineers visit the catalog UI and paste a GitHub URL. Backstage fetches and registers the file.

---

## What the catalog unlocks

Once populated, the catalog lets anyone answer questions that previously required Slack or archaeology:

- Which team owns this service?
- What APIs does this component expose?
- What depends on this database?
- Show me all services in the checkout platform that are in production.

Part 5 shows how TechDocs brings the runbooks, ADRs, and documentation for each service into the same portal — so the catalog entry and the docs live together.
