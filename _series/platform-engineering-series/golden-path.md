---
title: "The golden path: why platform engineering exists"
tags: [platform-engineering, backstage, developer-experience, devops, idp]
series_part: 1
toc: true
description: "Every team reinventing its own deployment pipeline, security baseline, and repo setup is toil at scale. The golden path is the answer — one opinionated, working starting point that encodes your standards once."
---

This is part 1 of the [Building a Golden Path with Backstage series](/series/#platform-engineering-series).

---

## The problem at scale

When an engineering organisation is small, tribal knowledge works. The senior engineer who wired up the first CI pipeline answers questions on Slack. The person who set up branch protection knows why those rules exist. New services get bootstrapped by copying an old one and hoping nothing important got missed.

At twenty engineers this is annoying. At fifty it becomes a bottleneck. At a hundred it breaks.

The symptoms are recognisable: teams take a week to get a new service to production because nobody documented the steps. Security review finds three repos without secret scanning because they were created before someone added the org-wide policy. An on-call engineer can't find where a service's runbook lives because every team chose a different convention. A new hire spends their first sprint chasing down Terraform state setup rather than writing product code.

---

## What a golden path is

A golden path is the opinionated, pre-approved way to do the common things: create a new service, set up CI/CD, configure observability, provision infrastructure. It is not the only way — it is the way your organisation has decided works, encoded so anyone can follow it without asking for help.

The concept comes from Spotify's engineering culture. Their insight was that developers should not have to choose between freedom and paved roads. A golden path narrows the decision space for undifferentiated work — repo scaffolding, pipeline wiring, observability setup — so engineers can spend their cognitive budget on the problems that actually matter.

---

## Why an Internal Developer Platform

An Internal Developer Platform (IDP) is the infrastructure that delivers the golden path. It is not a wiki, a runbook, or a Slack channel. It is a system that can execute: given a developer's intent ("I want a new Node.js microservice"), it provisions the outcome — repo created, CI wired, secrets injected, service registered — with no manual steps.

Backstage is the most widely adopted open-source foundation for building an IDP. It was created at Spotify, open-sourced in 2020, and is now a CNCF incubating project with adoption at American Airlines, Zalando, Expedia, and hundreds of other engineering organisations.

The next post sets up Backstage locally so you can see the pieces before we start building.
