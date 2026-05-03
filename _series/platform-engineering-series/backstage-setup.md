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
- Docker (for PostgreSQL in later posts; SQLite works locally and needs no setup)
- A GitHub account and a [personal access token](https://github.com/settings/tokens/new) with `repo` and `read:org` scopes

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

> 📷 **Screenshot** — Backstage home screen after `yarn dev`, showing the empty catalog and the left sidebar (Catalog, Create, Docs).

> **A note on versions**: Backstage releases new versions every two weeks. The `create-app` CLI always scaffolds the latest. Pin the version in `package.json` once you have a working setup — upgrading between versions is manageable but requires following the [changelog](https://backstage.io/docs/releases/v1-versioning-policy) carefully.

---

## The architecture in three layers

<figure><img src="/assets/images/platform-engineering-series/diagram-backstage-layers.svg" alt="Backstage in three layers: Your App on top, Plugins in the middle, Core at the foundation."></figure>

Backstage is built in three layers that are worth understanding before you start customising.

**The core** is maintained by the Backstage project. It provides the plugin framework, routing, authentication, and the catalog engine. You don't touch this.

**Plugins** are the units of functionality — the catalog, the scaffolder (templates), TechDocs, and any integrations. Each plugin is a separate npm package. The CLI installs a curated set by default; you add more as you need them. The [plugin marketplace](https://backstage.io/plugins) lists several hundred community and official plugins.

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

This is enough for local development. For production, replace the PAT with a [GitHub App](https://backstage.io/docs/integrations/github/github-apps) — the PAT is tied to your personal account, hits the 5,000 req/hr user rate limit, and breaks if you ever leave the org. A GitHub App authenticates as the app itself, gets 15,000 req/hr, and has no owner dependency. The setup takes about ten minutes: create the app in your org settings, generate a private key, and configure the `appId`, `clientId`, `clientSecret`, and `privateKey` in `app-config.yaml`. The Backstage docs walk through it step by step.

---

## Config files: local vs production

`app-config.yaml` is the base config checked into git. Sensitive values and environment-specific overrides go in `app-config.local.yaml` (gitignored) for local development, and `app-config.production.yaml` for production. Backstage merges them at startup — later files override earlier ones.

Keep secrets out of `app-config.yaml`. Use environment variable substitution (`${VAR}`) everywhere a secret appears.

---

## What we have so far

A running Backstage instance, GitHub connected, and a clear mental model of the three layers. In the next post we build the first Software Template — the mechanism that delivers the golden path.
