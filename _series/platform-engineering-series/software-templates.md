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

A Software Template is a Backstage entity that combines three things: a form the developer fills out, a Cookiecutter-style file template, and a sequence of actions that execute when the form is submitted — creating a GitHub repo, registering the service in the catalog, triggering a first pipeline run.

The developer sees a wizard. The platform team defines what happens. The output is a production-ready starting point, not a blank repo.

---

## Template anatomy

A template is a YAML file with the `Template` kind. It has three sections:

**`parameters`** defines the form fields — strings, dropdowns, checkboxes. Each field maps to a variable available in the steps.

**`steps`** is the action sequence. Backstage ships built-in actions for fetching a template directory, publishing to GitHub, registering in the catalog, and triggering webhooks. You can write custom actions for anything else.

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

Every file in the skeleton is rendered with the values the developer provided. The output is a repo that compiles, passes lint, and has a working pipeline from commit one.

---

## What this unlocks

A developer can go from "I need a new service" to a repository with CI, Docker build, and catalog registration in under two minutes. The platform team encoded those standards once. Every service that follows the golden path is automatically consistent.

Part 4 shows how the service catalog makes all of these services discoverable once they exist.
