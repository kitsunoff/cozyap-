# ADR-0023: The Application Boundary and Service-to-Service Access

## Status

Proposed, 2026-09-14. Depends on ADR-0021 and ADR-0022.

## Context

An earlier position held that an application is a pivot: a view over resources
joined by label, owning nothing. Constraint V5 supports that reading, and it is
cheap.

It breaks on the first question about one service calling another. A view cannot
grant or deny access, because there is nothing to enforce. And three different
demands are usually made of "application" at once:

| Demand | Needs a view | Needs ownership | Needs a runtime boundary |
| --- | --- | --- | --- |
| Show everything belonging to this application | yes | — | — |
| Create, roll back or delete it as a unit | — | yes | — |
| Decide who may talk to whom | — | — | **yes** |

A pivot answers only the first. `brick-model.md` §14 stated plainly that there
is no cross-application graph, which was an accurate description of the gap
rather than a decision.

OpenChoreo answers this by making its `Project` a bounded context that owns its
components and becomes a **Cell** at runtime: its own namespace, its own network
policies, gateways in four directions, and mTLS on all traffic including
intra-cell. Components inside a cell talk freely; everything crossing the
boundary goes through a gateway. Access itself is declared by the two parties —
the producer publishes an endpoint with a visibility, the consumer declares a
dependency and the address is injected — and never as a central graph.

## Decision

### 1. An application is a boundary; the view is a consequence

A `BlueprintInstance` owns **one namespace in its target cluster**, created with
a default-deny NetworkPolicy. Components inside it reach each other directly.
Nothing enters or leaves without an explicit declaration.

Without the default-deny, everything below is decoration: visibility levels
describe intent that nothing enforces. It is therefore part of the instance's
baseline, not a hardening pass.

Aggregation for the UI then falls out of ownership and needs no label
convention of its own.

### 2. Access is declared by both parties, never centrally

Three levels, three different authors, and the connection is at none of them:

| Level | Author | Declares |
| --- | --- | --- |
| `Blueprint` | package author | the boundary and what a node may expose — not who talks to whom |
| Producer's `Workload.spec.endpoints` | the producing application | **who may reach me** |
| Consumer's `Workload.spec.dependencies` | the consuming application | **what I need and where to inject it** |

```yaml
# producer
spec:
  endpoints:
    - { name: http, port: http, visibility: cluster }

# consumer
spec:
  dependencies:
    - app: billing
      endpoint: http
      env: { address: BILLING_URL }
```

`workload-controller` resolves the dependency: it finds the producing
`BlueprintInstance`, checks that the named endpoint is published at a sufficient
visibility, computes the address, injects the environment variables, and emits
the network policy on both sides. A dependency on an endpoint that is not
published, or not published widely enough, fails with that as the message —
it is not silently opened.

A central table of who may call whom is deliberately absent. It would make the
platform team a bottleneck for every integration between two product teams,
which is the failure mode this model exists to avoid.

### 3. Visibility has three levels, and the third is not cosmetic

| Value | Reachable from | Enforced by |
| --- | --- | --- |
| `app` | the same `BlueprintInstance` | namespace boundary; the default |
| `cluster` | another application in the same tenant Kubernetes cluster | `CiliumNetworkPolicy` on both sides |
| `public` | through ingress, with TLS | ingress plus whatever authentication the endpoint declares |

Every endpoint is `app` unless it says otherwise.

The third level carries a constraint that OpenChoreo does not have, because its
cells are namespaces in one data plane while ours may be **different clusters**.
`targetCluster` is an instance parameter, so two applications of the same tenant
can land in different Kubernetes clusters with no shared network. Between them a
network policy is meaningless and `public` is the only path.

This must be surfaced in the UI at the point of choosing, not discovered
afterwards: selecting `cluster` visibility for a consumer in another cluster is
a configuration that cannot work, and the platform knows it at authoring time.

Enforcement is available because Cozystack tenant Kubernetes clusters ship
Cilium as their CNI, alongside Gateway API CRDs and `ingress-nginx`.

### 4. This is the one permitted reference between instances

ADR-0021 forbids cross-instance references, and this is an exception to that
rule. It is kept narrow on purpose:

- **one kind only** — an endpoint, never an arbitrary node output of another
  application;
- **with the producer's consent** — via `visibility`, denied by default;
- **within one tenant** — applications of different tenants do not see each
  other at all;
- **one-directional** — the consumer receives an address; the producer neither
  knows nor depends on the consumer, so no cycle can form between applications.

A request to read something else out of another application is answered with
"publish an endpoint", not with a widening of the rule.

## Consequences

### Positive

- Service-to-service access is expressible at all, which the pivot model could
  not do.
- The platform team does not mediate connections between product teams.
- Default-deny per application is a stronger posture than Cozystack's namespace
  isolation alone, and it comes from a policy the platform generates rather than
  one a developer writes.
- Endpoints and their visibility are the raw material for a service catalogue
  later, at no extra cost now.

### Negative

- The application is no longer a cheap view. It owns a namespace and a policy in
  a cluster it may not share with its consumers, and `workload-controller` gains
  cross-instance resolution.
- A `cluster`-visible endpoint is unreachable from another cluster, which is a
  genuinely confusing failure unless the UI prevents it up front.
- Network policy correctness is now the platform's responsibility. A wrong
  generated policy is an outage, and a too-permissive one is a security finding.
- Adopting resources the platform did not create — constraint V5's "joined by
  annotation or label" — no longer follows from the model, since membership is
  ownership. If adoption is wanted it needs a deliberate read-only mechanism.

## Open questions

1. **May applications of different tenants call each other?** Assumed no, which
   simplifies both the policies and the catalogue. For an ISP serving several
   customers that is almost certainly correct; for one enterprise with internal
   tenants it may not be.
2. **mTLS.** OpenChoreo encrypts all traffic including intra-cell. Nothing here
   does, and retrofitting it means a mesh in the data plane, which constraint V6
   resists.
3. **Egress to third parties.** Default-deny blocks it. Where the allowlist is
   declared — node type, workload, or instance — is undecided.
4. **Endpoint authentication for `public`.** TLS is settled; who authenticates
   the caller is not.

## Relationship to prior art

- **`design/brick-model.md` §14** — "no cross-application graph" is superseded by
  the narrow exception in §4.
- **ADR-0007** — the networking primitives survive; how an application requests
  them is now `Workload.spec.endpoints`.
- **ADR-0021** — the no-cross-instance-references rule and its single exception.
