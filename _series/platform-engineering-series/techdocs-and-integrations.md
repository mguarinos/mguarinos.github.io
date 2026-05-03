---
title: "TechDocs and GitHub Actions: docs-as-code and pipeline integration"
tags: [platform-engineering, backstage, techdocs, github-actions, terraform, integrations]
series_part: 5
toc: true
description: "TechDocs brings your runbooks and ADRs into Backstage from Markdown. GitHub Actions integration closes the loop — templates trigger pipelines, and pipeline status surfaces in the catalog."
---

This is part 5 of the [Building a Golden Path with Backstage series](/series/#platform-engineering-series).

---

## TechDocs: documentation that lives with the code

TechDocs is Backstage's docs-as-code system. It renders MkDocs-style Markdown from your repositories and surfaces it inside the catalog entry for each service. The goal is simple: when an engineer opens a service in Backstage, they see the runbook, the architecture decision records, and the API docs without leaving the portal.

### Setting up TechDocs

Add a `mkdocs.yml` at the repo root:

```yaml
site_name: Payments Service
nav:
  - Home: index.md
  - Runbook: runbook.md
  - Architecture: adr/
docs_dir: docs
```

Add the annotation to your `catalog-info.yaml`:

```yaml
annotations:
  backstage.io/techdocs-ref: dir:.
```

Backstage will render `docs/` on every catalog page load. In production you'll want to pre-build and publish to an S3 bucket so the portal doesn't build on demand — but for local development the basic setup works immediately.

### What to put in TechDocs

The documents most worth having here are the ones engineers look for during incidents:

- **Runbook**: how to deploy, rollback, and restart the service
- **On-call guide**: alert meanings, escalation paths, known issues
- **Architecture decision records**: why the database is PostgreSQL and not MongoDB
- **Dependencies**: external services, rate limits, known failure modes

---

## GitHub Actions integration

The `@backstage/plugin-github-actions` plugin surfaces workflow runs directly on a component's catalog page. Engineers see the last ten pipeline runs, their statuses, and can trigger reruns without leaving the portal.

Install the plugin in `packages/app`:

```bash
yarn add @backstage/plugin-github-actions
```

Wire it into the entity page:

```tsx
import { EntityGithubActionsContent } from '@backstage/plugin-github-actions';

// Inside your EntityPage component, add a tab:
<EntityLayout.Route path="/ci-cd" title="CI/CD">
  <EntityGithubActionsContent />
</EntityLayout.Route>
```

The annotation on the `catalog-info.yaml` drives which repo's workflows are shown:

```yaml
annotations:
  github.com/project-slug: your-org/payments-service
```

---

## Closing the loop with Terraform

The Software Template from part 3 can provision Terraform-managed infrastructure as part of scaffolding. Add a step that triggers a GitHub Actions workflow which runs `terraform apply`:

```yaml
- id: trigger-terraform
  name: Provision infrastructure
  action: github:actions:dispatch
  input:
    repoUrl: github.com?owner=your-org&repo=platform-infra
    workflowId: provision-service.yml
    branchOrTagName: main
    workflowInputs:
      service_name: ${{ parameters.name }}
      environment: dev
```

The template creates the repo, the pipeline provisions the database and IAM roles, and the whole flow completes before the developer opens their IDE.

---

## Where to go from here

This series covered the core Backstage loop: the problem, local setup, Software Templates, the service catalog, and TechDocs with CI/CD integration. A few directions worth exploring next:

- **Authentication and user management**: Backstage supports GitHub, Google, and Okta SSO for controlling who can see and do what.
- **Custom plugins**: if your organisation has internal tooling — a cost dashboard, a feature flag UI, a deployment approval flow — Backstage plugins bring them into the same surface.
- **Kubernetes plugin**: surfaces pod health, resource usage, and deployment status directly on catalog pages for services running in Kubernetes.
- **Metrics and SLOs**: combined with the observability stack from this blog's earlier series, you can surface SLO burn rates on catalog pages, making reliability visible without context switching.

The catalog, the templates, and TechDocs are a viable starting point for most teams. Deploy it, import your most critical services first, and build from there.
