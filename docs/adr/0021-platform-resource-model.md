# ADR-0021: Platform Resource Model

## Status

Proposed, 2026-09-14. Supersedes ADR-0001 through ADR-0020 and
`docs/design/brick-model.md`. Carry-over conclusions are in `docs/prior-art.md`.

Driven by requirement 5.14 (`docs/requirements/5.14-developer-platform.md`) and
the product owner constraints (`docs/requirements/platform-vision.md`).

## Context

Three previous designs failed in the same way: each introduced a general
mechanism — lifecycle hooks, atoms with a bespoke execution layer, bricks with
five implementation kinds — before any of them had a user. Each mechanism then
had to answer questions no requirement had asked.

Two things are different now.

**The substrate is known.** Cozystack already provides the managed applications
requirement R2 lists, a registry, ingress and certificates in tenant clusters,
observability, a secret store, a package format, and Flux driving all of it. The
platform does not need a provisioning engine; it needs the layer that runs
customer code on top of what exists.

**The shape is known.** Platforms that attach an arbitrary-workload layer to an
existing managed-service API converge on one structure: a thin orchestrator over
a small set of purpose-built controllers, with service access declared by the
two parties rather than centrally. Korifi and OpenChoreo are both this shape.
This ADR adopts it deliberately rather than deriving a new one.

## Decision

### 1. Six objects, three of them authored by packages

| Object | Scope | Written by | Purpose |
| --- | --- | --- | --- |
| `NodeType` | cluster | package | A reusable building block: what it creates, when it is ready, what it publishes |
| `Blueprint` | cluster | package | The graph: a parameter schema plus nodes wired by references |
| `BindingProfile` | cluster | package | How to extract connection details from a managed service |
| `BlueprintInstance` | namespace | user | An application: parameters in, aggregated graph status out |
| `Workload` | namespace | orchestrator | A running application component, with revisions and bindings |
| `WorkloadRevision` | namespace | `Workload` | An immutable snapshot of one released version |

`NodeType` is separate from `Blueprint` rather than inlined into it because a
node type is reusable: a dozen blueprints all want the same PostgreSQL, and a
catalogue that copies its definition into each of them diverges by the third
one.

No object is generated per blueprint. A blueprint is data, so the catalogue does
not multiply cluster-scoped CRDs and a tenant is granted one resource type
rather than one per catalogue entry (`prior-art.md` §3).

### 2. Two controllers, and a rule for when a third is allowed

- **`blueprint-controller`** reconciles `BlueprintInstance`: resolves the
  blueprint, builds the graph, applies nodes in topological order, keeps them
  consistent, and aggregates status.
- **`workload-controller`** reconciles `Workload`: computes revisions, resolves
  bindings and dependencies, drives the release, and reports application-level
  status.

Everything else a node may create is somebody else's controller: Flux for
HelmRelease, kpack for builds, Argo for imperative work, the Cozystack operators
for managed applications.

> **A type earns its own controller only if it needs at least one of: version
> history, reach into another cluster, or reaction to external state. Otherwise
> it is declarative.**

Applied to the current catalogue this yields exactly the two controllers above.
`Workload` qualifies on all three counts — it keeps revisions, it deploys into a
tenant cluster, and it must react to a rebuilt image and to a rotated
credential. A build does not qualify: kpack is already a controller, and the
platform only creates an `Image` and reads `status.latestImage`.

This rule exists because the failure mode of this architecture is well known:
Korifi, which has the same shape, is still pre-1.0 and covers a subset of its
target API. If the controller count reaches four, the rule was not being
applied.

### 3. The graph is derived from references, never declared

A `Blueprint` node references another node's status through CEL. Ordering is the
topological sort of those references. Order is never written down separately,
because two sources of truth for a graph always drift — kro's rule, adopted
verbatim.

```yaml
apiVersion: cozyap.io/v1alpha1
kind: Blueprint
metadata:
  name: java-service
spec:
  version: 2.1.0

  parameters:
    type: object
    properties:
      repoUrl:       { type: string }
      ref:           { type: string, default: main }
      dbSize:        { type: string, enum: [small, medium, large], default: small }
      needsBucket:   { type: boolean, default: false }
      domain:        { type: string }
      targetCluster: { type: string }
    required: [repoUrl, domain, targetCluster]

  nodes:
    - id: db
      type: cozystack-postgres
      params:
        size: instance.spec.dbSize

    - id: media
      type: cozystack-bucket
      includeWhen: instance.spec.needsBucket

    - id: build
      type: kpack-build
      params:
        repoUrl: instance.spec.repoUrl
        ref:     instance.spec.ref

    - id: api
      type: workload
      params:
        image:  build.status.image           # edge: api depends on build
        target: instance.spec.targetCluster
        domain: instance.spec.domain
        ports:  [ { name: http, port: 8080 } ]
      bindings:
        db:    db.status.binding             # edge: api depends on db
        media: media.status.binding          # skipped when the node is excluded

  actions:
    - { id: scaffold, type: repo-scaffold }
    - { id: migrate,  type: db-migration, bindings: { db: db.status.binding } }
```

**CEL is the expression language, and its evaluation context is restricted.**
The language itself is fixed, standard, non-Turing-complete, type-checked at
apply time and not ours to grow — which is precisely why it is preferred to a
homegrown syntax that acquires one feature per request. The discipline is
applied to the context instead:

- an expression may read `instance.spec.*` and `<nodeId>.status.*`;
- it may not read arbitrary fields of arbitrary cluster objects;
- a node type backed by a third-party resource declares which of that
  resource's fields become `status` on the node (§4), so blueprints never
  couple to an implementation detail.

`includeWhen` takes a boolean. A node excluded by it, and every reference to it,
is skipped.

### 4. Node types declare what they create and what they publish

A node type is data in a package. Three extractors — `fromField`, `fromSecret`,
`literal` — cover every case in the current catalogue: a digest from a kpack
`Image`, an address from a `Service`, a password from a `Secret`.

```yaml
apiVersion: cozyap.io/v1alpha1
kind: NodeType
metadata:
  name: kpack-build
spec:
  parameters:
    type: object
    properties:
      repoUrl: { type: string }
      ref:     { type: string, default: main }

  creates:
    apiVersion: kpack.io/v1alpha2
    kind: Image
    name: "{{ .instance }}-{{ .node }}"
    spec:
      tag: "{{ .platform.registry }}/{{ .ns }}/{{ .instance }}-{{ .node }}"
      source:
        git: { url: "{{ .params.repoUrl }}", revision: "{{ .params.ref }}" }

  ready:
    condition: Ready

  status:
    image: { fromField: { path: "status.latestImage" } }
```

Two syntaxes appear, and the difference is semantic rather than cosmetic:
`instance.spec.x` and `node.status.y` are CEL in a blueprint and **create graph
edges**; `{{ .params.x }}` is local templating inside a node type and creates
none.

A node type that creates a Cozystack managed application creates a `HelmRelease`
with the three `apps.cozystack.io/application.*` labels and the conventional
name prefix. `cozystack-api` projects any such HelmRelease as its typed resource,
so the tenant still sees `Postgres/payments-db` in the Cozystack dashboard
without the platform ever writing through the aggregated API.

### 5. `Workload` owns revisions; Flux performs the apply

`Workload` is the one node type with a controller.

```yaml
apiVersion: cozyap.io/v1alpha1
kind: Workload
metadata:
  name: payments-api
  namespace: tenant-acme
spec:
  target: { cluster: prod, namespace: payments }
  image: harbor.example.io/tenant-acme/payments-api@sha256:abc...
  replicas: 2
  ports: [ { name: http, port: 8080 } ]

  endpoints:
    - { name: http, port: http, visibility: app }

  bindings:
    - name: db
      from: { kind: Postgres, name: payments-db }
      project:
        files: { path: /bindings/db }
        env:   { prefix: DATABASE_ }

  dependencies:
    - { app: billing, endpoint: http, env: { address: BILLING_URL } }

  revisionHistoryLimit: 10

status:
  phase: Ready
  revision: { current: 41, previous: 40 }
  replicas: { desired: 2, ready: 2 }
  url: https://payments.example.com
  bindings:
    - { name: db, resolved: true, secretChecksum: "sha256:9f2c..." }
```

**Revisions are separate immutable objects, not a list in status.** A
`WorkloadRevision` records the resolved image digest, source commit, values and
bindings of one release. Status is derived state and may be rebuilt; an old
image digest exists nowhere else. The pattern is `Deployment` → `ReplicaSet`,
including `revisionHistoryLimit`, so nothing new has to be learned and
`spec.rollbackTo` behaves the way `kubectl rollout undo` already does. This
closes requirement gap G1 (`prior-art.md` §2).

**The controller computes; Flux applies.** `workload-controller` resolves
bindings, selects the revision and builds the desired state, then emits a
`HelmRelease` carrying `kubeConfig` at the target cluster and reads it back.
Cross-cluster apply, connection caching, drift and retry stay with Flux, which
already does this for every Cozystack tenant cluster addon. The expensive and
uninteresting half of a remote-apply controller is therefore not written, and
replacing Flux with a direct apply later changes one component of the controller
and nothing else.

`secretChecksum` is carried into a pod annotation. A rotated credential changes
the checksum, which restarts the workload — otherwise an application keeps the
old password until its next deployment.

### 6. Binding is data, and its wire format is `servicebinding.io`

How to reach a managed service is a per-type fact, and a new managed service
must not require a platform release. It is therefore a package object:

```yaml
apiVersion: cozyap.io/v1alpha1
kind: BindingProfile
metadata:
  name: cozystack-postgres
spec:
  for: { group: apps.cozystack.io, kind: Postgres }
  type: postgresql
  fields:
    host:
      fromField:
        kind: Service
        name: "postgres-{{ .name }}-external-write"
        path: "status.loadBalancer.ingress[0].ip"
    port:     { literal: "5432" }
    username: { literal: "app" }
    password: { fromSecret: { name: "postgres-{{ .name }}-credentials", key: "app" } }
```

A node type whose created resource matches a `BindingProfile` publishes
`status.binding` automatically. That is where `db.status.binding` in the §3
blueprint comes from: the node type declares nothing about connections, the
profile supplies them, and swapping the profile — a different PostgreSQL, an
external one — changes no blueprint.

The projected result follows the Service Binding for Kubernetes specification:
files under `$SERVICE_BINDING_ROOT/<name>/`, with a mandatory `type` entry. Two
things follow. Paketo buildpacks and Spring Cloud Bindings consume that layout
natively, so for a large share of applications requirement R3 needs no
application change at all. And the contract is someone else's, already
specified, rather than a port type system of our own.

The specification's reference *implementation* is not adopted — it projects into
workloads in its own cluster, and the data plane stays clean. The format is
adopted; the controller is not.

## Consequences

### Positive

- One general mechanism, two controllers, and a rule that says when a third is
  permitted. The previous designs each had five mechanisms and no such rule.
- Every piece is either someone else's proven component or a small object of our
  own; nothing is a new general-purpose engine.
- Rollback, credential rotation and developer-facing status — the three things
  requirement 5.14 depends on most — are owned outright rather than inferred.
- A new node type, a new managed service or a new blueprint is a package, with
  no platform release and no code.

### Negative

- `BlueprintInstance` and `Workload` are ordinary CRDs, so `cozystack-api` does
  not project them and the Cozystack dashboard does not show them without work.
  The platform's surface is therefore a **plugin inside `cozystack-ui`**, which
  requires a plugin mechanism to be contributed upstream first (ADR-0024).
- Tenant RBAC for these types is ours to define. The controllers themselves are
  cluster-wide, like every other shared Cozystack controller; tenant permissions
  are namespace-scoped on the namespaced types (ADR-0024 §3).
- Reconciliation, finalizers, drift and retry for two types are ours. The
  unpleasant case — a finalizer on a `Workload` whose target cluster is gone —
  must be designed before it is met.
- The orchestrator owns its nodes and reconciles them continuously, so hand
  edits to a node are reverted. The instance is the source of truth; a node is
  not independently editable. This is a deliberate reversal of the earlier
  "scaffold once" position and the price of consistency.

### Neutral, but worth stating

Composition is fixed by the blueprint. A user varies parameters; they do not add
a node to a running application. Free composition, if it is ever wanted, is a
separate authoring surface over the same objects — not a property of this model.

## Open questions

1. **Blueprint version migration.** An instance is pinned to a blueprint
   version. What a bump does to running instances is undecided, and the simple
   answer — nothing, new version applies to new instances only — has not been
   tested against requirement R6.
2. **Finalizer semantics against an unreachable target cluster.** Blocking
   forever is wrong; deleting the record and orphaning the workload is also
   wrong.
3. **Multi-component applications.** Several `Workload` nodes in one blueprint
   work. Two blueprints referencing each other do not, by construction — see
   ADR-0023 for the one narrow exception.
4. **Consumption accounting (R7 / gap G2).** Nothing in this ADR addresses it.
   Where usage metadata attaches — node type, instance, or revision — should be
   settled before the first package ships, because it is the field that cannot
   be retrofitted into distributed packages.

## Relationship to prior art

ADR-0001 through ADR-0017 no longer exist as files; they were removed with the
retired series and are retrievable from git history. Their durable conclusions
are in `docs/prior-art.md`.

- **ADR-0009** — typed ports reduce to `BindingProfile` plus the
  `servicebinding.io` secret shape.
- **ADR-0010 / 0011 / 0012** — the execution-layer question is void; there is no
  bespoke execution layer. See ADR-0022.
- **ADR-0017** — dynamic fan-out and higher-order nodes stay out of scope; a
  node is always exactly one.
- **ADR-0019** — its central argument, that the core needs an own controller for
  status and revisions, is upheld and narrowed to `Workload`. Its adoption of
  Crossplane is dropped (`prior-art.md` §9).
- **ADR-0020** — the content model survives as the package contents in
  ADR-0022; the invented bundle format does not.
- **`design/brick-model.md`** — the vocabulary survives, the five implementation
  kinds do not.
