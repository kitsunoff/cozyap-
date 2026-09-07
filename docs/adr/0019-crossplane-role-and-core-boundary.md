# ADR-0019: Crossplane's Role and the Core/Dependency Boundary

## Status

Proposed. Depends on ADR-0018 (per-tenant instance). Reverses the rejection of
Crossplane recorded in ADR-0015 §2.

## Context

ADR-0015 rejected Crossplane on one ground: its package and function model
requires **installing** a runtime into the control plane, which is
unacceptable in a plane shared with other tenants and incompatible with a
large dynamic community catalog.

Both premises are now false:

- the plane is no longer shared (ADR-0018);
- requirement R6 states the catalog is extended and versioned **by the
  Customer** — curated by a single actor, not a community.

Three further pressures point at Crossplane:

1. **Modularity.** Customers will diverge. Each will want a different set of
   dependency types and integrations, without a platform release per request.
2. **External systems are in scope.** GitLab and GitHub integrations are
   expected; Gitea and others are plausible. These are not Cozystack managed
   applications — they need real controllers.
3. **Terraform provider reuse.** Where a Terraform provider exists for a
   system, Upjet can generate a Crossplane provider from it. This converts
   "customer wants an integration with X" from months of controller work into
   days, which is a capability that can be promised commercially.

Crossplane is therefore adopted — but as **a glue layer plus a fleet of
ready-made controllers**, not as the platform's architecture. A substantial
purpose-built layer exists regardless.

## Decision

### 1. Crossplane is consumed behind a narrow interface

Two separable purchases, taken deliberately:

- **Providers (ready-made controllers)** — the primary value. Writing these
  ourselves is expensive and pointless.
- **The composition engine (XRD, Composition, Functions)** — adopted for
  dependencies only. It is not a library; wherever it is used it owns the CRDs,
  the reconcile loop, the status surface and the error surface.

### 2. The boundary: core is ours, dependencies are Crossplane's

| Layer | Owner | Rationale |
| --- | --- | --- |
| `Application` — build, revisions, rollback, deploy, ingress, credential and observability injection, developer-facing status | **Own controller** (controller-runtime) | Identical across all customers, so modularity buys nothing here; status quality and revisions are critical and both are weak in compositions |
| Dependencies and integrations — database, cache, queue, object storage, VCS, anything a customer adds | **XRD + Composition in a bundle** | This is exactly where customers diverge and where extension without a platform release is required |

Reasons the core is not expressed as a composite resource, given requirement
5.14 specifically:

- **Status and errors.** The user is a developer who does not know Kubernetes
  and must be shown "the build failed, here is the log" or "the database is
  starting". In a composition, status flows managed resource → composite → UI,
  and there is no good mechanism for surfacing a human-readable error upward;
  collecting and interpreting conditions falls to us anyway. An own controller
  owns `status` outright and writes exactly what the UI renders. For a
  requirement whose user does not know Kubernetes, status quality is half the
  product.
- **Revisions and rollback.** Compositions do not provide workload revisions
  (requirement gap G1). In an own controller, revisions are a natural part of
  the model.
- **Ordering and imperative steps.** Build via kpack, wait for the database,
  inject credentials, then deploy — plain code in a controller, implicit
  machinery through composed-resource readiness in a composition.
- **Debuggability.** A stack trace versus `kubectl describe` across a chain of
  four objects plus function logs.

### 3. The core↔dependency contract

Deliberately tiny, so that the dependency implementation stays replaceable:

```text
in:   name, class, parameters, owner
out:  Ready / NotReady + reason
      reference to a Secret carrying connection parameters
      (+ metadata for consumption accounting)
```

This is ADR-0009's typed ports reduced to a single port type — a connection
secret plus readiness. Roughly ten fields. Everything else lives inside the
bundle.

### 4. Design goal: Crossplane stays replaceable

Behind a contract this narrow, the dependency layer can later be reimplemented
with an own controller, a Helm release, or a lighter composition engine without
touching the core. **This is recorded as an explicit design goal**, because
without it Crossplane will diffuse into the core within a couple of quarters.

### 5. External integrations: a three-tier strategy

| Tier | Situation | Approach | Cost |
| --- | --- | --- | --- |
| 1 | A maintained Crossplane provider exists | Use it as-is (e.g. `crossplane-contrib/provider-gitlab`, which already ships namespaced APIs) | none |
| 2 | No provider, but a Terraform provider exists and the system is strategic | Generate with Upjet, curate a **subset** of resources, maintain it ourselves | weeks for the first, days each afterwards |
| 3 | Long tail; a one-off request from a single customer | `crossplane-contrib/provider-terraform` — run a Terraform module as a managed resource | hours |

Tier 3 matters more than it appears: it makes the modularity promise honest.
Not every customer request needs a provider written for it, and there is a
cheap fallback that requires no platform release. It must be part of the bundle
contract from the start (ADR-0020).

### 6. Upjet — what must be known before promising timelines

Upjet reads a Terraform provider's schema and generates Go types, CRDs and
controllers. Its current no-fork architecture loads the Terraform provider as a
Go plugin and invokes its CRUD functions directly rather than shelling out to
the Terraform CLI, so runtime overhead is that of an ordinary controller.

Constraints:

- **CRD explosion.** Generating a whole provider yields one CRD per Terraform
  resource — on the order of a hundred for GitLab, dozens for GitHub.
  Generating everything degrades API discovery. **Rule: always generate a
  curated subset**, combined with selective resource activation (ADR-0018 §3).
- **Generation is not free.** Per-resource configuration — external-name
  mapping, cross-resource references, sensitive fields, late-initialisation —
  is written by hand. Realistically: weeks for the first provider including
  learning Upjet, days to a week for each subsequent one. We then own it
  permanently: releases, upstream updates, CVEs.
- **State semantics differ.** Terraform providers assume a state file; Upjet
  reconstructs state from the managed resource. Resources with weak import
  support or many computed fields map poorly, and this only surfaces per
  resource — budget for verification.
- **Credentials.** Each such provider needs a broadly-scoped token on the
  customer's VCS. In Mode A this concentrates credentials for all tenants; in
  Mode B it is the customer's own responsibility. A further argument that Mode B
  is canonical.
- **Licensing is a checklist item, not a technical one.** Since August 2023
  Terraform and HashiCorp-owned providers are under BUSL. The GitLab, GitHub
  and Gitea providers are maintained outside HashiCorp, but **each must be
  checked against its own LICENSE file before generation**, and whether
  generating a provider from BUSL sources is permissible for a commercial
  platform is a legal question to settle once, in advance.
- Beware stale artefacts: a `provider-jet-gitea` exists but is a personal
  repository on the previous generation (terrajet). Regenerate rather than
  adopt.

### 7. VCS integration is not in v1

Requirement 5.14 does **not** ask for VCS-as-managed-resource. "Fetch source
from the customer's repository" is a git clone with credentials plus a webhook,
which kpack already covers. Creating repositories, managing members, CI
variables and deploy keys is a later stage.

Upjet is therefore, at present, **a justification for the architectural choice
and a commercial capability claim — not sprint work.** The first generated
provider is built when a customer states a concrete requirement for it.

## Consequences

### Positive

- The provider ecosystem plus Upjet gives a credible answer to "customer wants
  an integration with X" without a platform release per request.
- The core keeps full control of the two things requirement 5.14 depends on
  most: developer-facing status and revisions.
- The narrow contract keeps the Crossplane commitment reversible.

### Negative

- Two technologies in one product: an own controller and a composition engine.
  Contributors must learn both, and the boundary must be policed.
- We become maintainers of any Upjet-generated provider, indefinitely.
- Crossplane closes **none** of the following, all of which remain ours to
  build: the `Application` CRD and controller, revisions and rollback, kpack
  and registry and webhooks, credential injection, observability and secrets,
  the blueprint catalog and its versioning, consumption accounting, the UI and
  its view descriptors, the bundle installer, the RBAC model, and both
  installation modes.

### Calibration

For the current requirement's dependency set the "ready-made controllers"
reduce mostly to `provider-kubernetes` and `provider-helm`, because the
databases, caches and queues are already Cozystack managed applications with
their own operators. The large provider ecosystem pays off specifically for
resources **outside** Cozystack — which, given the expected VCS integrations,
is now established as in scope. That is what makes Crossplane the right choice
rather than a lighter composition engine.

## Relationship to other ADRs

- **ADR-0009** — the atom contract survives only as §3, the core↔dependency
  contract.
- **ADR-0015** — §2's rejection of Crossplane is reversed here.
- **ADR-0017** — out of scope for v1 per requirement 5.14.
- **ADR-0018** — the instance model that makes installation-based
  extensibility acceptable.
- **ADR-0020** — the bundle that carries XRDs, Compositions and everything
  Crossplane cannot carry.
