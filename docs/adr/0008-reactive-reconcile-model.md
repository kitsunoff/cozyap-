# ADR-0008: Move to Reactive Reconcile Model

## Status

Proposed. Supersedes ADR-0001 through ADR-0007 (also Proposed, kept for
historical context).

This is a meta-ADR. It records the architectural shift that motivates the
new ADR series; detailed contracts live in the follow-up ADRs. Execution
layer (how atoms actually run — Temporal workflows vs. Pods with
stdin/stdout) is still **open** and tracked separately in ADR-0010
(Option A: Temporal), ADR-0011 (Option B: Pods), and ADR-0012 (comparison
and decision).

## Context

The original design (ADR-0001 to ADR-0007, RAW.md) modelled Cozyap as a
package-manager-on-Argo. Each `ApplicationTemplate` shipped four imperative
hook DAGs — `install`, `upgrade`, `remove`, and a periodic `suggestUpgrade` —
plus user-triggered `Action` workflows. The controller picked which hook to
run based on the CR's lifecycle state.

Three pressures forced a redesign before any code shipped:

1. **Drift heal was not a first-class concern.** Periodic re-apply needed a
   fifth workflow type on top of the four hooks, with its own state machine.
   `suggestUpgrade` (poll registry/git) was a sixth, and the cascade after a
   bump (rebuild image → restart deployment → invalidate cache) had no
   defined mechanism.

2. **Monolithic templates blocked backend swap.** An `ApplicationTemplate`
   for "Wordpress" baked in MySQL, Container, Ingress, and TLS as one
   indivisible Helm-like blob. Replacing CNPG with Zalando, or MinIO with
   external S3, required rewriting the template. Cozystack's whole pitch is
   composable managed services — the template model didn't reflect that.

3. **Hook semantics drifted across ADRs.** Step spec was redefined in three
   places (ADR-0001 args/env, ADR-0004 `script:`/`stdin:`, ADR-0005
   `secretRefs`); `cluster-admin` SA per workflow gave any template author
   unscoped power inside the Environment; idempotency was a wish, not a
   contract.

The reactive reconcile model captured in `docs/cozya-new-notes.md` resolves
all three by collapsing the lifecycle to a single operation and decomposing
the application into composable atoms.

## Decision

### The shift in one sentence

`install` / `upgrade` / `remove` / `suggestUpgrade` / drift heal are all the
same operation — **make actual match desired** — and become a single
`reconcile` workflow per atom. The application is a graph of atoms wired by
typed ports, not a monolithic template.

### Core concepts (full contracts in ADR-0009–0014)

- **Atom** — versioned, community-writable building block. Declares typed
  input ports, typed output ports (k8s reference kinds: `secret-ref`,
  `service-ref`, `configmap-ref`, …), a parameter schema, a `reconcile` DAG
  (drives state), and a `status` DAG (read-only health probe publishing named
  conditions). Atoms are black boxes: their internal k8s objects are not
  surfaced to tenants.

- **AtomTemplate** — YAML definition of an atom in the catalog.
  Cluster-scoped, read-only for tenants, versioned with semver.

- **ApplicationTemplate** — versioned graph of atoms plus form fields
  contributed by a `UserInput` atom. The template author wires atoms in a
  builder UI or YAML; the result is portable across clusters.

- **Application** — namespace-scoped CR in a tenant namespace. Holds
  `spec.values` for the template's parameters and references an
  `ApplicationTemplate` version.

- **Reconcile** — the only lifecycle operation. Triggered by:
  - cron (drift safety net, every 4–24 h),
  - Argo Events sensors watching `Application.spec` and the materialised
    outputs of upstream atoms (cascade).

  Removal is detected by `deletionTimestamp` inside the reconcile DAG; a
  finalizer holds the CR until reconcile exits cleanly.

- **Status** — separate, read-only Argo Workflow per atom. Runs more often
  than reconcile (30–60 s cron), publishes a list of typed conditions
  (`type`, `state`, `reason`, `message`) to
  `Application.status.atoms[<name>].conditions`. Aggregated worst-state wins.

- **Action** — short, one-shot Argo Workflow for user-initiated operations
  that are not desired-state (manual backup, restart, restore). Does not
  interfere with reconcile. Carried over from ADR-0001 with the same retention
  rules.

### Execution layer: Argo Workflows is out, two options remain

The original design (ADR-0004) used Argo Workflows as the DAG engine inside
every Environment. The reactive redesign drops Argo entirely:

- **Argo Workflows is an imperative DAG runner.** Workflows are designed to
  start, run, finish, archive. We need a continuous reconciler that runs
  many times per object lifetime. Argo Workflow CRs on every cron tick
  pollute etcd and waste controller cycles.
- **Argo's placement question** (host vs. each Environment) has no good
  answer for our load profile.
- **The reactive model wants a real SDK contract**, not a YAML DSL. Both
  remaining options give that.

Two execution layers remain on the table for the reactive model:

- **Option A — Temporal** (ADR-0010). Atoms are Temporal workflows +
  activities packaged in worker images. A shared Temporal cluster on the
  host side; atom workers run in the tenant control plane; reconcile and
  status are workflow types invoked by the Cozyap operator. Native
  durable execution for long-running side effects.
- **Option B — Pods with stdin/stdout** (ADR-0011). Atoms are container
  images with a `cozyap reconcile` entrypoint that reads JSON on stdin,
  performs the reconcile, and emits JSON on stdout. The Cozyap operator
  invokes them as Pods (or Jobs) on each reconcile trigger. Durability is
  DIY via state persisted in `AtomInstance.status`.

ADR-0012 compares both options head-to-head and records the decision.
Whichever wins, the **atom contract** (ADR-0009) — typed ports,
fingerprinted output materialisation, conditions schema, versioning —
stays the same; only the executor differs.

Cascade detection stays in the operator (k8s is the source of truth) in
both options.

### What the new ADR series covers

| ADR | Scope | Status |
|-----|-------|--------|
| 0008 Reactive Reconcile Model | This document. Meta-ADR for the shift away from install/upgrade/remove hooks. | Proposed |
| 0009 Atom Contract | Port types, output materialisation, conditions schema, atom-version compatibility. Executor-neutral. | Proposed |
| 0010 Reconcile Engine (Option A: Temporal) | Operator↔Temporal scheduling, atom worker lifecycle, transition tracking, SDK contract | Proposed alternative |
| 0011 Reconcile Engine (Option B: Pods stdin/stdout) | Operator→Pod invocation, JSON I/O contract, state persistence, transition tracking, SDK contract | Proposed alternative |
| 0012 Execution Layer Decision | Side-by-side comparison of 0010 vs 0011 with trade-off analysis and a recorded choice | Proposed (decision pending) |
| 0013 Application Graph | Composition rules, cycle detection, cross-Application references, `UserInput` / `Constant` semantics | Proposed |
| 0014 Status and Conditions API | Status workflow contract, condition publishing, aggregation rules | Proposed |
| 0015 Platform Integration | Execution placement, platform wiring injection (observability, Vault, identity), tenant SA scope, dataplane kubeconfig flow. Refines based on 0012 outcome. | Pending 0012 |
| 0016 Catalog and Versioning | `AtomTemplate` + `ApplicationTemplate` storage, semver policy, sync from git | Proposed |

### What survives from the old ADRs

- **Environment abstraction** (ADR-0002) — name, kubeconfig wiring, multi-env
  per tenant. Revisited in ADR-0013 for controller-set and Argo placement.
- **Networking primitives** (ADR-0007) — Ingress, LoadBalancer, cert-manager,
  External-DNS. Now expressed as atoms in the graph (`Ingress` atom consumes
  `service-ref` + `tls-secret-ref`).
- **Cozystack multi-tenancy** (ADR-0002) — tenant = namespace, RBAC and
  quotas come from Cozystack. No re-implementation at the Cozyap layer.
- **Action as one-shot** (ADR-0001) — kept as-is, sits outside the reconcile
  loop.
- **User personas and flows** (ADR-0005) — end-user, operator, developer
  flows largely unchanged; template-author flow shifts from "write three
  hooks" to "compose atoms (or write one atom)".

### What is dropped

- `install` / `upgrade` / `remove` hooks as distinct workflow types.
- `suggestUpgrade` as a separate hook. Polling external state (registry,
  git, certs) happens inside an atom's `reconcile` and propagates via
  output cascade.
- Per-application monolithic templates. Anything bigger than one k8s object
  is composed from atoms.
- Aggregated API server with PostgreSQL backend (ADR-0006) — replaced by
  versioned CRD storage; revisited in ADR-0014.
- `cluster-admin` workflow ServiceAccount as the default. Scoping is revisited
  in ADR-0013.
- **Argo Workflows entirely** (ADR-0004). Replaced by Temporal as the
  durable execution engine.

## Consequences

### Positive

- One mental model — reconcile — covers install, upgrade, remove, drift,
  external-source updates, and cascades.
- Atom contract (typed outputs + named conditions) makes backend swap free:
  the same `Container` consuming a `secret-ref` works whether the upstream
  Postgres atom wraps CNPG, Zalando, or external RDS.
- Drift heal is the default behaviour, not an extra feature.
- Status is decoupled from reconcile: health stays observable while reconcile
  is idle, drift in a status condition triggers a reconcile.
- The catalog becomes a graph of small reusable pieces — closer to Cozystack's
  composability story than monolithic templates.

### Negative

- Authoring barrier shifts. End-user templates can be wired in a UI, but
  writing a new atom is more demanding than the old "three hooks" model:
  idempotence is mandatory, conditions are a public API, ports are typed.
- Two workflow templates per atom (reconcile + status) multiplied by the
  number of atoms in a graph increases Argo Workflow object count. Mitigated
  by short-circuit reconciles (no-op exit) but real load needs measurement.
- The model has more concepts to teach: atoms, ports, conditions, cascade,
  transitions. Documentation and the builder UI carry that weight.

### Risks

- The composite-vs-independent execution choice (ADR-0010) is load-bearing.
  Picking the wrong default will either flood Argo with workflows or produce
  partial-failure pathologies.
- Side-effect transitions (DB migration before image bump, basebackup before
  major-version upgrade) need a transition-id mechanism that does not yet
  exist in Argo Workflows. ADR-0010 must define it.
- Cross-Application atom references are unresolved (ADR-0011). Shared managed
  Postgres is a real Cozystack scenario; if v1 ships without it, customers
  hit the limit early.
- Atom-version compatibility in running applications is unresolved
  (ADR-0009 / ADR-0014). A wrong default (auto-bump everywhere) breaks
  tenants on every catalog release; another wrong default (pin everything)
  freezes the platform.

These risks block implementation, not the architectural direction. They are
addressed in the follow-up ADRs.
