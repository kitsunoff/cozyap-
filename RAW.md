# Cozystack User Apps

## What is it

An application catalog for end-user tenant clusters built on top of Cozystack. A Pterodactyl-like management panel for ISP customers. Users don't need to know Kubernetes — they just install apps from the catalog.

## Target Audience

Clients of ISP providers who have been given a tenant cluster via Cozystack.

## Core Architecture

**Two abstractions:**

`ApplicationTemplate` — PKGBUILD-inspired spec. Cluster-scoped on the host cluster. Readonly for tenants. Describes application lifecycle hooks.

`Application` — instance of a specific application. Namespace-scoped, lives inside the user's tenant cluster.

**Hooks:**
- `install` / `upgrade` / `remove`
- `suggestUpgrade` — periodically checks for available updates
- Custom lifecycle hooks

**Each hook is a set of steps:**
```yaml
hooks:
  install:
    steps:
      - name: deploy
        image: alpine/helm:3.14
        args: [helm, install, "{{ .App.Name }}"]
```

Image + args. No magic. Executed as a Job inside the tenant cluster.

**DAG engine:**

Argo Workflows runs inside the tenant cluster and executes steps. Isolation is natural — one tenant does not affect another. Sufficient for MVP, replaceable with a minimal custom Job runner if needed.

## Key Property

All the mess happens inside the tenant cluster. The ISP host cluster stays clean.

## Platform Integrations

The controller automatically appends platform-level steps to any hook — observability (Prometheus, Grafana, Loki) and Vault for secrets. AppTemplate authors don't write them, users don't see them.

## Community Catalog

AUR-inspired model. Templates are written by the community, reviewed, and published to the catalog. Portable — works on any Cozystack cluster.

**Initial demo applications:**
- Minecraft
- CS2
- Node.js app with auto build

## Extension into Dev Platform

ApplicationTemplate naturally extends into a dev platform — a `build` hook is added with Kaniko, and `suggestUpgrade` watches git commits and tags. CozyDP is simply a separate set of AppTemplates with `source.type: git`. One controller, one abstraction, two use cases.

## Why Not One CRD Per Application

The current Cozystack model uses one CRD per application type. This is expensive: CRDs are cluster-wide, flood the API server, and require cluster-admin for tenants to manage their own apps. ApplicationTemplate solves this with a single universal abstraction.