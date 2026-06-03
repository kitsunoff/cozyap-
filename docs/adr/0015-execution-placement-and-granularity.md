# ADR-0015: Execution Placement and Granularity

## Status

Proposed. Follows ADR-0008 (Reactive Reconcile Model) and ADR-0009 (Atom
Contract). Fills the slot reserved as "Platform Integration / execution
placement" in the ADR-0008 index.

This ADR **assumes the Pods/stdin-stdout execution layer** (ADR-0011) and
records, as motivation, why Crossplane-as-engine and Temporal were rejected
for the runner role. The formal execution-layer choice still belongs to
ADR-0012; this ADR presumes its outcome and should be read together with it.

## Context

Atoms are arbitrary, community- or tenant-authored container images that
reconcile real state (ADR-0009). Two questions were left open:

- **Where** does an atom's reconcile container run?
- **How** are reconcile runs packed into Pods, and at what cost/capability?

Three pressures shape the answer:

1. **A shared control plane cannot safely run arbitrary atom code.** Running
   tenant/community atom containers in the host plane is a privilege-
   escalation and cross-tenant blast-radius problem. Execution must be
   tenant-scoped.

2. **Extensibility-by-installation is rejected.** Crossplane's package/
   function model installs a runtime (Deployment, CRDs, RBAC) into the
   control plane to make an extension usable. In a shared plane that means
   installing tenant code into the host; even per-tenant it is heavyweight
   and admin-level. That model fits a curated, static set of extensions, not
   a large dynamic community catalog where "using an atom" must not require a
   cluster install. Therefore atoms run **on demand as Pods** — using an atom
   is *running an image*, never *installing a package*. (Temporal was also
   rejected for the runner: durable execution at the cost of a stateful HA
   service whose outage suspends all tenants.)

3. **Placement and cost must be explicit and tunable.** Cozystack gives each
   tenant its own Environment (a control plane). Some atoms legitimately need
   host access (ordering a managed service from Cozystack); most act on the
   tenant runtime. Cold-start cost and stateful capability differ per atom.
   None of this should be a hardcoded global; it should be declarative.

## Decision

### 1. Two planes: shared declaration, placed execution

- The **shared host API server** holds the catalog (`AtomTemplate`,
  OCI-distributed) and the desired-state objects (`AtomInstance`). Lightweight
  data only.
- **No tenant code runs in the host plane by default.** Execution is directed
  by *placement* into a tenant-owned execution zone.

This keeps one pane over all `AtomInstance`s (enabling a fleet/governance
view) without standing up a separate API server per tenant.

### 2. Placement is first-class (two axes — do not collapse)

Vocabulary aligns with Open Cluster Management `Placement` / Karmada.

- **`atomRunnerPlacement`** — *where the reconcile container runs*. Hidden
  field, default = the tenant's `system` placement. Overridable to
  `host-system` **only by platform-trusted (signed) atoms** (e.g. an atom
  that orders a managed service from the Cozystack host). Community/tenant
  atoms are forced to tenant-system; host placement is gated by policy so it
  cannot become a loophole to run code in the host.

- **`targetPlacement`** — *where the produced workload/effect lands*. Explicit
  on workload-producing atoms (e.g. `Container`), and **propagated along the
  graph**: `Service` → `Container`, `Ingress` → `Service` inherit the
  placement of what they reference. The builder **validates** that a typed-
  port edge does not silently cross a placement boundary — this is placement's
  analog of cycle detection.

- **Credential bridging.** When runner placement ≠ target placement, the
  platform wires **scoped credentials** from runner → target. This is
  `executor.requires.dataplaneAccess` (ADR-0009) parameterized by
  `targetPlacement`. Three-part contract: *where I run / where I land / what
  creds bridge them*.

### 3. Execution mode is an enum

There is a gradation, and one of the boolean combinations is illegal
(`stateful` requires a warm process). An enum makes the illegal state
unrepresentable.

```yaml
execution:
  mode: ephemeral | warm | stateful   # default: ephemeral
```

| `mode` | Image | Process | Typical | scale-to-zero | cold-rebuild | Barrier |
|--------|-------|---------|---------|---------------|--------------|---------|
| `ephemeral` | pulled per run | fresh Job | container, service, ingress | n/a | n/a | low |
| `warm` | hot | fresh exec per request | git-repo, buildpacks (start latency matters) | yes | n/a | low |
| `stateful` | hot | long-lived, holds state | watch/webhook sources, connection holders | no (or accepts cold-rebuild) | required | high (≈ mini-controller) |

- **`ephemeral`** — a Job per reconcile, cold every time. Default. Pure
  `(params, observed) → (desired, outputs)`. Lowest authoring barrier.
- **`warm`** — a warm pod that `exec`s the atom entrypoint per request with
  stdin/stdout. Amortizes the expensive part (image pull) while keeping the
  one-shot contract and per-reconcile state isolation. Warm ≠ stateful.
- **`stateful`** — a long-lived process that holds state between reconciles.
  Earns its cost mainly for **long-lived watches/webhooks on external state**
  (event-driven instead of cron-poll), plus persistent connection pools and
  in-memory caches.

**Hard rule for `stateful`:** in-memory state is a **cache**; the cluster
remains the source of truth. The pod may die at any time, so a stateful atom
MUST cold-rebuild its state from cluster objects on restart. The catalog
linter requires a kill-restart-converge test **only** for `mode: stateful`.

**Why enum, not two booleans:** `stateful` implies `warm`; two booleans would
permit the impossible "stateful + cold". The enum is also extensible — a
future `durable` tier (for long side-effects that must survive a mid-operation
crash, e.g. basebackup → major-version migration) can be added as a fourth
value rather than a new flag.

### 4. One image per Pod; pooling is same-type, per-tenant

- Atom = image, so a Pod runs **one** image. "One Pod, many `AtomInstance`s"
  means many instances of the **same atom type**, never different atoms
  together (no docker-in-docker).
- Pooling **never crosses tenants** (memory and credential isolation).
  Placement already scopes Pods to the tenant `system` placement, so pooling
  is at most per `(atom type, tenant)`.
- The `ephemeral` / `warm` / `stateful` choice is a per-atom property plus a
  runtime policy (e.g. promote hot atoms to `warm`, scale idle warm pools to
  zero), not a global architectural decision.

## Consequences

### Positive

- Host-vs-tenant execution becomes a **declarative field**, not a hardcoded
  architecture. Atoms that must touch the host (managed-service ordering) and
  atoms that act tenant-side are expressed in the same model.
- The shared declaration plane gives **one pane over all `AtomInstance`s** →
  a fleet/governance view (the governance gap relative to an IDP) without
  per-tenant API servers.
- **No installation**: using an atom is running an image. The community
  catalog stays dynamic; there is no package-manager admin operation per atom.
- Execution cost and capability are **tunable per atom** (`ephemeral` default;
  `warm`/`stateful` opt-in) without changing the contract for simple atoms.

### Negative

- The shared host API plus a host scheduler concentrates scheduling power
  across tenants and must be hardened (see Risks).
- The `stateful` tier lets authors write mini-controllers — the highest
  complexity tier. It must stay rare and expert, not the default.
- Placement propagation and boundary validation are real builder engineering.

### Risks

- **Host placement as a privilege loophole** if trust-gating is weak.
  Mitigation: only platform-signed atoms may request `host-system`; the
  policy is enforced, not advisory.
- **Host scheduler blast radius.** Mitigation: the scheduler is a *scheduler,
  not an actor* — it places a Pod into the tenant's execution zone, and that
  Pod runs with a **tenant-scoped ServiceAccount only**; the host component
  does not act inside the tenant with host credentials.
- **Shared API-server scale** with all tenants' `AtomInstance`s in one etcd,
  and **cross-placement cascade**: output materialisation must be chosen
  (mirror outputs into `AtomInstance.status` in the shared API vs per-placement
  watchers) — this interacts with the open output-materialisation question
  (notes §16).
- **Stateful correctness**: an atom whose correctness depends on surviving
  memory is broken. Mitigation: enforce cold-rebuild via the linter.

## Relationship to other ADRs

- **ADR-0008** — preserves reconcile-only and cluster-as-source-of-truth;
  the stateful cold-rebuild rule defends that invariant.
- **ADR-0009** — `atomRunnerPlacement`, `targetPlacement`, and `execution.mode`
  are new fields on the atom contract; `executor.requires.dataplaneAccess` is
  parameterized by `targetPlacement`.
- **ADR-0011** — assumes Pods/stdin-stdout (a KRM-Functions-shaped contract)
  as the runner.
- **ADR-0012** — presumes the Pods option; records the rejection of
  Crossplane-as-engine (Functions/Packages = control-plane installation,
  incompatible with a dynamic multi-tenant catalog) and Temporal (ops weight,
  shared blast radius) as motivation. The formal decision is recorded there.
- **ADR-0017** — generators and the `CreateApplication` higher-order atom run
  as atoms, so they inherit placement and `execution.mode`.

## Open questions

- **Output materialisation across placements** — `status` mirror in the shared
  API vs per-placement watchers — for cascade detection.
- **Host scheduler hardening** — which component holds which credentials, and
  how the tenant-scoped SA is projected into the execution zone.
- **Defaults** — warm-pool sizing and the stateful idle budget per tenant
  (operator-set, tenant-overridable within limits).
- **A `durable` fourth mode** — needed for long side-effects surviving a
  mid-operation crash, or handled by `transitionId` discipline within
  `stateful`?
