# ADR-0025: Log Access

## Status

Proposed, 2026-09-15. Closes open question O13 of `docs/decisions.md`, answers
question 5 of ADR-0022, and refines ADR-0024 §2.

## Context

Requirement 5.14 specifies a developer who does not know Kubernetes, and
`prior-art.md` §1 records that showing such a user *why* something failed is
half the product. Two log sources matter and neither is reachable from a
browser:

- **Application pods** run in a tenant Kubernetes cluster. The console talks to
  the Cozystack host API server and has no route to another cluster's API.
- **Build pods** are created by kpack in the host cluster and garbage-collected
  when the build completes, so `pods/log` has a short and unpredictable window.

ADR-0024 §2 adopted the console's no-backend model. Taken literally it leaves no
place for this.

## What already exists

Cozystack collects both classes of log into VictoriaLogs without any work from
this platform:

- The `monitoring` package creates a `VLCluster` per tenant — `vlinsert`,
  `vlselect` and `vlstorage` with a retention period — read today by a Grafana
  datasource at `vlselect-<name>.<namespace>.svc:9471`.
- The `monitoring-agents` addon runs fluent-bit as a DaemonSet **inside each
  tenant Kubernetes cluster**, shipping every pod's output to
  `vlinsert-<name>.<tenant-namespace>.svc:9481`.

The stream is labelled with everything the platform needs to select by:

```text
_stream_fields = stream, kubernetes_pod_name,
                 kubernetes_container_name, kubernetes_namespace_name
plus filters adding: tenant=<namespace>, cluster=<cluster name>
```

So application logs are already stored, already per-tenant, and already labelled
by cluster, namespace, pod and container. What is missing is a path from the
console to them that respects who is asking.

## Decision

### 1. Logs are a subresource, served by an aggregated API server

```text
GET /apis/cozyap.io/v1alpha1/namespaces/<ns>/blueprintinstances/<name>/logs
      ?node=api&since=1h&container=app&follow=true&filter=<text>
GET .../workloads/<name>/logs
```

The instance-level subresource is the developer-facing default — "show me my
application's logs", across its nodes, including the build. The workload-level
one narrows to a single component.

This is the shape Kubernetes already uses for `pods/log`, `pods/exec` and
`metrics.k8s.io`: a read path that proxies and shapes data behind the API
server, rather than a store.

### 2. The invariant that ADR-0024 §2 was protecting, restated

> **The platform adds no API surface outside the Kubernetes API.**

That is the property worth keeping, and an aggregated API server keeps it: the
same `/apis/` prefix, the same credentials, the same client already in the
console, the same RBAC, no second origin and no CORS. A standalone REST service
would have broken it; this does not. ADR-0024 §2's "no backend of our own" is
narrowed to that statement.

### 3. The server builds the query; the caller never supplies one

This is the reason for the component, not an implementation detail.

The caller passes a time range, an optional container, an optional free-text
filter, and `follow`. The **stream selector is constructed by the server** from
the identity of the object being read: it resolves the instance to its nodes,
each node to its pods and their target cluster and namespace, and emits LogsQL
scoped to exactly those streams.

The cheap alternatives fail precisely here:

| Alternative | Why not |
| --- | --- |
| Console queries VictoriaLogs directly | second origin, second authentication, CORS; breaks §2 |
| Console reaches VictoriaLogs through the API server's service proxy | works with Kubernetes credentials and needs no new component, but RBAC on a service proxy is all-or-nothing: whoever may proxy may run **any** LogsQL, so one application's owner reads another's logs inside the same tenant |

Access control is then ordinary RBAC — `get` on `blueprintinstances/logs` — and
the tenant boundary is enforced twice over, by RBAC on the object and by a
selector the caller cannot influence.

### 4. A server of ours, in our API group

Two homes were considered. Extending `cozystack-api`, which is already an
aggregated API server, adds no new component and would give every Cozystack
application logs — a genuinely larger benefit. It is rejected for now because
the selector construction in §3 requires knowing this platform's objects and
their node-to-pod mapping, which does not belong in Cozystack's API server, and
because it would couple the platform's release to Cozystack's.

Serving `cozyap.io` ourselves keeps that knowledge where it lives. If the
mapping is later generalised, moving the subresource upstream is a contained
change.

**The operational hazard must be stated:** a failing `APIService` degrades
discovery for every client of the cluster, not only for callers of this path.
Mitigations are mandatory rather than advisable — at least two replicas, a
readiness probe that fails closed, and no dependency in the serving path on
anything that can be slow. The path is read-only, which limits the blast radius
but does not remove it.

### 5. Build logs need collection in the host cluster

Application logs are already collected (see above). kpack builds run in the host
cluster, and whether the host's own pod logs reach a `VLCluster` by default is
**not established** — `monitoring-agents` ships fluent-bit, but its deployment
into the host cluster rather than into tenant Kubernetes clusters has not been
confirmed. This is recorded as V6 in `docs/decisions.md`.

If they are not collected, the platform ships a collector scoped to its build
namespace rather than turning on cluster-wide collection it does not own.

## Consequences

### Positive

- The largest undesigned area in the model is closed with one small component
  and a mechanism Kubernetes already has.
- Application log collection is free: it is already running.
- Per-application scoping is enforced by construction, not by trusting a query.
- Build logs outlive the build pod, which `pods/log` cannot offer.
- The console gains logs without learning a second protocol.

### Negative

- A third controller-shaped component, after the two of ADR-0021. It does not
  violate that ADR's rule — it reconciles nothing and owns no type — but it is
  still ours to run, and an `APIService` has failure modes a Deployment does not.
- Log retention becomes a product-visible property. VictoriaLogs retention is
  configured per storage, and a rollback to a revision whose build log has aged
  out will show no log. Retention and `revisionHistoryLimit` should be chosen
  together.
- Streaming with `follow` means long-lived connections through the aggregation
  layer, which is a known source of timeout and proxy-buffering problems.

## Open questions

1. Whether `follow` is in the first version at all, or whether polling a bounded
   window is enough for the first release.
2. How much free-text filtering to expose. A full LogsQL passthrough re-creates
   the injection problem §3 exists to prevent; a substring match is safe and may
   be insufficient.
3. Whether build logs should additionally be summarised into
   `WorkloadRevision.status` — the failing step and the last lines — so that the
   common question is answered without opening a log view at all.

## Relationship to other ADRs

ADR-0001 through ADR-0017 no longer exist as files; they were removed with the
retired series and are retrievable from git history. Their durable conclusions
are in `docs/prior-art.md`.

- **ADR-0022** — question 5 is answered here.
- **ADR-0024 §2** — "no backend of our own" is narrowed to "no API surface
  outside the Kubernetes API".
- **ADR-0006** — rejected an aggregated API server backed by PostgreSQL as
  over-engineered for catalogue *storage*. That rejection stands; this is a read
  and proxy path, which is what the aggregation layer is for.
