---
title: "Software Templates: the golden path in action"
tags: [platform-engineering, backstage, scaffolder, templates, automation, devops]
series_part: 3
toc: true
description: "Software Templates are how Backstage delivers the golden path — a developer fills in a form, and a fully wired service comes out the other side. Here's how to write one from scratch."
---

This is part 3 of the [Building a Golden Path with Backstage series](/series/#platform-engineering-series).

---

## What a Software Template does

A Software Template is a Backstage entity that combines three things: a form the developer fills out, a file template rendered with the form values, and a sequence of actions that execute on submission — creating a GitHub repo, registering the service in the catalog, triggering a first pipeline run.

The developer sees a wizard. The platform team defines what happens. The output is a production-ready starting point, not a blank repo.

> 📷 **Screenshot** — The scaffolder wizard, step 1: "Service name" and "Owning team" fields filled in, "Next" button visible.

---

## Template anatomy

A template is a YAML file with the `Template` kind, registered in the catalog like any other entity. It has three sections:

**`parameters`** defines the form fields — strings, dropdowns, checkboxes. Each field maps to a variable available in the steps. Backstage uses [react-jsonschema-form](https://rjsf-team.github.io/react-jsonschema-form/) under the hood, so any JSON Schema constraint works.

**`steps`** is the action sequence. Backstage ships [built-in actions](https://backstage.io/docs/features/software-templates/builtin-actions) for fetching a template directory, publishing to GitHub, registering in the catalog, and triggering webhooks. You can write custom actions for anything else.

**`output`** defines what links to surface to the developer when the template finishes — the new repo URL, the catalog entry, the first pipeline run.

---

## A minimal Node.js service template

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: nodejs-service
  title: Node.js Microservice
  description: Creates a new Node.js service with CI, Docker, and observability pre-wired
spec:
  owner: platform-team
  type: service

  parameters:
    - title: Service details
      required: [name, description]
      properties:
        name:
          title: Service name
          type: string
          pattern: '^[a-z][a-z0-9-]*$'
        description:
          title: Description
          type: string
        owner:
          title: Owning team
          type: string
          ui:field: OwnerPicker
          ui:options:
            allowedKinds: [Group]

  steps:
    - id: fetch-template
      name: Fetch template files
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          description: ${{ parameters.description }}
          owner: ${{ parameters.owner }}

    - id: publish
      name: Publish to GitHub
      action: publish:github
      input:
        allowedHosts: [github.com]
        repoUrl: github.com?owner=your-org&repo=${{ parameters.name }}
        defaultBranch: main
        description: ${{ parameters.description }}

    - id: register
      name: Register in catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml

  output:
    links:
      - title: Repository
        url: ${{ steps.publish.output.remoteUrl }}
      - title: Open in catalog
        entityRef: ${{ steps.register.output.entityRef }}
```

---

## The skeleton directory

The `skeleton/` directory next to the template YAML contains your file templates. Variables use the `${{ values.name }}` syntax:

```
skeleton/
├── catalog-info.yaml
├── package.json
├── Dockerfile
├── .github/
│   └── workflows/
│       └── ci.yml
└── src/
    └── index.js
```

<figure><img src="/assets/images/platform-engineering-series/diagram-template-rendering.svg" alt="The skeleton/ directory with template variables is rendered with form values to produce the output repository."></figure>

Every file in the skeleton is rendered with the values the developer provided. The output is a repo that compiles, passes lint, and has a working pipeline from commit one.

A short skeleton `catalog-info.yaml` that uses the template values:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ${{ values.name }}
  description: ${{ values.description }}
  annotations:
    github.com/project-slug: your-org/${{ values.name }}
    backstage.io/techdocs-ref: dir:.
spec:
  type: service
  lifecycle: production
  owner: ${{ values.owner }}
```

---

## Running the template

After registering the template in the catalog, developers find it under **Create** in the Backstage sidebar. They fill in the form, review the step log as actions execute, and get links to the new repo and catalog entry when it completes.

> 📷 **Screenshot** — Scaffolder execution log with all steps showing green checkmarks, and the "Repository" and "Open in catalog" links at the bottom.

---

## What this unlocks

A developer can go from "I need a new service" to a repository with CI, Docker build, and catalog registration in under two minutes. The platform team encoded those standards once. Every service that follows the golden path is automatically consistent.

Part 4 shows how the service catalog makes all of these services discoverable once they exist.
