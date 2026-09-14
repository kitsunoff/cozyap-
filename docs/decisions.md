# Decision Register

## Purpose

One line per question, with a status. An ADR answers *why A and not B*; this
file answers *is this question still open at all*.

A new question is a new row, not a reason to revisit the model. Keep it current;
it is cheaper than rediscovering that something was already settled.

Statuses: **closed** (decided, see the reference), **open** (needs a decision
before the work it blocks), **deferred** (deliberately out of scope for now),
**verify** (assumed, must be confirmed against a live cluster).

## Closed

| # | Question | Answer | Where |
| --- | --- | --- | --- |
| C1 | What is the resource model | Six objects, three shipped as package data | ADR-0021 §1 |
| C2 | How many controllers do we write | Two, plus a rule for when a third is allowed | ADR-0021 §2 |
| C3 | How is the graph ordered | Topological sort of CEL references; order is never declared | ADR-0021 §3 |
| C4 | Expression language | CEL, with a restricted evaluation context | ADR-0021 §3 |
| C5 | Where do blueprints live | Data in a package, not a generated CRD per blueprint | ADR-0021 §1 |
| C6 | How is rollback done | `WorkloadRevision`, mirroring Deployment and ReplicaSet | ADR-0021 §5 |
| C7 | Who performs the cross-cluster apply | Flux, via a `HelmRelease` carrying `kubeConfig` | ADR-0021 §5 |
| C8 | How are connection parameters delivered | `BindingProfile` plus the `servicebinding.io` file layout | ADR-0021 §6 |
| C9 | What happens on credential rotation | Secret checksum in a pod annotation restarts the workload | ADR-0021 §5 |
| C10 | Do we adopt the Service Binding reference implementation | No. The format, not the controller | ADR-0021 §6 |
| C11 | Convergence engine | Flux and the operators Cozystack already ships; no engine of ours | ADR-0022 §2 |
| C12 | Imperative work | Argo Workflows; no `Task` primitive is built | ADR-0022 §3 |
| C13 | How is a one-shot action triggered once per desired state | Workflow name derived from a hash of resolved parameters | ADR-0022 §3 |
| C14 | Crossplane | Not adopted; it would install an engine to call Helm | `prior-art.md` §9 |
| C15 | Upjet and generated providers | Not the default answer; an integration is a workflow template | ADR-0022 §3 |
| C16 | Distribution | Cozystack `Package` / `PackageSource`; only the content model is ours | ADR-0022 §5 |
| C17 | Where does platform machinery run | Host cluster; data planes run workloads only | ADR-0022 §6 |
| C18 | Is an application a pivot | No — a boundary that owns a namespace with default-deny | ADR-0023 §1 |
| C19 | Where are service-to-service connections declared | By producer and consumer, never centrally | ADR-0023 §2 |
| C20 | Are cross-instance references allowed | Only endpoints, with consent, within one tenant, one-directional | ADR-0023 §4 |
| C21 | Can a user hand-edit a node of a running application | No. The instance is the source of truth and nodes are reconciled | ADR-0021, consequences |
| C22 | Do we need our own UI | Yes. Our CRDs are not projected by `cozystack-api` | ADR-0021, consequences |
| C23 | Does the platform write through the aggregated API | No. It writes HelmReleases with the Cozystack application labels | ADR-0021 §4 |

## Open

| # | Question | Blocks | Note |
| --- | --- | --- | --- |
| O1 | kpack or a `Containerfile` build first | the build node type | kpack satisfies R1 and is heavier; a Containerfile build is the escape hatch either way |
| O2 | Argo Events, or polling from `workload-controller` | release triggering | Events is the least certain of the added components |
| O3 | Our own UI: patch `cozystack-ui`, a separate panel, or none in the first iteration | weeks of scope | a first iteration with no UI still proves the architecture |
| O4 | Tenant RBAC model for our CRDs | anything a tenant touches | ClusterRoles aggregated into Cozystack's tenant roles |
| O5 | Finalizer semantics when a target cluster is unreachable | `workload-controller` | blocking forever is wrong, orphaning is also wrong |
| O6 | What a blueprint version bump does to running instances | catalogue upgrades, R6 | the simple answer is "nothing"; untested against the requirement |
| O7 | Where consumption metadata attaches | R7, gap G2 | node type, instance or revision; cannot be retrofitted into shipped packages |
| O8 | May applications of different tenants call each other | policy and catalogue scope | assumed no |
| O9 | Egress allowlist to third parties | default-deny makes it necessary | node type, workload or instance |
| O10 | Argo archive retention and its cost | operations | tasks are how everything imperative happens |

## Deferred

| # | Question | Why |
| --- | --- | --- |
| D1 | Graph editor | the model derives topology from references, so it stays possible; no requirement asks for it |
| D2 | Dynamic fan-out, higher-order nodes | ADR-0017; a node is always exactly one |
| D3 | Module Federation UI plugins | revisit when declarative views provably fall short |
| D4 | mTLS inside the data plane | would mean a mesh in a plane that must stay clean |
| D5 | Adopting resources the platform did not create | membership is ownership now; needs a deliberate read-only mechanism |
| D6 | Fleet view across instances | needs an aggregation layer that is not designed |
| D7 | Durable execution tier | both the original arguments for and against it are void; re-derive if ever needed |

## Verify against a live cluster

Assumptions the design rests on that have not been observed. Each can invalidate
part of the model.

| # | Assumption | If false |
| --- | --- | --- |
| V1 | A pod in a tenant Kubernetes cluster can reach a managed application through its `external: true` LoadBalancer service | the managed-dependency model does not work as designed; R2 and R3 need another path |
| V2 | The platform may create a `HelmRelease` with `kubeConfig` pointing at a tenant cluster's admin kubeconfig secret | cross-cluster deployment needs a different mechanism |
| V3 | A HelmRelease created by us, carrying the three `apps.cozystack.io/application.*` labels and the conventional name prefix, is projected by `cozystack-api` as its typed resource | managed dependencies lose their Cozystack dashboard and RBAC surface |
| V4 | `WorkloadMonitor`-derived conditions are absent for workloads in a tenant cluster, so readiness must come from our own controller | status design changes |
| V5 | Cilium network policies in a tenant Kubernetes cluster can express the `cluster` visibility level | visibility collapses to `app` and `public` |

V1 is the one to run first: it can invalidate the dependency model outright.

## Changing this file

Add a row when a question appears. Move it, do not delete it, when it is
answered — a closed question with a pointer is worth more than a missing row.
When an answer changes, the ADR changes and the row follows it.
