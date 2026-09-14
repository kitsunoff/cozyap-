# Prior Art — What the First ADR Series Established

## Status

Written 2026-09-14, when the ADR-0001…0020 series was retired in favour of the
model recorded in ADR-0021…0023.

This document exists for one reason: a handful of conclusions from the retired
series are expensive to rediscover, and without them recorded the same
arguments will be held again in three months. Everything else about that series
is in git history and does not need to be carried forward.

## The three designs that came before

| Series | Model | Why it was abandoned |
| --- | --- | --- |
| ADR-0001…0007 | `ApplicationTemplate` with `install` / `upgrade` / `remove` / `suggestUpgrade` hooks, executed as Argo Workflows | Drift healing needed a fifth workflow type; monolithic templates blocked backend swap; hook semantics were redefined in three separate ADRs |
| ADR-0008…0017 | Atoms: `AtomTemplate` / `AtomInstance`, typed ports, a single reactive `reconcile` per atom, plus a purpose-built execution layer | The execution layer was never decided (three ADRs on Temporal vs Pods); the model was more general than any requirement asked for |
| ADR-0018…0020 + `design/brick-model.md` | Bricks: uniform contract, five implementation kinds, Crossplane for dependencies, own OCI bundle format | Five implementation kinds meant five mechanisms to learn; Crossplane reduced to `provider-helm` for the actual dependency set; the bundle format duplicated Cozystack `Package` |

## Conclusions that survive

These are the load-bearing findings. Each is followed by why it still matters.

### 1. Status quality is half the product

Requirement 5.14 specifies a user who does not know Kubernetes. Such a user must
be shown "the build failed, here is the log" and "the database is starting" —
never a chain of four objects to `kubectl describe`.

*Consequence:* any layer that cannot write a purpose-designed status is
disqualified from owning the application object. This was ADR-0019's argument
for an own controller and it is the reason ADR-0021 keeps `Workload` as a
controller rather than a chart.

### 2. Revisions cannot be retrofitted

Requirement R4 asks for rollback. A purely declarative "desired equals actual"
model has no concept of history, so rollback is not derivable from it. The
snapshot of resolved inputs — image digest, commit, values, bindings — has to
exist from the first release or the history is simply absent for everything
deployed before the feature lands.

*Consequence:* `WorkloadRevision` is in ADR-0021 from day one, not on a roadmap.

### 3. An XRD creates a CRD, and CRDs are cluster-scoped

Crossplane v2 makes composite and managed resources namespaced, but definitions
stay cluster-wide. The same is true of any design that generates a CRD per
blueprint. In a single installation this means the catalogue, the provider set
and provider *versions* are common to all tenants; per-tenant modularity
degrades to an RBAC-scoped subset of one catalogue.

*Consequence:* this must never be sold as per-customer modularity, and it is why
ADR-0021 uses one `BlueprintInstance` CRD with blueprints as data rather than a
generated CRD per blueprint.

### 4. Temporal was rejected, and the reason has changed

The argument for Temporal was durable execution; the argument against it was
that one outage suspends every tenant. Once the platform became per-instance
rather than a shared plane, a Temporal cluster per tenant stopped being
economically defensible and the shared-blast-radius argument disappeared with
it. Both arguments are now void, and neither should be reused as-is if a durable
execution tier is ever reconsidered.

### 5. Argo Workflows was rejected for the wrong role

ADR-0008 removed Argo with three arguments: a workflow is one-shot while
reconcile is continuous; workflow placement had no good answer; the model wanted
an SDK rather than a YAML DSL. All three concerned Argo as the *reconcile*
engine. None of them applies to Argo as a *task* engine, and in that role it was
never evaluated. `brick-model.md` §9 then specified a `Task` primitive whose
feature list — ordered containers, per-step retries and timeouts, TTL cleanup,
log surface, scheduled variant — is Argo Workflows restated.

*Consequence:* ADR-0022 adopts Argo for imperative work and does not build a
`Task` primitive.

### 6. Status must never be a pod

Measured on a single node: a reconcile pod costs about 3.0 s warm and 7.5 s
cold, with a sustained ceiling near 3.2 pods/s and a safe budget around
1 pod/s per node. Drift checking on a multi-hour interval is free even at a
thousand applications. A per-resource status probe on a 30–60 s cron exceeds a
single node's ceiling at roughly fifty applications.

*Consequence:* status is derived from watches, in-process. Drift intervals are
measured in hours. Full numbers in `research/cold-start-measurements.md`.

### 7. Inline code in a custom resource is remote code execution with a YAML front end

Acceptable, but only with a minimal per-task service account, explicit RBAC on
who may author it, default-deny egress, mandatory limits and timeouts,
dependencies resolved at build time rather than run time, and an idempotency
key. ADR-0004 gave workflows `cluster-admin`; that is the specific mistake not
to repeat.

### 8. A Crossplane Configuration package carries only XRDs and Compositions

UI descriptors, dashboards, RBAC presets, policies and consumption-accounting
metadata have no place in it, and the community workaround of wrapping static
objects in `provider-kubernetes` `Object` resources inverts the architecture.
The content-model conclusion stands even though the distribution conclusion was
withdrawn in favour of Cozystack `Package`.

### 9. The dependency set of requirement 5.14 already exists

Database, cache, queue and object storage are all Cozystack managed
applications with their own operators. For that set, a composition engine
reduces to `provider-kubernetes` and `provider-helm` — it would install an
engine in order to call Helm, which is already called. Crossplane's value lies
entirely outside Cozystack's catalogue, and requirement 5.14 does not reach
there.

### 10. Not everything attached to an application is a node

From OAM: an autoscaler, a backup schedule, a network policy, a sidecar, a pod
disruption budget, a resource limit — these modify one resource and produce
nothing another resource consumes. Modelling them as graph nodes triples node
count for no information.

*Rule:* if it has an output another node consumes, it is a node; if it only
modifies one node's behaviour, it is a field on that node.

## Corrections to the retired series

Recorded because these errors are in text that still exists in git history.

- **`flavors` is `variants`.** Cozystack's package model has
  `PackageSource.spec.variants[]` and `Package.spec.variant`. ADR-0020's
  amendment and `brick-model.md` §12 both call it `flavors`.
- **A Cozystack application's secret is not a connection secret.** It carries
  credentials only. `postgres-<n>-credentials` maps usernames to passwords with
  no host, port or database; `redis-<n>-auth` holds a single `password` key;
  `kafka-<n>-*` holds certificates and no credentials at all. The connection
  address has to be composed from the service name, which follows a different
  convention per application type. This is why ADR-0021 makes binding profiles
  explicit data rather than assuming a uniform secret shape.
- **An AGL resource has no writable status.** `ApplicationStatus` in
  `cozystack-api` is four fields — `version`, `conditions`, `namespace`,
  `externalIPsCount` — projected from the backing HelmRelease plus a
  `WorkloadMonitor`. No controller can write to it. Several earlier notes
  assumed otherwise.

## What was researched and should not be researched again

- The IDP landscape, split into portal / application model / orchestrator /
  runtime PaaS, with what to take from each: `research/idp-landscape.md`.
- Crossplane, kro and Kratix as composition substrates:
  `research/crossplane-as-platform-base.md`.
- Pod cold-start and churn budgets: `research/cold-start-measurements.md`.
- How platforms with an existing managed-service API attach an arbitrary-workload
  layer — service brokers, `servicebinding.io`, Korifi, Epinio, OpenShift
  topology, Heroku add-ons: recorded in ADR-0022 and ADR-0023 rather than a
  separate note.

## Retired documents

`docs/adr/0001-*` through `docs/adr/0017-*` were removed. `docs/adr/0018-*`
through `docs/adr/0020-*` and `docs/design/brick-model.md` are superseded by
ADR-0021…0023; their vocabulary survives, their structure does not. All remain
retrievable from git history.
