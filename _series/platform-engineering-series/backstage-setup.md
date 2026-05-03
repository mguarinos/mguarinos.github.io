---
title: "Getting Backstage running: local setup and the anatomy of the app"
tags: [platform-engineering, backstage, typescript, nodejs, developer-portal]
series_part: 2
toc: true
description: "Spin up Backstage locally, understand its layered architecture, and wire up the GitHub integration — the foundation everything else in this series builds on."
---

This is part 2 of the [Building a Golden Path with Backstage series](/series/#platform-engineering-series).

---

## Prerequisites

- Node.js 18+ and Yarn
- Docker (for the PostgreSQL backend in later posts; SQLite works locally)
- A GitHub account and a personal access token with `repo` and `read:org` scopes

---

## Creating a new Backstage app

Backstage ships a CLI that scaffolds a working app in one command:

```bash
npx @backstage/create-app@latest
```

You'll be prompted for an app name — use something like `my-portal`. The CLI generates a monorepo with two packages:

- `packages/app` — the React frontend
- `packages/backend` — the Node.js backend

Start it:

```bash
cd my-portal
yarn dev
```

Open `http://localhost:3000`. You'll see the Backstage home screen with an empty catalog.

---

## The architecture in three layers

Backstage is built in three layers that are worth understanding before you start customising.

**The core** is maintained by the Backstage project. It provides the plugin framework, routing, authentication, and the catalog engine. You don't touch this.

**Plugins** are the units of functionality — the catalog, the scaffolder (templates), TechDocs, and any integrations. Each plugin is a separate npm package. The CLI installs a curated set by default; you add more as you need them.

**Your app** is the configuration layer — which plugins to enable, how they're wired together, your organisation's branding, and the `app-config.yaml` that controls runtime behaviour. This is the code you own and commit.

---

## Wiring up GitHub

Open `app-config.yaml` and add your GitHub integration:

```yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}
```

Set the environment variable:

```bash
export GITHUB_TOKEN=your_pat_here
yarn dev
```

With this in place, Backstage can read catalog files from your GitHub repositories, which is the foundation for the service catalog in part 4.

---

## What we have so far

A running Backstage instance, GitHub connected, and a clear mental model of the three layers. In the next post we build the first Software Template — the mechanism that delivers the golden path.
