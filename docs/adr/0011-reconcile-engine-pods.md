# ADR-0011: Reconcile Engine — Option B: Pods with stdin/stdout

## Status

Proposed alternative. Parallel to ADR-0010 (Option A: Temporal).
Comparison and decision in ADR-0012.

Follows ADR-0008 (Reactive Reconcile Model) and ADR-0009 (Atom Contract).
This ADR adds the Pods-specific fields to `AtomTemplate.spec.executor` and
defines the JSON I/O contract, durability model, and worker lifecycle for the
Pods execution layer.

## Context

ADR-0010 describes the Temporal option, which provides durable execution at
the cost of a shared stateful dependency. This ADR describes the alternative:
atoms as plain container images invoked as Kubernetes Jobs. The execution
model is simpler — no external workflow server, no SDK beyond stdlib — but
durability becomes the atom author's responsibility.

The core idea: the operator runs one Job per reconcile trigger, passes
context as JSON on stdin, and reads the result from stdout. State that must
survive a Job restart (e.g. "migration already ran") is stored in
`AtomInstance.status` and passed back in the next invocation.

## Decision

### Deployment topology

```
Host cluster (cozy-system)
└── cozyap-operator           # the platform controller

Tenant control-plane cluster (per tenant)
└── cozyap-operator-agent     # watches AtomInstances, creates Jobs for reconcile triggers
    # No persistent atom worker processes — Jobs are ephemeral
```

There is no external workflow server. Every reconcile trigger creates a
Kubernetes Job that runs the atom image, processes JSON from stdin, writes
JSON to stdout, and exits. The operator watches Job completion and patches
`AtomInstance.status` with the result.

### AtomTemplate executor extension for Pods

Option B extends the neutral `executor` shape from ADR-0009 with
Pods-specific fields:

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
    # Optional entrypoint override. Default: image ENTRYPOINT.
    # The operator calls: <command> reconcile
    command: [cozyap-atom]
    timeout: 30m
    cron: 4h

  status:
    command: [cozyap-atom]    # called as: <command> status
    cron: 60s
    publishes: [ClusterReady, BackupOk, StorageHealthy]

  actions:
    - name: backup
      displayName: "Create backup"
      # called as: <command> action backup
      command: [cozyap-atom]
      params: [...]
```

If `command` is omitted, the image's own `ENTRYPOINT`/`CMD` is used with
the subcommand appended (`reconcile`, `status`, `action <name>`).

### JSON payload (stdin)

The operator writes a single JSON object to the Job Pod's stdin:

```json
{
  "run": {
    "type": "reconcile",
    "atomInstance": {
      "name": "my-app-postgres",
      "namespace": "tenant-acme",
      "generation": 7
    }
  },
  "params": {
    "version": "16",
    "replicas": 3,
    "storage": "50Gi"
  },
  "inputs": {
    "backupBucket": {
      "kind": "Secret",
      "name": "my-app-s3-bucket-secret",
      "namespace": "tenant-acme"
    }
  },
  "environment": {
    "dataplaneKubeconfig": "<base64-encoded kubeconfig>",
    "hostKubeconfig": null,
    "vault": null
  },
  "state": {
    "transitionId": "migrate-pg14-to-pg16-done",
    "custom": { }
  }
}
```

Fields:

- `run.type` — `reconcile`, `status`, or `action`.
- `params` — validated against `AtomTemplate.spec.params` before Job creation.
- `inputs` — fully resolved k8s refs from upstream `AtomInstance.status.outputs`.
- `environment.dataplaneKubeconfig` — base64-encoded kubeconfig injected only
  if `requires.dataplaneAccess: true`.
- `state` — the previous `AtomInstance.status.executorState` (see below);
  empty on first run. The atom uses this to skip already-committed transitions.

### JSON result (stdout)

The atom writes a single JSON object to stdout before exiting 0:

```json
{
  "outputs": {
    "credentials": {
      "kind": "Secret",
      "name": "my-app-postgres-credentials",
      "key": "connection-string"
    },
    "service": {
      "kind": "Service",
      "name": "my-app-postgres-rw",
      "port": "postgres"
    }
  },
  "conditions": [
    {
      "type": "ClusterReady",
      "status": "True",
      "state": "ok",
      "reason": "ClusterHealthy",
      "message": "all 3 instances streaming WAL"
    },
    {
      "type": "BackupOk",
      "status": "True",
      "state": "ok",
      "reason": "BackupRecent",
      "message": "last backup 4h ago"
    }
  ],
  "state": {
    "transitionId": "migrate-pg14-to-pg16-done",
    "custom": { }
  }
}
```

Fields:

- `outputs` — k8s refs for each declared output port. Fingerprints are
  computed by the operator after reading the referenced objects.
- `conditions` — conditions to publish; must cover all names in
  `AtomTemplate.spec.status.publishes`.
- `state.transitionId` — the most recent committed transition (see below).
  Passed back on the next invocation so the atom can skip completed work.
- `state.custom` — opaque JSON blob (≤ 4 KiB) for atom-specific persistent
  state. Stored in `AtomInstance.status.executorState.custom`.

Non-zero exit code signals failure. The operator reads stderr for the failure
message and sets the `Reconciled` condition accordingly.

### Durability via transition tracking

Without a workflow engine, durability is the atom's responsibility. The
mechanism is a `transitionId` — a string token the atom writes after
committing a side-effect that must not repeat.

```python
# Example Python atom logic
def reconcile(ctx):
    if ctx.state.transition_id != "migrate-pg14-to-pg16":
        run_migration(ctx)
        # Write result with transition_id set — if we crash after this
        # write succeeds, the next run will see the token and skip.
        return Result(
            ...,
            state=State(transition_id="migrate-pg14-to-pg16")
        )
    # Migration already done — proceed with normal reconcile
    ensure_running(ctx)
    return Result(...)
```

The invariant: **write the result (including the new `transitionId`) only
after the side effect commits.** The operator stores the returned state in
`AtomInstance.status.executorState` and passes it back on the next
invocation.

This is DIY compared to Temporal's activity deduplication, but sufficient for
a disciplined atom author. The catalog linter checks that atoms with major
version changes (e.g. a param change that implies a migration) declare a
corresponding `transitionId` strategy in their documentation.

### AtomInstance status extensions

```yaml
status:
  executorState:
    transitionId: "migrate-pg14-to-pg16-done"
    custom: {}             # opaque, passed back on next invocation

  lastReconcileRun:
    id: my-app-postgres-reconcile-7f3a1b   # Job name
    phase: Succeeded
    startedAt: 2026-05-20T10:30:00Z
    finishedAt: 2026-05-20T10:32:00Z
```

`executorState` is written by the operator from the atom's stdout result. It
is not interpreted by the operator — only passed back to the next run.

### Job lifecycle and serialisation

The operator enforces exactly-one-reconcile-at-a-time via a label on active
Jobs:

- Before creating a reconcile Job, the operator checks for an existing active
  Job with label `cozyapps.cozystack.io/atominstance=<name>` and
  `cozyapps.cozystack.io/run-type=reconcile`.
- If one exists, the new trigger is debounced: the operator sets a pending
  flag on `AtomInstance` and starts a fresh Job as soon as the current one
  completes.
- If no active Job exists, the operator creates one immediately.

Status Jobs (health probes) run independently. They use a different
`run-type=status` label and do not block reconcile Jobs.

Jobs are created with `restartPolicy: Never`. Failed Jobs are not
automatically retried by Kubernetes; the operator retries on the next
reconcile trigger (cron or event). Failure back-off is managed by the
operator's reconcile queue (exponential back-off, configurable per
`AtomTemplate`).

### Resource isolation

Each Job runs with:

- `resources` from `AtomTemplate.spec.executor.resources`.
- `serviceAccountName: cozyap-atom-runner` (namespaced, minimal RBAC — read
  own `AtomInstance`, no write to CRD group).
- `dataplaneKubeconfig` injected as a projected `Secret` volume (not an env
  var); the atom reads it from a well-known path (`/run/cozyap/kubeconfig`).
- `automountServiceAccountToken: false` — the atom does not need the in-pod
  SA token.

### Long-running operations

Jobs have a hard timeout from `AtomTemplate.spec.reconcile.timeout`. For
operations that take longer than the timeout (unusual, but possible for very
large databases), the atom must split the work across multiple invocations
using the `transitionId` mechanism:

```
Run 1: start migration, write transitionId="migration-started"
       Job exits after partial work, timeout triggers.
Run 2: reads transitionId="migration-started", resumes from checkpoint,
       writes transitionId="migration-complete" on success.
```

This requires the atom to implement re-entrant logic. It is more complex than
the Temporal approach but is achievable with a disciplined SDK.

### Platform integration

When `requires.hostAccess: true`, the operator additionally injects a
`hostKubeconfig` into the stdin payload. The atom uses this to order managed
services from Cozystack (e.g. create a `HelmRelease` for CNPG).

When `requires.vault: true`, a Vault token is injected via the `vault` field.

Platform wiring details (observability injection, identity-linked SA) are in
ADR-0015.

## Consequences

### Positive

- No external workflow server: eliminates Temporal as an operational
  dependency. The entire execution layer is Kubernetes Jobs — standard,
  well-understood, observable with `kubectl`.
- Zero idle resource consumption: no persistent worker Deployments. Jobs
  are ephemeral; the tenant control plane is quiet between reconcile triggers.
- Wide language support: atoms can be written in any language that can read
  stdin and write stdout. No Temporal SDK is required.
- Blast radius is per-Pod: a broken atom image crashes its own Job, not a
  shared workflow server.
- Simpler debugging: `kubectl logs job/<name>` shows the full atom run.
  No separate Temporal UI needed.

### Negative

- Durability is DIY: transition tracking via `transitionId` requires
  discipline from atom authors. A missed `transitionId` write means a
  side-effect re-runs on restart.
- No built-in retry semantics within a single run: if an activity fails
  mid-reconcile, the Job fails and the operator retries on the next trigger.
  There is no intra-run checkpoint.
- Long-running operations (>30 min) require re-entrant atom logic across
  multiple Job invocations. More complex than Temporal's native long-running
  workflow support.
- Job startup latency: each reconcile trigger pays a Pod scheduling + image
  pull overhead (~1–5 s cold, <1 s warm with prepulled image). For
  high-frequency status probes (30–60 s cron), this may matter.

### Risks

- **Re-entrant correctness**: if an atom does not implement `transitionId`
  correctly, a Job failure during a migration leaves the database in an
  inconsistent state. The catalog linter must check that atoms with known
  risky operations document and test their re-entrant paths.
- **State size limit**: `executorState.custom` is capped at 4 KiB. Atoms that
  need to persist large intermediate state (e.g. a list of shards to migrate)
  must store it in a dedicated `ConfigMap` or `Secret` and reference it via
  `transitionId`.
- **Job fan-out**: in a cluster with many tenants and many atoms, a single
  cron tick produces O(tenants × atoms) Jobs simultaneously. The operator must
  stagger Job creation to avoid API server bursts.
- **stdout contamination**: if the atom image writes anything other than the
  JSON result to stdout (e.g. a library logs to stdout), the operator's JSON
  parse fails. Atoms must write logs to stderr exclusively.
