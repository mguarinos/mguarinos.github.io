---
title: "Production Backstage: PostgreSQL, authentication, and TechDocs at scale"
tags: [platform-engineering, backstage, postgresql, auth, techdocs, s3, production]
series_part: 9
toc: true
description: "The local setup from part 2 runs on SQLite with no auth. Production needs PostgreSQL, SSO, and pre-built TechDocs. Here's what changes and why each piece matters."
---

This is part 9 of the [Building a Golden Path with Backstage series](/series/#platform-engineering-series).

---

## What breaks at scale with the local setup

The local setup from [part 2](/posts/platform-engineering-series/backstage-setup/) deliberately keeps things simple. Three things hit hard limits as usage grows:

**SQLite** is single-writer. The moment two backend instances try to write catalog updates simultaneously — which happens the instant you run more than one pod — you get lock contention and data corruption. PostgreSQL is the only supported production database.

**No authentication** means the catalog is open to anyone who can reach the URL. Ownership data is useless if anyone can register as anyone. SSO maps catalog entities to real users and enables permission checks.

**On-demand TechDocs** builds documentation on every page load. With dozens of services and hundreds of catalog page views per day, this becomes a significant source of latency and backend CPU — and occasionally a hard failure when a repo's `mkdocs.yml` has a syntax error.

The companion repository (`docker-compose.yml`) runs PostgreSQL from the start, so if you've been following along, the database switch is already done. This post covers the auth and TechDocs pieces.

---

## PostgreSQL backend

The `docker-compose.yml` already runs PostgreSQL 15:

```yaml
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER:     ${POSTGRES_USER:-backstage}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB:       backstage
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U backstage"]
      interval: 5s
      timeout: 5s
      retries: 5
```

In `app-config.yaml`:

```yaml
backend:
  database:
    client: pg
    connection:
      host:     ${POSTGRES_HOST}
      port:     ${POSTGRES_PORT:-5432}
      user:     ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}
      database: backstage
      ssl:
        require:             true
        rejectUnauthorized:  false
```

Backstage runs migrations automatically on startup. The first start after switching from SQLite takes slightly longer as it creates the schema.

---

## Authentication with GitHub OAuth

Backstage ships with [several auth providers](https://backstage.io/docs/auth/). GitHub OAuth is the natural choice for teams already on GitHub — it maps GitHub usernames directly to catalog `User` entities, and group membership maps to catalog `Group` entities.

**1. Create a GitHub OAuth App**

In your GitHub org settings → Developer settings → OAuth Apps → New OAuth App:
- Homepage URL: `https://your-backstage-domain`
- Authorization callback URL: `https://your-backstage-domain/api/auth/github/handler/frame`

Note the Client ID and Client Secret.

**2. Configure `app-config.yaml`**

```yaml
auth:
  environment: production
  providers:
    github:
      production:
        clientId:     ${AUTH_GITHUB_CLIENT_ID}
        clientSecret: ${AUTH_GITHUB_CLIENT_SECRET}
        signIn:
          resolvers:
            - resolver: usernameMatchingUserEntityName
```

**3. Enable sign-in in the frontend**

In `packages/app/src/App.tsx`:

```tsx
import { githubAuthApiRef } from '@backstage/core-plugin-api';
import { SignInPage } from '@backstage/core-components';

const app = createApp({
  components: {
    SignInPage: props => (
      <SignInPage
        {...props}
        auto
        provider={{
          id: 'github-auth-provider',
          title: 'GitHub',
          message: 'Sign in using GitHub',
          apiRef: githubAuthApiRef,
        }}
      />
    ),
  },
});
```

With this in place, users authenticate with GitHub before they can view the catalog. Catalog `User` and `Group` entities populate from your GitHub org automatically via the GitHub Org provider.

> 📷 **Screenshot** — The Backstage sign-in page with the "Sign in using GitHub" button.

---

## Syncing GitHub org to catalog users and groups

The GitHub Org provider reads your org's teams and members and creates corresponding `User` and `Group` entities in the catalog. This is what makes `owner: payments-team` in a `catalog-info.yaml` resolve to a real group with real members.

```yaml
# app-config.yaml
catalog:
  providers:
    githubOrg:
      id: production
      githubUrl: https://github.com
      orgs: [your-org]
      schedule:
        frequency: { hours: 1 }
        timeout:   { minutes: 5 }
```

After the first sync, navigating to a `Group` entity shows all members, all components they own, and all systems they are responsible for.

---

## TechDocs: pre-build to S3

On-demand TechDocs builds work locally. In production, pre-build every service's docs in its own CI pipeline and publish to S3. Backstage serves the pre-built static files directly.

**`app-config.yaml` for production TechDocs:**

```yaml
techdocs:
  builder:   external
  publisher:
    type:    awsS3
    awsS3:
      bucketName: ${TECHDOCS_S3_BUCKET}
      region:     ${AWS_REGION}
      sse:        aws:kms
```

**CI step in the service repo (`.github/workflows/ci.yml`):**

```yaml
- name: Build and publish TechDocs
  if: github.ref == 'refs/heads/main'
  env:
    TECHDOCS_S3_BUCKET: ${{ vars.TECHDOCS_S3_BUCKET }}
    AWS_REGION:         ${{ vars.AWS_REGION }}
  run: |
    pip install mkdocs-techdocs-core
    npx @techdocs/cli publish \
      --publisher-type awsS3 \
      --storage-name $TECHDOCS_S3_BUCKET \
      --entity default/Component/${{ env.SERVICE_NAME }}
```

The TechDocs CLI builds the MkDocs site and uploads it to S3 under a key path Backstage knows to look up by entity reference. Every merge to main triggers a fresh doc build. No staleness, no on-demand build latency.

The IAM role used by the CI pipeline needs `s3:PutObject` on the TechDocs bucket. The `account-baseline` Terraform module from part 7 provisions this permission on the `cicd` role.

> 📷 **Screenshot** — The Docs tab on a service in production Backstage: content served from S3, noticeably faster than the on-demand build.

---

## Keeping Backstage up to date

Backstage releases new versions every two weeks. The upgrade path is documented in the [Backstage changelog](https://backstage.io/docs/releases/v1-versioning-policy), and the `@backstage/cli` ships a `versions:bump` command that updates all Backstage packages to their latest compatible versions:

```bash
yarn backstage-cli versions:bump
```

Run this monthly. Falling behind by more than a few versions makes the upgrade harder — plugin APIs change, and the diff becomes large. The `create-app` scaffolder pins exact versions; the best practice is to commit `yarn.lock` and treat Backstage upgrades as any other dependency update: a PR with a tested build.

---

## What the full production setup looks like

After this post, the stack is:

| Component | Setup |
|---|---|
| Backstage | Running on ECS Fargate, 2 tasks |
| Database | PostgreSQL 15 on RDS (or Aurora Serverless v2) |
| Auth | GitHub OAuth, org sync hourly |
| GitHub integration | GitHub App (15k req/hr, org-scoped) |
| TechDocs | Pre-built to S3 in each service's CI pipeline |
| Catalog discovery | GitHub org scan every 30 minutes |

The series has covered the full arc: from the problem, through local setup, Software Templates, the service catalog, TechDocs, service skeletons, infrastructure templates, API contracts, and production hardening. The companion repository at [github.com/mguarinos/backstage-golden-path](https://github.com/mguarinos/backstage-golden-path) has working implementations of everything described across all nine posts.
