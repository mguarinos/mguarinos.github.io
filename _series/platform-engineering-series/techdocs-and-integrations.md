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

[TechDocs](https://backstage.io/docs/features/techdocs/) is Backstage's docs-as-code system. It renders [MkDocs](https://www.mkdocs.org/)-style Markdown from your repositories and surfaces it inside the catalog entry for each service. The goal is simple: when an engineer opens a service in Backstage, they see the runbook, the architecture decision records, and the API docs without leaving the portal.

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

Backstage will render `docs/` on every catalog page load. This is fine for local development, but for production you want to pre-build and publish to an S3 bucket (or GCS) using the [TechDocs CLI](https://backstage.io/docs/features/techdocs/cli) in CI. That way the portal serves pre-built static files instead of building on demand, which is both faster and more reliable.

> 📷 **Screenshot** — The Docs tab on a component page: rendered markdown content with the left-hand nav showing "Home", "Runbook", "Architecture" sections.

### What to put in TechDocs

The documents most worth having here are the ones engineers reach for during incidents:

- **Runbook**: how to deploy, rollback, and restart the service
- **On-call guide**: alert meanings, escalation paths, known failure modes
- **Architecture decision records**: why the database is PostgreSQL and not MongoDB
- **Dependencies**: external services, rate limits, circuit breaker behaviour

---

## GitHub Actions integration

The [`@backstage/plugin-github-actions`](https://github.com/backstage/backstage/tree/master/plugins/github-actions) plugin surfaces workflow runs directly on a component's catalog page. Engineers see recent pipeline runs, their statuses, and can trigger reruns without leaving the portal.

Install the plugin in `packages/app`:

```bash
yarn --cwd packages/app add @backstage/plugin-github-actions
```

Wire it into the entity page in `packages/app/src/components/catalog/EntityPage.tsx`:

```tsx
import { EntityGithubActionsContent, isGithubActionsAvailable } from '@backstage/plugin-github-actions';

// Add a CI/CD tab to the service entity page:
<EntityLayout.Route path="/ci-cd" title="CI/CD" if={isGithubActionsAvailable}>
  <EntityGithubActionsContent />
</EntityLayout.Route>
```

The `github.com/project-slug` annotation on the `catalog-info.yaml` drives which repo's workflows are shown — which is already there if your services were scaffolded with the template from part 3.

> 📷 **Screenshot** — The CI/CD tab on a component page: recent workflow runs with green/red status badges, branch name, commit SHA, and duration.

---

## Closing the loop with Terraform

The Software Template from part 3 can provision Terraform-managed infrastructure as part of scaffolding. Add a step that dispatches a GitHub Actions workflow which runs `terraform apply`:

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

The template creates the repo, the pipeline provisions the database and IAM roles, and the whole flow completes before the developer opens their IDE. This pattern pairs naturally with the [GitHub OIDC + Terraform setup](/posts/2026/04/20/github-aws-oidc-cicd/) covered earlier on this blog.

---

## Production: switch to a GitHub App

If you're still using a PAT from part 2, now is the time to replace it. At production scale — hundreds of services being discovered, pipelines surfaced, TechDocs rendered — you will hit the PAT's 5,000 req/hr rate limit and have no good path to recovery without breaking the portal.

A [GitHub App](https://backstage.io/docs/integrations/github/github-apps) gives you:

- 15,000 req/hr (installation-based, not user-based)
- Org-level identity not tied to any individual account
- Fine-grained permissions — read-only on repos by default
- Clean audit logs in GitHub that show the app, not a person

To create one: go to your GitHub org settings → Developer settings → GitHub Apps → New GitHub App. Grant it `Contents: Read`, `Metadata: Read`, `Actions: Read` (and `Contents: Write` if your templates need to push code). Generate a private key, then configure Backstage:

```yaml
integrations:
  github:
    - host: github.com
      apps:
        - appId: ${GITHUB_APP_ID}
          clientId: ${GITHUB_APP_CLIENT_ID}
          clientSecret: ${GITHUB_APP_CLIENT_SECRET}
          webhookSecret: ${GITHUB_APP_WEBHOOK_SECRET}
          privateKey: ${GITHUB_APP_PRIVATE_KEY}
```

---

## Where to go from here

This series covered the core Backstage loop: the problem, local setup, Software Templates, the service catalog, and TechDocs with CI/CD integration. A few directions worth exploring next:

- **Authentication and SSO**: Backstage ships with no auth by default. For production, wire up [GitHub OAuth, Google, or Okta](https://backstage.io/docs/auth/) so catalog ownership maps to real users and groups.
- **Custom plugins**: if your organisation has internal tooling — a cost dashboard, a feature flag UI, a deployment approval flow — [Backstage plugins](https://backstage.io/docs/plugins/create-a-plugin) bring them into the same surface.
- **Kubernetes plugin**: the [`@backstage/plugin-kubernetes`](https://backstage.io/docs/features/kubernetes/) plugin surfaces pod health, resource usage, and deployment status directly on catalog pages.
- **Metrics and SLOs**: combined with the [observability stack from this blog's earlier series](/series/#observability-series), you can surface SLO burn rates on catalog pages — reliability visible without context switching.

The catalog, the templates, and TechDocs are a viable starting point for most teams. Deploy it, import your most critical services first, and build from there.
