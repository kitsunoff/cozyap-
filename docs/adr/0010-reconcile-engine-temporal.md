# ADR-0010: Reconcile Engine — Option A: Temporal

## Status

Proposed alternative. Parallel to ADR-0011 (Option B: Pods with stdin/stdout).
Comparison and decision in ADR-0012.

Follows ADR-0008 (Reactive Reconcile Model) and ADR-0009 (Atom Contract).
This ADR adds the Temporal-specific fields to `AtomTemplate.spec.executor` and
defines the SDK contract, worker lifecycle, and transition tracking for the
Temporal execution layer.

## Context

ADR-0008 established that atoms need a durable execution engine: reconcile must
survive worker restarts, side-effect operations (DB migrations, basebackups)
must not re-execute after partial completion, and long-running operations
(schema upgrades that take 20–40 min) need reliable execution without the
operator polling a Job status.

Temporal is a durable execution platform built around workflow-as-code. A
workflow function executes arbitrary logic; Temporal records each activity
completion in its history so that a worker restart replays from the last
checkpoint rather than from the beginning. This makes it a natural fit for
the reconcile model.

## Decision

### Deployment topology

```
Host cluster (cozy-system)
├── temporal-server          # shared Temporal cluster (single instance for all tenants)
│   └── backed by Cozystack-managed PostgreSQL
│
└── cozyap-operator          # the platform controller
    └── platform-worker      # Temporal worker registered on platform task queue

Tenant control-plane cluster (per tenant)
├── cozyap-operator-agent    # watches AtomInstances, enqueues Temporal runs
│
├── atom-worker-postgres     # Deployment — one per AtomTemplate referenced by tenant
├── atom-worker-redis        # Deployment — one per AtomTemplate referenced by tenant
└── atom-worker-…
```

Key constraints:

- **One shared Temporal cluster** on the host side. Tenant isolation is via
  Temporal namespaces (one per tenant).
- **Atom workers run in the tenant control plane**, not in the dataplane
  (user workload) cluster. They hold kubeconfig credentials for their
  target dataplane; granting dataplane access is the platform's job (ADR-0015).
- The platform worker (host side) handles catalog operations and managed-service
  ordering that require host cluster access. It does not touch atom business
  logic.

### Temporal namespace per tenant

Each tenant maps to one Temporal namespace named `cozyap-{tenantID}`. Benefits:

- Workflow history is isolated per tenant — a tenant cannot enumerate another
  tenant's workflows.
- Namespace-scoped retention and rate-limit policies.
- Worker registration is scoped: atom workers poll only their tenant namespace.

The cozyap-operator creates the Temporal namespace as part of tenant
provisioning. On tenant deletion, the namespace (and its full workflow history)
is archived then purged after the retention window.

### AtomTemplate executor extension for Temporal

Option A extends the neutral `executor` shape from ADR-0009 with
Temporal-specific fields:

```yaml
spec:
  executor:
    image: ghcr.io/cozystack/cozyap-atoms/postgres:1.2.0
    resources:
      requests: { cpu: 100m, memory: 128Mi }
      limits:   { cpu: 500m, memory: 512Mi }
    requires:
      dataplaneAccess: true
      hostAccess:      false
      vault:           false

  reconcile:
    workflowType: PostgresReconcile    # Temporal workflow type registered by worker
    timeout: 30m
    cron: 4h

  status:
    workflowType: PostgresStatus
    cron: 60s
    publishes: [ClusterReady, BackupOk, StorageHealthy]

  actions:
    - name: backup
      displayName: "Create backup"
      workflowType: PostgresActionBackup
      params: [...]
```

`workflowType` is the Temporal workflow type name that the atom worker image
registers with the Temporal server at startup. The operator starts workflow
runs by type name on the tenant's task queue.

### Workflow ID scheme and serialisation

The operator derives a stable workflow ID for every run:

| Run category | Workflow ID |
|---|---|
| reconcile | `{namespace}/{atomInstanceName}/reconcile` |
| status | `{namespace}/{atomInstanceName}/status` |
| action | `{namespace}/{atomInstanceName}/action/{actionCRName}` |

`WorkflowIDReusePolicy` for reconcile runs is set to
`WORKFLOW_ID_REUSE_POLICY_TERMINATE_IF_RUNNING`. If a reconcile trigger fires
while the previous reconcile is still running (e.g. cron overlaps a long
schema migration), Temporal terminates the in-flight workflow and starts a
fresh one. The atom's reconcile logic is idempotent (ADR-0009 §"What an atom
MUST guarantee"), so termination and restart is safe.

Status workflows use `WORKFLOW_ID_REUSE_POLICY_ALLOW_DUPLICATE` — they are
read-only probes and can run concurrently with reconcile without risk.

### SDK contract

Atom authors import the Cozyap Go SDK (or a future Python/Java SDK). The SDK
wraps the Temporal client and exposes a typed context:

```go
// ReconcileContext is passed to every reconcile workflow function.
type ReconcileContext struct {
    // AtomInstance identity
    Namespace        string
    AtomInstanceName string

    // Resolved inputs (k8s refs in tenant namespace)
    Inputs map[string]cozyap.Ref

    // Parameter values (type-checked against AtomTemplate.spec.params)
    Params map[string]any

    // Environment access
    Dataplane cozyap.KubeClient  // kubeconfig-backed client for the target Environment
    Host      cozyap.KubeClient  // host cluster client (nil if requires.hostAccess = false)
    Vault     cozyap.VaultClient // nil if requires.vault = false

    // Transition tracking (see "Transition tracking" below)
    Transitions cozyap.TransitionStore
}

// ReconcileResult is returned from the workflow function.
// The operator reads this and patches AtomInstance.status.
type ReconcileResult struct {
    Outputs    map[string]cozyap.OutputRef
    Conditions []cozyap.Condition
}
```

The SDK registers the workflow types declared in `AtomTemplate` with the
Temporal client. The operator delivers the `ReconcileContext` as the workflow
input payload (JSON-serialised). The workflow function returns a
`ReconcileResult`; the operator reads it via Temporal's
`DescribeWorkflowExecution` result and patches the CR.

Atom workers do **not** write to `AtomInstance.status` directly. They do not
hold RBAC to the `cozyapps.cozystack.io` API group.

### Transition tracking

Long-running side-effect operations — major-version database upgrades,
basebackups before version change, certificate rotations — must not re-execute
if the workflow is terminated and restarted. The SDK exposes a
`TransitionStore` backed by Temporal activity IDs:

```go
// Attempt starts a named transition. If already committed, returns (result, nil)
// from the stored activity result without re-executing fn.
result, err := ctx.Transitions.Attempt("migrate-pg14-to-pg16", func() (any, error) {
    return runMigration(ctx)
})
```

Each named transition maps to a Temporal activity with a deterministic ID
derived from `{atomInstanceName}/{transitionName}`. Temporal's activity
deduplication means the function body executes exactly once per transition
name, even across workflow restarts.

Transition names are scoped to the `AtomInstance`. The atom is responsible
for choosing transition names that are unique within a single reconcile goal
state (e.g. include the target version in the name:
`"migrate-pg14-to-pg16-storage50Gi"`).

### Atom worker lifecycle

The operator manages a `Deployment` per `(tenant, AtomTemplate)` pair:

1. When a tenant's `Application` first references an `AtomTemplate`, the
   operator creates an atom worker Deployment in the tenant control plane,
   using `executor.image` and `executor.resources` from the template.
2. The worker registers its workflow types with the Temporal server using the
   tenant's Temporal namespace and task queue
   `cozyap-atoms-{tenantID}-{atomTemplateName}`.
3. When no `AtomInstance` references the `AtomTemplate` (all applications
   uninstalled), the operator scales the Deployment to 0 replicas. A KEDA
   `ScaledObject` triggers scale-back-up when new tasks appear on the task
   queue.
4. When a new patch version of `AtomTemplate` is published, the operator
   performs a rolling update of the Deployment. In-flight workflow runs
   complete on the old version; new runs pick up the new worker image.
5. On tenant deletion, the Deployment is removed after the Temporal namespace
   has been archived.

### Platform worker

A single `platform-worker` Deployment runs in `cozy-system` on the host
cluster. It registers on the `cozyap-platform` task queue and handles:

- Cozystack managed-service ordering (e.g. "provision a CNPG cluster via
  the Cozystack HelmRelease API").
- Platform wiring steps: injecting observability sidecars, Vault policies,
  identity-linked ServiceAccounts into the tenant namespace.
- Catalog operations: publishing new `AtomTemplate` versions to the CRD store.

Atom workers delegate host-side operations to the platform worker via a typed
activity stub. The platform worker does not execute atom business logic.

### Reconcile trigger flow

```
Trigger source                 cozyap-operator              Temporal
─────────────────────────────────────────────────────────────────────
cron tick (4h)            →  enqueue reconcile  →  StartWorkflow(
spec changed              →  enqueue reconcile       workflowType=PostgresReconcile,
upstream fingerprint diff →  enqueue reconcile       id=ns/name/reconcile,
                                                      input=ReconcileContext,
                                                      taskQueue=cozyap-atoms-…)
                                                  ↓
                                          workflow runs activities
                                          on atom worker in tenant CP
                                                  ↓
                                          returns ReconcileResult
                                                  ↓
                                     operator patches AtomInstance.status
```

### Operator handles Temporal outage

If the Temporal cluster is unavailable:

- No new reconcile workflows can be started. Pending enqueue requests are
  retried with exponential back-off by the operator.
- In-flight workflows are suspended at their current activity boundary;
  Temporal replays from the last committed activity when the server recovers.
- Status probes stop updating. The `Reconciled` condition transitions to
  `Unknown` after two missed cron intervals.
- Cron reconciles accumulate; on recovery, the operator starts the
  outstanding reconciles immediately (at most one per `AtomInstance`).

## Consequences

### Positive

- Durability is built-in: workflow restarts replay from the last committed
  activity, no custom checkpoint logic needed in atoms.
- Transition tracking (`Transitions.Attempt`) gives a clean, auditable
  primitive for risky side-effect operations without the atom author
  managing it manually.
- Temporal UI gives full workflow history for debugging: every activity
  input/output, retry, and timeout is recorded.
- Scale-to-zero is clean: workers are Deployments managed by the operator;
  KEDA scales on queue depth without the operator polling.
- Tenant isolation is strong: separate Temporal namespaces, separate task
  queues, atom workers per tenant.

### Negative

- Temporal cluster is an additional stateful dependency: it needs HA
  deployment, a PostgreSQL backend, and operational runbooks.
- Atom authors must learn Temporal concepts (activities, workflow replay,
  determinism constraints) in addition to the Cozyap SDK.
- A Temporal cluster outage affects all tenants simultaneously — wide blast
  radius compared to a per-Pod model.
- KEDA is an additional dependency for scale-to-zero; without it, idle atom
  workers consume resources continuously.

### Risks

- **Temporal replay determinism**: if an atom workflow function is
  non-deterministic (e.g. reads current time, uses random IDs), Temporal
  replays will fail with `NonDeterministicError`. The catalog linter must
  detect common violations (unchecked `time.Now()`, `rand`, direct HTTP
  calls outside activities).
- **Worker version skew**: if the atom worker image is updated while
  in-flight workflows still run against the old history, Temporal may
  reject the new workflow code as non-deterministic. Patch versions must not
  change workflow code structure — only activity implementations. The version
  gate in `AtomTemplate` (major bump for breaking changes) mirrors this.
- **Task queue naming collision**: if two tenants happen to share the same
  `tenantID` (e.g. during a re-provisioning cycle), their task queues
  collide. The operator must enforce globally unique `tenantID` generation.
