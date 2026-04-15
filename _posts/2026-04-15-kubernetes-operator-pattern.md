---
layout: post
title: "The Kubernetes Operator Pattern: teaching your cluster to manage anything"
date: 2026-04-15
categories: [kubernetes, infrastructure]
tags: [kubernetes, operators, kopf, cloudflare, python, crd]
toc: true
description: "Operators extend Kubernetes' reconciliation model beyond pods and services to anything — DNS records, database users, cloud resources. Here's the mental model, the mechanics, and a concrete DNS operator to make it tangible."
---

When you run `kubectl apply`, nothing executes your manifest directly. The API server writes your desired state to etcd, and a control loop running somewhere in the cluster notices the gap between what you asked for and what currently exists — then closes it. That loop is a controller. A Deployment is a controller. A ReplicaSet is a controller. The entire Kubernetes architecture is built on this pattern.

An operator is what happens when you take that same pattern and point it at something you own: a database, a DNS record, an SSL certificate, a Slack channel. The operator extends Kubernetes' reconciliation model to resources that have nothing to do with running containers.

This post uses a Cloudflare DNS operator as the running example — an operator that watches `CloudflareDNSRecord` objects in the cluster and syncs them to the Cloudflare API. The source is [here](https://github.com/mguarinos/kubernetes-cloudflare-dns-operator).

---

## The problem operators solve

Helm charts and plain manifests handle static configuration well. You describe what you want, apply it, and Kubernetes makes it happen. But this works because Kubernetes itself knows how to reconcile pods, services, and config maps. It has no idea what a Cloudflare DNS record is.

The traditional answer is automation outside the cluster: a CI pipeline that calls `curl` against the Cloudflare API when a variable changes, a shell script someone runs manually, a Terraform workspace that drifts quietly for months. These work, but they all share the same problem: the external resource is not a first-class citizen in the cluster. You can't `kubectl get` it, you can't set a `dependsOn`, you can't see its status alongside your other resources. And when someone edits it directly in the Cloudflare dashboard, nothing notices.

Operators bring external resources inside Kubernetes' reconciliation boundary. Once you have an operator, a DNS record is just another Kubernetes object.

---

## CRDs: giving Kubernetes new vocabulary

Before you can write an operator, you need to teach Kubernetes what your resource type looks like. Custom Resource Definitions (CRDs) are the mechanism. A CRD is itself a Kubernetes manifest — you apply it once, and from that point on the API server accepts and stores objects of that type.

The DNS operator's CRD registers the `CloudflareDNSRecord` kind under `example.com/v1`. The CRD is a manifest like any other — you `kubectl apply` it once and the API server learns the new type:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: cloudflarednsrecords.example.com   # <plural>.<group>
spec:
  group: example.com
  scope: Namespaced
  names:
    plural: cloudflarednsrecords
    singular: cloudflarednsrecord
    kind: CloudflareDNSRecord
    shortNames: [cfdr]
  versions:
    - name: v1
      served: true
      storage: true
      # Status is a separate write path — the operator patches it without
      # triggering an on.update on the spec.
      subresources:
        status: {}
      # OpenAPI schema: the API server validates every object before storing it.
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: [zone_id, name, type, content]
              properties:
                zone_id: { type: string }
                name:    { type: string }
                type:
                  type: string
                  enum: [A, AAAA, CNAME, TXT, MX, NS, SRV, CAA]
                content: { type: string }
                ttl:     { type: integer, default: 1 }
                proxied: { type: boolean, default: false }
            status:
              type: object
              x-kubernetes-preserve-unknown-fields: true
```

After applying it, this becomes a valid object the API server will accept and store:

```yaml
apiVersion: example.com/v1
kind: CloudflareDNSRecord
metadata:
  name: my-a-record
  namespace: cf-operator
spec:
  zone_id: "abc123"
  name: "app.example.com"
  type: "A"
  content: "1.2.3.4"
  ttl: 1
  proxied: false
```

At this point nothing else happens — there is no controller yet watching these objects. That is the operator's job.

CRDs also define a `status` subresource, which is a separate write path from the spec. The operator uses it to record what it observed: the Cloudflare record ID it created, the last sync timestamp, and a `sync_status` field that can be `Healthy`, `Drift Detected`, or an error message. The subresource separation means the operator can patch status without triggering a reconciliation on the spec — there is no event loop between the two.

---

## The reconciliation loop

An operator is a process — typically a pod in the cluster — that watches the Kubernetes API for events on its custom resource type and reacts to them. The core of the DNS operator is four handlers:

**on.create** — when a `CloudflareDNSRecord` object appears, call `cf.dns.records.create`, then write the returned Cloudflare record ID into `.status.record_id`. That ID is the link between the Kubernetes object and the external resource. Without it, the operator cannot update or delete the record later.

**on.update** — when the spec changes, call `cf.dns.records.update` with the new values. If the status has no `record_id` (the operator was offline when the object was created), fall back to creating the record rather than failing.

**on.delete** — delete the Cloudflare record before allowing Kubernetes to remove the object. The finalizer (described below) is what makes this ordering possible.

**timer** — every 60 seconds, fetch the live record from Cloudflare and compare it to the spec. If they differ, revert Cloudflare to match Kubernetes.

Together these four handlers mean the operator never needs to be told what changed. It observes events and acts on them. If the operator crashes and restarts, the reconciliation loop catches up automatically — any pending creates become updates, any missed deletes are replayed.

This is the core promise of the operator pattern: **eventual consistency, not eventual hope**. You declare what you want and the operator drives toward it continuously.

<figure>
  <img src="/assets/images/operator/reconciliation-loop.svg" alt="The reconciliation loop: Watch → Compare → Reconcile → Patch Status, with a bypass arc for the 'matches' case and a feedback loop back to Watch">
  <figcaption>The four-step loop. When the live state matches the spec, the Reconcile step is skipped entirely (green arc). The loop repeats on every event and every 60-second timer tick.</figcaption>
</figure>

---

## Finalizers: the guarantee on delete

Without a finalizer, `kubectl delete` removes the Kubernetes object immediately and the Cloudflare record is left behind. Finalizers prevent that.

When the operator starts, it registers the string `dns.example.com/cloudflare-cleanup` as a finalizer on every object it manages. Kubernetes will not actually delete an object that has a finalizer on it — it only sets a `deletionTimestamp` and blocks. The API server then fires a delete event to the operator.

The operator's delete handler calls `cf.dns.records.delete`. If that call succeeds, the handler returns and the operator removes the finalizer. Kubernetes sees the finalizer list is now empty and removes the object. If the Cloudflare call fails — a network blip, a rate limit — the handler raises a temporary error and the framework retries it every 30 seconds. The finalizer stays in place until the deletion is confirmed.

The result: as long as the operator is running, it is impossible for a `kubectl delete` to leave an orphaned DNS record. The guarantee is structural, not procedural.

<figure>
  <img src="/assets/images/operator/finalizer.svg" alt="Without a finalizer: kubectl delete removes the object immediately, leaving the Cloudflare DNS record orphaned. With a finalizer: the object is blocked until the operator confirms the record is deleted from Cloudflare.">
  <figcaption>Without a finalizer, the Kubernetes object disappears before anything cleans up Cloudflare. The finalizer inverts the order: the external resource is deleted first, then Kubernetes removes the object.</figcaption>
</figure>

---

## Drift detection: Kubernetes as the source of truth

The timer handler is where the operator earns its keep.

Kubernetes stores the desired state. Cloudflare holds the live state. These can diverge whenever a human edits a record directly in the Cloudflare dashboard. Without an operator, that divergence is silent — your IaC says one thing, your DNS actually does another.

Every 60 seconds the operator fetches the record from Cloudflare and diffs `content`, `proxied`, and `ttl` against the spec. If anything differs, it logs the exact discrepancy and calls `cf.dns.records.update` to revert it. The status is patched to `Drift Detected` the moment divergence is found, and back to `Healthy` once the revert succeeds.

```
Drift detected on record id=a1b2c3 name=app.example.com:
  content : live='5.6.7.8'   desired='1.2.3.4'
  proxied : live=True         desired=False
Reverting to K8s spec.
```

This is what "Kubernetes is the source of truth" actually means in practice: not a policy, but a running process that enforces it. Any change made outside the cluster is overwritten within a minute.

---

## Authentication the cloud-native way

The operator needs a Cloudflare API token. The naive approach is an environment variable on the pod — which means the token appears in the Deployment manifest, which ends up in git. The cloud-native approach is a Kubernetes Secret.

```bash
kubectl create secret generic cloudflare-api-token \
  --from-literal=token=<YOUR_TOKEN> \
  -n cf-operator
```

The operator reads the secret at runtime using the Kubernetes API, not at startup from the environment. Two consequences: the token never appears in any manifest or shell history, and rotating it takes effect on the next API call without restarting the operator.

The RBAC Role for secret access uses `resourceNames: ["cloudflare-api-token"]` — the operator can get exactly that one secret and nothing else. The Cloudflare token scoped to `Zone → DNS → Edit` on the other side. Both ends are least-privilege.

---

## Choosing a framework

Operators in Go using [kubebuilder](https://book.kubebuilder.io/) or the [Operator SDK](https://sdk.operatorframework.io/) are the production standard. You get generated boilerplate, built-in status conditions, strong typing, and the full ecosystem of controller-runtime tooling. The tradeoff is ceremony: before writing a single line of business logic you are wiring up schemes, registering types, and configuring manager options.

[Kopf](https://kopf.readthedocs.io/) (Kubernetes Operator Pythonic Framework) flips that tradeoff. A handler is a decorated Python function. The framework handles watches, retries, status patching, finalizer registration, and leader-election. For an operator with a small surface area — a handful of handlers, one external API — the reduction in boilerplate is significant without giving up the important guarantees.

The DNS operator uses kopf. The entire operator logic fits in one file. For a team already fluent in Python and operating against well-understood Python SDKs (the Cloudflare client, boto3, the GitHub API), this is often the right call.

---

## When to write an operator

An operator is the right tool when:

- You have an external resource with a lifecycle (create, update, delete) that needs to track Kubernetes objects.
- You want drift detection — you need Kubernetes to be authoritative, not just a one-shot provisioner.
- The resource type is long-lived and managed by multiple people, where a CI script or manual Terraform run is too fragile.

An operator is overkill when you just need to run a job on deploy, transform a config value, or provision something once. A `Job`, a Helm hook, or an init container is simpler and easier to reason about.

The Kubernetes API is remarkably good at storing and watching state. An operator is how you make that machinery work for resources Kubernetes was never designed to know about.
