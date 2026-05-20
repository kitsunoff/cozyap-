# ADR-0009: Atom Contract

## Status

Proposed. Follows ADR-0008 (Move to Reactive Reconcile Model).

This ADR is **executor-neutral**: it defines the public contract of an atom
regardless of which execution layer is chosen in ADR-0010/0011. Execution
semantics (invocation protocol, retry/timeout policy, transition tracking,
worker deployment) live in ADR-0010 (Temporal) and ADR-0011 (Pods with
stdin/stdout). The decision between the two options is in ADR-0012.

## Context

Atoms are the unit of composition in Cozyap. Anything bigger than a single
Kubernetes object — and many things smaller than a Helm release — is built by
wiring atoms into a graph. For that to work, the contract between atoms has
to be precise enough that:

- a downstream atom does not care which upstream atom produces its input as
  long as the port types match,
- a tenant can swap one backend atom for another (CNPG Postgres → external
  RDS Postgres) without rewriting the graph,
- the platform controller can detect change and propagate it without polling
  or external event bus,
- catalog versioning is unambiguous and reproducible.

This ADR defines that contract: the two CRDs (`AtomTemplate` and
`AtomInstance`), the typed port system, the output materialisation format,
the conditions schema, and the versioning policy.

## Decision

### Two CRDs, mirrored

The atom layer follows the same template→instance split as
ApplicationTemplate→Application:

| CRD | Scope | Mutability | Owner |
|-----|-------|------------|-------|
| `AtomTemplate` | Cluster-scoped | Read-only for tenants, written by the catalog | Platform |
| `AtomInstance` | Namespaced | Reconciled by the controller, referenced by `Application` | Application (ownerRef) |

An `AtomInstance` is created by the platform controller for every node in
an `Application`'s graph. Tenants do not write `AtomInstance` directly; they
edit the parent `Application.spec`.

### AtomTemplate

```yaml
apiVersion: cozyapps.cozystack.io/v1alpha1
kind: AtomTemplate
metadata:
  name: postgres
  labels:
    cozyapps.cozystack.io/version: "1.2.0"   # semver, immutable for a given object
spec:
  # Human-readable
  displayName: "PostgreSQL"
  description: "Managed PostgreSQL cluster"
  icon: "https://catalog.cozystack.io/icons/postgres.svg"
  category: database

  # Typed ports
  inputs:
    backupBucket: { type: secret-ref, required: false }
  outputs:
    credentials: { type: secret-ref }
    service:     { type: service-ref }

  # Parameter schema
  params:
    - name: version
      type: enum
      enum: ["14", "15", "16"]
      default: "16"
    - name: replicas
      type: int
      default: 1
      ui: { label: "Replicas", min: 1, max: 5 }
    - name: storage
      type: enum
      enum: ["10Gi", "50Gi", "100Gi"]
      default: "10Gi"

  # Executor image. How this image is invoked — SDK registration, stdin/stdout
  # protocol, etc. — is defined by the execution layer (ADR-0010 or ADR-0011).
  # Execution-layer-specific fields (workflowType, command overrides, etc.) are
  # added by each ADR on top of this base shape.
  executor:
    image: ghcr.io/cozystack/cozyap-atoms/postgres:1.2.0
    resources:
      requests: { cpu: 100m, memory: 128Mi }
      limits:   { cpu: 500m, memory: 512Mi }
    # Optional dataplane access requirements; the operator wires the
    # appropriate kubeconfig as a projected secret.
    requires:
      dataplaneAccess: true    # needs kubeconfig to the target Environment
      hostAccess:      false   # does not order managed services from host
      vault:           false

  # Reconcile timing — executor-neutral. Invocation mechanism is in ADR-0010/0011.
  reconcile:
    timeout: 30m
    cron: 4h            # drift-heal safety net

  # Read-only health probe. Condition names published here are the atom's
  # public API; removing or renaming any is a major version bump.
  status:
    cron: 60s
    publishes:
      - ClusterReady
      - BackupOk
      - StorageHealthy

  # User-triggered one-shots, carried over from ADR-0001.
  actions:
    - name: backup
      displayName: "Create backup"
      params: [...]
```

Execution-layer-specific fields (`workflowType` for Temporal, `command`
overrides for Pods) are defined in ADR-0010 and ADR-0011 respectively. They
extend this base shape without changing the fields defined above.

### Atom versioning

- Atoms are versioned by **semver**. The version sits on a label
  (`cozyapps.cozystack.io/version`), not in the object name, so the same
  `metadata.name` can have many version objects in the catalog.
- An `ApplicationTemplate` references atoms with **explicit patch-level
  pins**: `postgres@1.2.0`. No ranges, no implicit upgrades.
- When the catalog publishes `postgres@1.2.1`, running applications continue
  to use `1.2.0`. The catalog UI flags "newer version available" against
  affected `ApplicationTemplate` versions; updating an `ApplicationTemplate`
  to pin the new atom version produces a new `ApplicationTemplate` version,
  which tenants opt into.
- Breaking changes (removed outputs, removed conditions, renamed port,
  parameter type change) require a **major** version bump.
- Adding outputs, conditions, or optional parameters is **minor**.
- Internal changes (different operator backend, bug fix in reconcile logic,
  changed image tag inside a step) are **patch**.

### Port type system

Thirteen types form the v1 vocabulary. Every port — input or output — is
declared with exactly one of these:

| Type | Carries | Backed by |
|------|---------|-----------|
| `secret-ref` | Reference to a Kubernetes Secret (with optional key) | `Secret` |
| `tls-secret-ref` | Reference to a TLS Secret (`tls.crt`/`tls.key` keys assumed) | `Secret` (type `kubernetes.io/tls`) |
| `configmap-ref` | Reference to a ConfigMap (with optional key) | `ConfigMap` |
| `service-ref` | Reference to a Service (with optional port name) | `Service` |
| `pvc-ref` | Reference to a PersistentVolumeClaim | `PersistentVolumeClaim` |
| `serviceaccount-ref` | Reference to a ServiceAccount | `ServiceAccount` |
| `workload-ref` | Reference to a workload object (Deployment/StatefulSet) | `Deployment` / `StatefulSet` |
| `image-ref` | Container image with tag/digest | string (semantic) |
| `ingress-host` | Hostname suitable for an Ingress rule | string (semantic) |
| `string` | Arbitrary UTF-8 string | scalar |
| `number` | Numeric (integer or float) | scalar |
| `boolean` | True/false | scalar |
| `any` | Escape hatch; opaque payload | bytes (base64) |

#### Subtyping

Compatibility is **one-way only**:

```text
tls-secret-ref  →  secret-ref      (a TLS Secret is a Secret)
image-ref       →  string          (an image is a string)
ingress-host    →  string          (a host is a string)
```

The reverse never holds — a generic `string` does not satisfy `image-ref`,
and a `secret-ref` does not satisfy `tls-secret-ref`. Siblings under the
same parent (`image-ref` ↔ `ingress-host`) do not interchange.

`any` is the universal sink (any type → `any`); it never serves as a source
for a typed input (the builder UI warns when the inferred type is `any`).

### Output materialisation

Atom-produced k8s objects (Secret, Service, etc.) are an **internal detail**
of the atom. The contract surface is `AtomInstance.status.outputs`:

```yaml
status:
  outputs:
    credentials:
      ref:
        kind: Secret
        name: my-app-postgres-credentials
        key: connection-string          # optional, port-type-dependent
      fingerprint: "sha256:9b74c9897bac…"   # see "Cascade detection"
      observedAt: 2026-05-20T10:32:00Z

    service:
      ref:
        kind: Service
        name: my-app-postgres-rw
        port: postgres                  # optional, named port
      fingerprint: "sha256:e3b0c44298fc…"
      observedAt: 2026-05-20T10:32:00Z
```

Rules:

1. **All output refs are same-namespace.** The referenced object lives in the
   same namespace as the `AtomInstance` (i.e. the tenant namespace).
   Cross-namespace refs are not allowed in v1.
2. **Atom owns the referenced object.** The atom's reconcile logic creates,
   updates, and (on remove) deletes the referenced k8s object. The
   controller sets ownerRefs from the k8s object back to the `AtomInstance`
   for garbage collection.
3. **Output keys are stable.** Once an atom has declared an output port name
   in `AtomTemplate.spec.outputs`, removing or renaming it is a major
   version bump.
4. **Optional `key`** narrows the binding (e.g. a specific key inside a
   Secret); without it, the downstream atom consumes the whole object.
5. **Optional `port`** (for `service-ref`) names which port to use when the
   service exposes several.

### Cascade detection

`fingerprint` is the change-detection primitive. It is a stable hash of the
**materialised content** an output represents — not the k8s object's
`resourceVersion` (which advances on irrelevant resyncs).

- For `secret-ref` / `configmap-ref` / `tls-secret-ref`: SHA-256 of the
  canonical-JSON-encoded `data` (only the keys referenced by `key`, if set).
- For `service-ref`: SHA-256 of `{clusterIP, ports[]}`.
- For `pvc-ref` / `serviceaccount-ref` / `workload-ref`: SHA-256 of
  `{name, namespace, uid}` (object identity).
- For `image-ref`: SHA-256 of the resolved image digest (or the full
  `repo:tag@sha256:…` string).
- For scalars (`string`/`number`/`boolean`/`ingress-host`): SHA-256 of the
  value.
- For `any`: SHA-256 of the raw payload.

The atom's reconcile logic computes and writes the fingerprint as the last
step. The controller does **not** trust k8s object mutation events for change
detection — it diffs `fingerprint` between successive reconciles.

When `AtomInstance.status.outputs.<port>.fingerprint` differs from the
previous observed value, the controller enqueues a reconcile for every
downstream `AtomInstance` whose `spec.inputs.<port>.from` references this
output. Identical fingerprints (no-op reconciles) do not propagate. This is
the **cascade** — implemented purely as a controller watch loop, no external
event bus required.

### AtomInstance

```yaml
apiVersion: cozyapps.cozystack.io/v1alpha1
kind: AtomInstance
metadata:
  name: my-app-postgres
  namespace: tenant-acme
  ownerReferences:
    - apiVersion: cozyapps.cozystack.io/v1alpha1
      kind: Application
      name: my-app
      controller: true
      blockOwnerDeletion: true
spec:
  templateRef:
    name: postgres
    version: "1.2.0"          # explicit pin (see "Atom versioning")

  # Parameter values, validated against AtomTemplate.spec.params
  params:
    version: "16"
    replicas: 3
    storage: "50Gi"

  # Input bindings — references to outputs of sibling AtomInstances
  inputs:
    backupBucket:
      from:
        atom: my-app-s3       # AtomInstance name in same namespace
        port: bucketSecret

  # Pin / track per output. Lives on the consumer side; pinning is a
  # tenant-level choice, not an atom-level choice.
  pins: {}
    # Example: freeze an image-ref output at a specific digest.

status:
  phase: Ready
    # Pending | Reconciling | Ready | Failed | Removing

  observedGeneration: 7

  conditions:
    - type: Reconciled
      status: "True"
      state: ok
      reason: ReconcileSucceeded
      message: "last reconcile applied no changes"
      lastTransitionTime: 2026-05-20T10:32:00Z
      observedGeneration: 7

    - type: ClusterReady          # from AtomTemplate.spec.status.publishes
      status: "True"
      state: ok
      reason: ClusterHealthy
      message: "all 3 instances streaming WAL"
      lastTransitionTime: 2026-05-20T10:31:45Z

    - type: BackupOk
      status: "False"
      state: warn
      reason: BackupStale
      message: "no successful backup in last 36h"
      lastTransitionTime: 2026-05-19T22:00:00Z

  outputs:
    credentials:
      ref: { kind: Secret, name: my-app-postgres-credentials, key: connection-string }
      fingerprint: "sha256:9b74c9897bac…"
      observedAt: 2026-05-20T10:32:00Z

  lastReconciledAt: 2026-05-20T10:32:00Z
  lastReconcileRun:
    id: my-app-postgres-reconcile-x7k9m   # execution-layer handle (workflow ID, Job name, etc.)
    phase: Succeeded
```

### Conditions schema

Each `AtomInstance.status.conditions` entry follows the Kubernetes Condition
convention with one extension:

| Field | Type | Source | Notes |
|-------|------|--------|-------|
| `type` | CamelCase string | atom-declared in `AtomTemplate.spec.status.publishes`, plus platform-reserved types | Stable public API of the atom |
| `status` | `"True"` / `"False"` / `"Unknown"` | atom | k8s-standard, so `kubectl wait --for=condition=…` works |
| `state` | `ok` / `warn` / `error` / `info` / `unknown` | atom | extension; UI uses it for finer-grained colouring than True/False |
| `reason` | CamelCase string | atom | machine-readable label |
| `message` | human-readable string | atom | one short sentence |
| `lastTransitionTime` | RFC3339 timestamp | controller (only on transition) | |
| `observedGeneration` | int64 | controller | matches `spec` generation when condition was last refreshed |

**`state` vs `status` mapping (informational, not enforced):**

| `state` | typical `status` |
|---------|------------------|
| `ok` | `"True"` |
| `warn` | `"True"` (degraded but functional) |
| `error` | `"False"` |
| `info` | `"True"` (transient, e.g. reconciling) |
| `unknown` | `"Unknown"` |

The atom is free to deviate (e.g. report `warn` with `status: "False"` if
business logic requires it).

**Reserved conditions** (the platform writes these, atom authors must not
declare them):

| `type` | Meaning |
|--------|---------|
| `Reconciled` | Result of the most recent reconcile run (success / failure / no-op) |
| `InputsResolved` | All `spec.inputs` are bound to existing upstream outputs |
| `Deleting` | `deletionTimestamp` set, removal in progress |

**Aggregation:**

- An `AtomInstance`'s aggregate state is the **worst** of its conditions'
  `state`, in order `error > warn > info > ok > unknown`.
- A condition is required only if the atom declared it in
  `AtomTemplate.spec.status.publishes`. Missing conditions are treated as
  `unknown` for aggregation.

### Executor-neutral reconcile invariants

The execution layer (ADR-0010 or ADR-0011) governs how reconcile, status, and
action runs are started, serialised, and reported back. Regardless of which
execution layer is chosen, the following invariants hold:

- **Serialisation.** Exactly one reconcile runs per `AtomInstance` at a time.
  The execution layer must enforce this; concurrent reconciles for the same
  `AtomInstance` are a contract violation.
- **Read-only status.** The status run is a health probe — it MUST NOT write
  to k8s objects in the dataplane. Running concurrently with reconcile is
  permitted.
- **Actions are concurrent.** Multiple actions may run in parallel and may
  overlap with a running reconcile. Conflicts are the atom author's
  responsibility.
- **Output via return value.** The reconcile run returns a typed result
  (outputs + conditions). The operator reads the result and patches
  `AtomInstance.status`. The executor does NOT patch CRs directly; it does
  not hold RBAC to the `cozyapps.cozystack.io` API group.
- **Uniform payload.** All three run categories receive the same context:
  params, resolved input refs, atom-instance metadata, and an environment
  handle (kubeconfig or equivalent). The execution layer delivers this
  payload; the format is defined in ADR-0010 or ADR-0011.

Concrete invocation rules, retry/timeout policies, transition tracking for
side-effect operations, and worker deployment lifecycle are in ADR-0010
(Temporal) and ADR-0011 (Pods with stdin/stdout).

### What an atom MUST guarantee

1. **Idempotence.** Running reconcile twice in a row with no external
   change MUST produce the same outputs (same fingerprints) on the second
   run. Enforced by the catalog linter (ADR-0016).
2. **Output stability.** A declared output port appears in
   `status.outputs` after every successful reconcile, with a stable
   `ref.kind` and a stable schema for `ref.name` / `ref.key`.
3. **Condition stability.** Every condition `type` in
   `AtomTemplate.spec.status.publishes` is published by every status
   run. Missing conditions are a contract violation (caught by the linter).
4. **No cross-namespace writes.** All k8s objects the atom creates live in
   the tenant namespace. (Atoms wrapping managed services may interact with
   the host Cozystack API via injected kubeconfig; that path is covered in
   ADR-0015.)
5. **Removal cleans up.** When reconcile runs with `deletionTimestamp` set
   on the parent Application, it removes everything it created and clears
   `status.outputs`.

## Consequences

### Positive

- A single, narrow contract surface (outputs + conditions + params) that
  works the same across CNPG, Zalando, RDS, or hand-rolled StatefulSets.
- Fingerprint-driven cascade eliminates the external event bus and makes
  no-op reconciles cheap.
- Patch-level pinning of atoms in `ApplicationTemplate` keeps running
  applications reproducible and protects tenants from accidental upgrades.
- k8s-standard Condition shape preserves `kubectl wait`, kube-state-metrics,
  and Prometheus alerting compatibility; the `state` extension is additive.
- `AtomInstance` as a real CR makes per-atom RBAC, watch, and per-atom
  reconciliation possible without inventing controller state.
- Executor-neutrality means the atom contract does not need to change when
  the execution layer decision (ADR-0012) is made.

### Negative

- Two CRDs per atom layer (`AtomTemplate` + `AtomInstance`) plus the
  Application graph means more objects in etcd than the original
  one-template-one-app model. For typical applications (≤ 10 atoms) this
  is still small, but pathological graphs need a quota.
- Fingerprint computation is per-port-type and lives in code; the atom
  contract leaks a little into the controller. Adding a new port type
  requires updating the fingerprint algorithm table above.
- The `state` field is non-standard. Tooling outside Cozyap (e.g. Argo CD)
  will ignore it; it is purely a UI/UX enrichment.
- Patch-pin policy means security fixes do **not** propagate
  automatically. The catalog UI must make "outdated atom" loud, and ISP
  operators need a sane bulk-update flow (deferred, but flagged here).

### Risks

- If an atom's reconcile writes the wrong fingerprint (e.g. accidentally
  includes a timestamp in the hash input), downstream atoms reconcile on
  every cron tick — quietly turning the cluster into a busy-loop. The
  catalog linter must run the idempotence check (reconcile twice, expect
  identical fingerprints).
- The platform-reserved `Reconciled` / `InputsResolved` / `Deleting`
  conditions collide with any pre-existing atom that used those names.
  Reserved before any catalog item ships.
- Same-namespace-only output refs are a v1 simplification that blocks
  shared-managed-service patterns (e.g. one Postgres across two
  Applications). ADR-0013 must close this with an explicit `ExternalRef`
  atom or cross-Application import mechanism.
