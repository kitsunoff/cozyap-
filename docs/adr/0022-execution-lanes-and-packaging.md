# ADR-0022: Execution Lanes and Packaging

## Status

Proposed, 2026-09-14. Depends on ADR-0021. Resolves the execution-layer
question left open by ADR-0010, ADR-0011 and ADR-0012 by removing it.

## Context

Every previous design tried to build one execution mechanism that covered both
continuous convergence and one-shot side effects. ADR-0004 used Argo Workflows
for both and could not express drift healing. ADR-0008 replaced them with a
reactive reconcile and then needed three ADRs to choose a runtime for it, which
were never concluded.

The two kinds of work are genuinely different and the tools for each already
exist. Nothing in requirement 5.14 asks the platform to provide either of them.

## Decision

### 1. Two lanes, with a discriminator that can be applied at review

> **If the desired state is a set of objects in a cluster, it is lane A.
> If the work is a sequence of calls to something outside, it is lane B.**

The discriminator is deliberately not "declarative versus imperative" or "simple
versus complex". Those blur within a quarter; this one does not.

| | Lane A — convergence | Lane B — action |
| --- | --- | --- |
| Runs on | Flux, plus the operator behind each resource | Argo Workflows |
| Semantics | continuous; self-healing; drift is corrected | one-shot per version of the request |
| Examples | managed applications, the workload, ingress, certificates | repository scaffolding, CI generation, database migration, documentation build, manual backup, credential rotation |
| Idempotency | free | the author's responsibility, always |
| Cost | no pods when nothing changes | a pod per step |

### 2. Lane A is Flux and what Cozystack already ships

The platform installs no convergence engine. Requirement 5.14's dependency set
is already a set of Cozystack managed applications, and a composition engine
placed in front of them would exist only to call Helm, which Flux already calls
(`prior-art.md` §9).

| Requirement | Mechanism | Origin |
| --- | --- | --- |
| R2 — database, cache, queue, object storage | `postgres`, `mariadb`, `mongodb`, `clickhouse`, `redis`, `kafka`, `rabbitmq`, `nats`, `opensearch`, `qdrant`, `bucket`, `seaweedfs` | Cozystack |
| R3 — connection parameters delivered to the application | `workload-controller` plus `BindingProfile` (ADR-0021 §6) | ours |
| R4 — published address | `ingress-nginx` addon, `cert-manager`, `external-dns` | Cozystack |
| R4 — rollback | `WorkloadRevision` (ADR-0021 §5) | ours |
| R5 — observability and secret storage | `monitoring`, `openbao` | Cozystack |
| R6 — versioned catalogue | `Package` / `PackageSource` (§5) | Cozystack |
| G3 — registry with retention | `harbor` | Cozystack |
| R1 — build from source without build scripts | kpack (§4) | added |
| R7 — consumption accounting | none | open |

Deployment into a tenant Kubernetes cluster uses a `HelmRelease` carrying
`kubeConfig` pointing at that cluster's admin kubeconfig secret. This is the
mechanism Cozystack already uses to install `ingress-nginx`, `cert-manager`,
`csi` and the monitoring agents into every tenant cluster; the platform reuses
it rather than writing a cross-cluster applier.

### 3. Lane B is Argo Workflows, and it is not a new primitive

`brick-model.md` §9 specified a `Task` primitive with ordered containers,
per-step retries and timeouts, TTL cleanup, a log surface, an idempotency key
and a scheduled variant. That is a description of Argo Workflows. It is adopted
instead of being reimplemented (`prior-art.md` §5).

A node type in lane B renders a `Workflow` whose **name derives from a hash of
the resolved node parameters**:

```yaml
kind: Workflow
metadata:
  name: "scaffold-{{ .instance }}-{{ .node }}-{{ .paramsHash }}"
spec:
  serviceAccountName: task-repo-scaffold
  workflowTemplateRef: { name: provision-gitlab-repo }
```

Parameters unchanged, the object is unchanged and re-applying it does nothing.
Parameters changed, the name changes and the work runs once for the new desired
state. No controller is required to obtain "run once per version of the
request".

Three properties are not optional, and the first two are how ADR-0004 went
wrong:

1. **A minimal ServiceAccount per `WorkflowTemplate`**, shipped alongside it in
   the same package. ADR-0004 gave workflows `cluster-admin`.
2. **No `ttlStrategy` on hash-named workflows.** Garbage collection removes the
   object, Flux recreates it, and the work runs a second time. History belongs
   in the Argo archive, not in live objects.
3. **Workflows run in the platform plane, never in a tenant's data plane.**
   Constraint V6, and also the difference between a task and a foothold.

A scheduled action is the same node type with a `schedule`, rendering a
`CronWorkflow`. Following Cozystack's own idiom for one-shot operations —
`BackupJob`, `RestoreJob`, `Plan` — an action is a named node, not a side
channel in the API.

Lane B also replaces a large part of ADR-0019's external-integration strategy.
An integration that a single customer requests is a `WorkflowTemplate` calling
that system's API or CLI: hours of work, shipped in a package, with no generated
provider to maintain indefinitely and no licence review of the Terraform
provider it would have been generated from.

### 4. Three components are added to the platform plane

Cozystack ships none of these, so each is a `Package` the platform owns and
maintains.

| Component | Why | Alternative considered |
| --- | --- | --- |
| kpack | R1's "without writing build scripts" means Cloud Native Buildpacks | Kaniko or BuildKit is simpler but does not satisfy R1; it remains the escape hatch for a repository that already has a `Containerfile` |
| Argo Workflows | lane B | plain Jobs cover a single step and reimplement Argo badly beyond that |
| Argo Events | triggers a release when a build produces a new digest | polling from `workload-controller`; acceptable, and the fallback if Argo Events proves fiddly |

The platform's own two controllers (ADR-0021 §2) are a fourth package.

### 5. Distribution is Cozystack `Package`; the content model is ours

ADR-0020 concluded that a bundle must be an OCI artefact of our own design. The
content-model half of that conclusion stands — a Crossplane Configuration
carries only XRDs and Compositions, and everything else a package must carry has
no place in it (`prior-art.md` §8). The distribution half is withdrawn:
Cozystack already ships `Package` / `PackageSource` with git and OCI sources,
install-time `variants` and a marketplace surface, and constraint V2 names it as
the vehicle.

A package may contain:

| Content | Consumed by |
| --- | --- |
| `NodeType`, `Blueprint`, `BindingProfile` | `blueprint-controller` |
| Helm charts backing node types | Flux |
| `WorkflowTemplate` and its ServiceAccount and RBAC | Argo |
| Operators a node type requires in a target cluster | the target cluster |
| Dashboards and alert rules | the observability stack |
| RBAC presets, policies, quotas | the platform |
| Consumption-accounting metadata | the metering pipeline, once it exists |

Two constraints on the content model. It carries no Cozystack assumptions, so a
non-Cozystack installer can consume the same artefact (constraint V7). And no
package contains platform code: a package is charts, templates and data.

`PackageSource.spec.variants` — several implementations of one package selected
at install time — is the same idea as a per-target implementation and should be
used rather than duplicated. Note that the field is `variants`, not `flavors`
(`prior-art.md`, corrections).

### 6. Where things run

| Plane | Contents |
| --- | --- |
| Cozystack host cluster | both platform controllers, kpack, Argo, the registry, every managed application, every `HelmRelease` |
| Tenant Kubernetes cluster | the customer's workloads, and the addons Cozystack already installs there |

A data plane runs workloads and nothing else by default (constraint V6). Where a
node type genuinely needs machinery inside the target cluster, it declares it and
the platform installs it on demand — and reference-counts it, because two
applications sharing an operator must not break when one is deleted.

## Consequences

### Positive

- The execution-layer question that consumed ADR-0010 through ADR-0012 is
  answered by not having an execution layer.
- Both lanes are mature components with their own operators, UIs and
  communities. Neither is ours to debug at the engine level.
- "A customer wants an integration with X" costs a package, not a release, and
  not a generated provider maintained forever.
- The reuse table is the strongest argument for the whole direction: net new
  code is two controllers, a UI and a set of charts.

### Negative

- Three components in the host cluster that Cozystack does not ship, each with
  its own upgrade path, CVEs and operational runbook.
- Lane B does not self-heal. A workflow that created a repository will not
  notice that the repository was deleted, because the parameter hash has not
  changed. Options, in descending honesty: accept it; a periodic `CronWorkflow`
  that verifies, bounded by the pod-churn budget in `prior-art.md` §6; or let the
  UI surface the discrepancy and offer re-running the action. The default is to
  accept it and to be explicit that lane B is not converging.
- Argo Events is the least certain of the three additions.

## Open questions

1. Whether kpack or a `Containerfile` build is the default for the first
   release, given kpack is the requirement-satisfying but heavier option.
2. Whether Argo Events is worth its operational cost against polling from
   `workload-controller`.
3. Argo archive retention, and who pays for it — tasks are how everything
   imperative happens, so log volume is a real cost.
4. Whether `--watch-configs-label-selector` should be enabled on Cozystack's
   helm-controller. Without it, values sourced from a ConfigMap or Secret
   propagate on the reconcile interval rather than on change. It is an
   installation-wide override of a shared component, not a fork.

## Relationship to prior art

- **ADR-0004** — Argo returns, for one-shot work only, and without
  `cluster-admin`.
- **ADR-0008** — its rejection of Argo applied to the reconcile role and remains
  correct there.
- **ADR-0010 / 0011 / 0012** — void; there is no bespoke execution layer to
  choose a runtime for.
- **ADR-0019 §5-6** — the three-tier integration strategy and Upjet are replaced
  by lane B for the long tail. Generating a provider stays available and is no
  longer the default answer.
- **ADR-0020** — content model kept, distribution mechanism replaced by
  Cozystack `Package`.
