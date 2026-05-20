# Cozyap — Architecture Notes

Brain-dump from a long design conversation. Captures the **reactive
reconcile** model we landed on, how atoms become Argo WorkflowTemplates,
and what falls out for UI. Ready to copy into `cozystack/cozyap-`.

---

## 1. Core abstraction — Atom

An **Atom** is a community-writable, versioned `WorkflowTemplate` (or a
small DSL that compiles to one) which describes:

- typed **input** and **output** ports (k8s reference kinds),
- a configurable **parameter** schema,
- a single **reconcile DAG** — steps that bring the world to the desired
  state and emit outputs.

```yaml
apiVersion: cozyapps.community/v1
kind: AtomTemplate
metadata: { name: postgres, version: 1.2.0 }
spec:
  inputs: []
  outputs:
    credentials: { type: secret-ref }
    service:     { type: service-ref }
  params:
    - { key: version,  type: enum, options: ["14","15","16"] }
    - { key: replicas, type: number, default: 1 }
    - { key: storage,  type: enum, options: ["10 GB", "50 GB", "100 GB"] }
  reconcile:
    steps:
      - get-current-state          # kubectl get cluster -o yaml
      - if deletionTimestamp: drain → final-backup → kubectl-delete
      - if currentVersion != desired:
          - pg-basebackup-to-s3    # side effect
          - bump-cluster-version
          - wait-rolling-restart
      - if not exists:
          - apply-cnpg-manifest
          - wait-ready
      - export-outputs:
          credentials: secret/{{cluster}}-app
          service:     service/{{cluster}}-rw
```

Authors contribute atoms as YAML PRs, no Go required. Atoms are
versioned (semver), cluster-scoped, read-only for tenants.

---

## 2. Application = Graph of Atoms

A user (or template author in builder UI) wires atoms together:

```
UserInput.host ─────────────┐
                            ├─→ Ingress
Postgres.credentials ──┐    │
Redis.credentials ─────┼─→ Container ─→ Service ──┘
                       │           │
                       └─→ envFrom │
                                   └─→ workload
TLSCert.secret ─────────────────┐
                                ├─→ Ingress
                                ▼
                          Ingress.url ─→ Application Output
```

- Nodes are atom invocations with parameter values.
- Edges are **k8s resource references** — `secret-ref`, `service-ref`,
  `configmap-ref`, `pvc-ref`, `serviceaccount-ref`, `tls-secret-ref`,
  `workload-ref`, plus semantic literals `image-ref`, `ingress-host`,
  scalars `string`/`number`/`boolean`/`any`.
- Subtyping: `tls-secret-ref → secret-ref` (TLS Secret is a Secret),
  `image-ref → string`, `ingress-host → string`. One-way only —
  siblings (`image-ref ↔ ingress-host`) do not interchange.

This graph is the **declarative state** of the application. Publish
saves it as the application's `ApplicationTemplate` (versioned).

---

## 3. The Reactive Reconcile Model

This is the heart of the design.

**Atoms are functions:**

```
atom(params, externalState) → outputs
```

- **params**: values from `Application.spec.values` (and `UserInput`
  fields, which themselves are atom outputs).
- **externalState**: what the atom polls — container registry tags,
  Git refs, certificate expiry, Vault paths, current k8s objects.
- **outputs**: materialised k8s objects (Secret/ConfigMap) with a
  predictable name, holding the values downstream atoms reference.

**Reconcile is the recomputation**: a single Argo Workflow polls
external state, compares to current outputs, applies the diff to the
cluster, writes new outputs.

**Change propagation:** when an atom's outputs differ from the previous
reconcile, downstream atoms (subscribed via Argo Events watching the
output Secret/ConfigMap) get their own reconcile triggered. Cascade
through the graph in topological order.

**Why this is good:**

| Use case                         | Falls out for free                                    |
|----------------------------------|-------------------------------------------------------|
| Auto-update image                | `Container.reconcile` polls registry → output bump   |
| Auto-rotate TLS                  | `TLSCert.reconcile` sees expiry → re-issue → cascade |
| GitOps                           | `GitSource.reconcile` sees new commit → build → out  |
| Postgres minor patch             | Atom self-upgrades, credentials may rotate → cascade |
| External secret pickup           | Watch Vault → materialise → cascade                  |
| Drift heal                       | Periodic reconcile sees actual ≠ outputs → fix       |
| User edited spec                 | Same reconcile, picks up new params                  |

All of these are the **same mechanism**. There is no separate
`install` / `upgrade` / `remove` lifecycle — only **reconcile**, with
conditional `when:` branches inside the workflow handling each case
(including `deletionTimestamp` for removal).

---

## 4. One Workflow, Not Four

**Decision:** one `reconcile` workflow type per atom, NOT four
(install/upgrade/remove/action).

Reasoning:

- Argo Workflows is an imperative DAG runner, not designed for
  long-lived state machines. Workflows should be short, finish, hit
  the archive.
- `install` / `upgrade` / `remove` are all the same operation when
  expressed as "make actual match desired" — only the inputs differ
  (no current state / current ≠ desired / deletionTimestamp set).
- Drift healing is **free** with a periodic reconcile. With separate
  install/upgrade workflows, you'd need a fifth periodic re-apply
  workflow on top.
- One DAG per atom keeps the contract clean. Author writes one place,
  not three.

**Triggers:**

- **Cron** — safety net (every 4–24h) for drift heal even when nothing
  in the cluster changed.
- **Argo Events** — sensor watches Application.spec changes and the
  output Secrets/ConfigMaps of upstream atoms. Triggers immediate
  reconcile when these change.

**Concurrency:** a lease per Application prevents two reconciles from
running at the same time.

**Removal:** detected via `Application.deletionTimestamp` inside the
reconcile workflow. A `when:` branch runs cleanup steps; finalisers on
the CR hold the object until reconcile exits cleanly.

**Actions** (backup, restart, restore) are still a small separate type
— short Argo Workflows invoked on demand via an `Action` resource.
They do not interfere with reconcile.

---

## 4a. Status Workflow — Separate Concern

Each atom carries **two** workflow templates, not one:

- **`reconcile`** — drives the cluster towards desired state, writes
  outputs. Cron + event-triggered. Has side effects.
- **`status`** — read-only probe that reports observed health as a
  list of typed conditions. Cron-only, runs more often than reconcile
  (e.g. every 30–60 s). Idempotent by construction.

```yaml
spec:
  outputs:
    credentials: { type: secret-ref }
    service:     { type: service-ref }
  reconcile:
    steps: [...]              # apply desired
  status:
    publishes:                # the conditions this atom guarantees
      - ClusterReady
      - BackupOk
      - StorageHealthy
    steps:
      - check-cluster-phase
      - check-last-backup-age
      - check-pvc-utilisation
      - aggregate              # → Application.status.atoms[atom].conditions
```

The status workflow publishes a **list of conditions** (mirrors the
k8s Conditions API) to `Application.status.atoms[<atomName>]`:

```yaml
status:
  atoms:
    postgres:
      status: healthy           # aggregate, worst-condition-wins
      lastChecked: 2024-01-15T10:30:00Z
      conditions:
        - type: ClusterReady
          state: ok
          reason: ClusterHealthy
          message: "all instances streaming WAL"
          lastTransitionAt: ...
        - type: BackupOk
          state: warn
          reason: BackupStale
          message: "no backup in last 36h"
          lastTransitionAt: ...
        - type: StorageHealthy
          state: ok
          reason: VolumesBound
          message: "all volumes bound, 64% used"
          lastTransitionAt: ...
```

Each **condition** carries `type` (machine name, declared by the
atom in `status.publishes`), `state` (`ok` / `warn` / `error` /
`info` / `unknown`), `reason` (CamelCase machine label) and a short
human `message`.

**Aggregate atom status = worst-state of its conditions** (with `info`
mapping to `reconciling`). No condition states → `pending`.

**Aggregate Application sync state** = worst across all atoms.
That's what `SyncStateBadge` reflects: `In sync` / `Drift detected` /
`Reconciling` / `Reconcile failed`.

**Why this matters:**

- **Decoupling**: status keeps reporting even when reconcile is idle.
  Drift detected by a status condition triggers a reconcile.
- **Visibility**: UI surfaces named conditions, not free-form
  messages. Stable identifiers, scriptable, alertable.
- **Composability**: a downstream atom's status workflow can read an
  upstream condition by name (`if postgres.ClusterReady != ok then …`).
- **Cost**: short and read-only, cheap to run frequently.

**Conditions contract:** atom-author declares the condition `type`s
they publish in `spec.status.publishes`. Stable, public API of the
atom. Adding a new condition type — minor version bump. Removing or
renaming — major.

---

## 4b. The Atom Is a Black Box

We deliberately do **not** expose the internal Kubernetes objects
(`Pod`, `ReplicaSet`, `Endpoints`, intermediate Secrets, …) in the
user-facing UI. The atom's contract is:

1. Its **outputs** (typed ports — `secret-ref`, `service-ref`, …).
2. Its **published conditions** (named, state-tagged, with messages).
3. Its **reconcile DAG** (visible to template authors, not tenants).

Whether the `Postgres` atom renders a CNPG `Cluster`, a Zalando
`postgresql`, a hand-rolled StatefulSet, or wraps an external RDS is
**none of the tenant's business**. What matters is:

- "Did the atom emit `credentials` (secret-ref) and `service`
  (service-ref)?"
- "Is `ClusterReady` ok?"
- "How is `BackupOk` doing?"

If the tenant needs lower-level debugging they drop to kubectl with
their kubeconfig. The platform UI stays at the **atom level**, which
is the only level where the contract is stable across atom versions
and managed-service backend swaps.

This also keeps the model **portable**: an atom written against CNPG
today and reimplemented against Crunchy Postgres tomorrow exposes
the same outputs and conditions. Tenant-facing graphs don't break.

---

## 5. Side Effects in a Declarative World

Declarative state alone can't express:

- DB migrations before image bumps
- Backup before Postgres version change
- Connection draining before pod replacement
- Cache warm-up after deploy

These live as **conditional steps inside the reconcile DAG** of the
atom that owns them. They are imperative, but they are scoped to the
atom and to specific transitions:

```yaml
- if currentVersion != desired:
    - pg-basebackup-to-s3      # side effect
    - bump-cluster-version
    - wait-rolling-restart
```

No global `preUpgrade` hooks. The atom that needs the side effect
declares it inline. Side effects are **not idempotent** by nature —
they execute only on the transition that requires them.

---

## 6. Composite Workflow Generation

For an `Application` with graph `G`:

1. Controller renders `G` from `Application.spec.values` + linked
   `ApplicationTemplate` version.
2. Controller compiles `G` into a composite Argo Workflow:
   - Topologically sort the atoms.
   - For each atom, emit a DAG task invoking the atom's reconcile
     `WorkflowTemplate` (`templateRef`).
   - Wire outputs: `tasks.postgres.outputs.parameters.credentials` →
     `tasks.container.arguments.parameters.envFromSecret`.
   - Add platform-provided steps automatically: observability
     wiring, Vault token mounting, lease acquisition.
3. Submit the composite workflow.

The composite workflow runs every reconcile. Each atom-task can short-
circuit (`exit 0`) on no-op, so the cost of a clean reconcile is just
"get + compare" per atom.

---

## 7. Idempotency Is a Hard Requirement

Every atom's reconcile DAG **must** be safe to re-run with no input
change. Enforce via:

- **Catalog linter** — automated tests at PR time, running the atom
  twice in a row, expecting the second run to be a no-op.
- **Server-side apply** for any `kubectl apply` step — guarantees
  convergence.
- **Skip-if-equal** before destructive ops — compute desired manifest,
  compare to current; if equal, exit before applying.

Without this, periodic reconcile rapidly burns the cluster.

---

## 8. Rate Limiting and Cycles

**Cycle detection** at builder time: the UI / linter rejects graphs
where `A.out → B.in` and `B.out → A.in` (any directed cycle). Reactive
storm protection.

**Rate limiting** at controller time:
- Minimum interval between reconciles of the same Application (e.g.
  30s) — debounces rapid event bursts.
- Maximum concurrent reconciles per cluster (e.g. 50) — prevents
  cascade storms.
- Backoff on consecutive failures — exponential, capped.

---

## 9. Pinning and Auto-Tracking

For atoms that observe external state (registry, git, certs), the user
decides per output whether it auto-tracks or pins:

```yaml
spec:
  values:
    container:
      image:
        track: "wordpress:6.4"          # auto-bump within 6.4.x
        pin:   "wordpress:6.4.1"         # explicit, no auto-bump
```

UI surfaces this as a lock/unlock icon on each external-source output.

---

## 10. K8s Primitives vs Managed Services

Atoms split into two categories:

**K8s Primitives** — one atom ↔ one k8s object. Container (Deployment),
Service, Secret, ConfigMap, PVC, ServiceAccount, Ingress, TLS Cert
(Certificate). Used when the author wants explicit control over the
manifest.

**Managed Services** — one atom hides multiple k8s objects (often via
an operator's CR). Postgres, Redis, S3/MinIO, Helm Chart. They expose
the same primitive outputs (`secret-ref` + `service-ref`) so the
downstream graph treats them identically — whether the Postgres is a
CNPG `Cluster`, a Zalando `postgresql`, or a hand-rolled StatefulSet
is an implementation detail.

This means the user's `Container.envFromSecret` is the **same edge**
regardless of where the Secret came from. Composable.

---

## 11. User Input — the Form Atom

The `UserInput` atom is special: it has no static outputs. Its outputs
are **dynamic**, derived from a list of fields the template author
configures on the node (`label`, `type`, `required`, `default`, etc).

Each field becomes:

- one **output port** on the node (typed by field type — string, number,
  boolean, image-ref, ingress-host, etc.),
- one **entry in the launch form** for the tenant.

This is the only point of contact between the template builder and
the deploy form. Single source of truth.

---

## 12. Constant Atom

Companion to `UserInput`: a literal baked into the template, not
exposed to the tenant. Type (string / number / boolean / image / host)
and value editable in the inspector. Useful for hardcoding versions,
ports, fixed hostnames.

In the from-input-handle drag suggestion popup, `Constant` is always
offered for scalar target types preconfigured with the right
`valueType`.

---

## 13. Param Exposure as Input Ports

Any atom parameter can be `exposed` from the inspector — it becomes a
typed input port on the left side of the node, and its inline editor
turns into a "from upstream" placeholder until a connection is wired.
Unbind drops the connection.

Pattern: Houdini / Unreal Blueprint.

---

## 14. UI Affordances We Want

**Builder canvas:**

- Drag from palette to spawn atom.
- Click an empty handle drag-release in pane → suggestion popup with
  compatible atoms (bidirectional — from output or from input).
- `Constant` always in the from-input list for scalar targets.
- Cycle detection inline.
- Multi-port handles (Container.envFromSecret accepts N secret-refs).

**Reactive Flow Preview (replaces Run Preview):**

- "Simulate" presets: Registry pushed image / TLS expiring / Git
  commit / Drift detected.
- Animation: source atom lights first, propagates downstream via
  edges, each downstream atom marked `changed` or `noop`.
- Acts as the live demo of the reactive model.

**Atom detail page:**

- **What I observe** — list of external sources (registry path, git
  ref pattern, expiry watchers) with last-poll timestamp.
- **What observes me** — downstream atoms in the current graph
  subscribed to this atom's outputs.
- **Reconcile DAG** — show the workflow steps with `when:` branches
  visually annotated.
- **Versions** — semver list, changelog between versions, graph diff.

**Application detail page:**

- **Sync state badge** at the top: `In sync` / `Drift detected` /
  `Reconciling` / `Reconcile failed`.
- **Topology** section — list of atoms in the application graph,
  each with status pill and expandable list of published conditions
  (Conditions-style rows: state icon + type + reason + message).
  **K8s objects under each atom are not exposed** — atom is a
  black box. See §4b.
- **Changes** timeline (user-facing name for reconcile runs). Each
  entry: trigger (cron / spec / event), result chip (changed / noop /
  failed), outputs diff, duration.
- **Pin status** per parameter — lock / auto-track icons.
- Generic metrics (uptime, CPU/RAM/storage, restarts) and logs viewer.

---

## 15. What This Replaces in Original cozyap-RAW

The original spec described separate `install` / `upgrade` / `remove`
hooks plus periodic `suggestUpgrade`. This model collapses all of
them into a single reconcile workflow per atom with conditional
branches. `suggestUpgrade` becomes implicit — the atom polls registry
or git inside reconcile, outputs the new ref, and downstream cascade
handles the rest.

`Action` resources are kept as a separate concept for true one-shot
operations (manual backup, restart). They invoke their own short Argo
workflow and don't interact with reconcile.

---

## 16. Open Questions

- **Output materialisation contract**: where exactly does an atom
  write its outputs? Convention `Secret/<app>-<atom>-out` with known
  keys? Or `Application.status.outputs.<atom>.<port>`? Latter is
  cleaner but coarser for change detection.
- **Cross-Application atoms**: can `App-B.Container.envFromSecret`
  reference `App-A.Postgres.credentials`? Cross-app graph edges open
  up sharing but break tenant isolation. Probably no in v1.
- **GitOps source of truth**: where does `Application.spec` live —
  in-cluster CR (current assumption) or in Git (ArgoCD reconciles
  CR from Git)? Both work; pick one for v1.
- **Atom-author SDK**: the YAML DSL needs companion tooling — local
  `cozyap atom test ./postgres` that runs reconcile twice and checks
  idempotence, plus a graph-level smoke test runner.
- **UI editor for the reconcile DAG itself**: out of scope for v1.
  Atom-authors write YAML; UI consumes it read-only.

---

## 17. Glossary

- **Atom** — versioned WorkflowTemplate with typed ports and a
  reconcile DAG. Cluster-scoped, read-only for tenants.
- **AtomTemplate** — the YAML definition of an atom in the registry.
- **ApplicationTemplate** — versioned graph of atoms, plus form
  fields from User Input. Cluster-scoped, read-only for tenants.
- **Application** — instance of a template, namespace-scoped, owned
  by a tenant. Holds `spec.values` for the template's parameters.
- **Reconcile** — single Argo Workflow executing the application's
  composite graph DAG. The only lifecycle operation.
- **Action** — separate one-shot Argo Workflow for user-initiated
  operations (backup, restart, restore).
- **Output materialisation** — atoms write reconcile results to a
  known k8s object so downstream atoms can watch + react.
- **Port type** — typed contract between atoms, modelled after k8s
  reference kinds (secret-ref, service-ref, …).
