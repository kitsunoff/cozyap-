# ADR-0018: Platform Instance Model and Installation Modes

## Status

**Superseded by ADR-0021, ADR-0022 and ADR-0023 (2026-09-14.)** Its instance
model and its statement of Mode A's cluster-scoped-CRD limit survive in
`docs/prior-art.md` §3. Kept for historical context.

Proposed. Supersedes the two-plane model of ADR-0015 and resolves ADR-0012.

Driven by requirement 5.14 (`docs/requirements/5.14-developer-platform.md`).

## Context

ADR-0015 designed a **shared** declaration plane: one host API server holding
all tenants' `AtomInstance` objects, with execution directed into tenant zones
by a placement mechanism. That design carried a large amount of accidental
complexity — two placement axes, credential bridging between planes, a host
scheduler that had to be hardened against cross-tenant abuse, and an open
question about output materialisation across placements.

All of that complexity exists to solve one problem: **arbitrary tenant or
community code must not run in a plane shared with other tenants.**

The problem disappears if the plane is not shared.

Cozystack already provisions each tenant its own Kubernetes cluster. A
tenant-owned control plane therefore is not a new cost centre — it already
exists. Installing platform controllers into it is an incremental cost of
a few hundred megabytes, not an additional API server.

## Decision

### 1. The platform is an instance, not a shared service

A Cozyap developer platform is **installed into a control plane that the
tenant owns**. There is no shared plane holding other tenants' objects, no
host-side execution of tenant code, and no cross-plane credential bridging.

### 2. Two installation modes

| | Mode A — host-resident | Mode B — dedicated |
| --- | --- | --- |
| Platform components run in | the Cozystack host cluster | the tenant's managed Kubernetes cluster |
| Cost per tenant | low (shared components) | one platform footprint per tenant |
| Bundle set | **one common set per installation** | per-tenant |
| Provider versions | **common per installation** | per-tenant |
| Tenant personalisation | RBAC subset of the common catalog | own catalog |
| Suits | a single customer, or tenants that are uniform | multiple customers with divergent requirements |

**Mode B is canonical.** Mode A is the degenerate case: "the host cluster is
also a cluster, it just has one owner."

### 3. Mode A's limitation must be stated, not implied

An XRD creates a **CRD, and CRDs are cluster-scoped**. Crossplane v2 makes
composite and managed resources namespaced, but the definitions remain
cluster-wide. Consequently, in Mode A:

- the set of XRDs and Compositions is common to all tenants of the
  installation;
- the set of providers and **their versions** is common; two customers needing
  different major versions of the same provider do not fit in one cluster;
- composition functions are Deployments, also common.

Therefore, in Mode A **per-tenant modularity degrades to an RBAC-scoped subset
of a common catalog**. It is not "your own bundle set"; it is "you are
permitted these five composite types out of the installation's twenty".

This wording is deliberate and must survive into product material. Selling
Mode A with a promise of per-customer modularity would be a misrepresentation.

**Mitigation for Mode A**, where it is nevertheless required: use Crossplane
v2 managed resource definitions to selectively activate only the provider
resources actually needed, instead of installing every CRD a provider ships.
For providers generated from large Terraform providers this is the difference
between a working API server and degraded discovery.

### 4. Implementation rule that keeps the two modes from diverging

> The platform always reaches its target cluster through an explicit client
> taken from configuration. It never relies on in-cluster assumptions about
> being the host.

With this rule the installation mode is a deployment parameter, not a branch
in the code. Without it, Mode A inevitably accumulates assumptions ("we are in
the host, we have access to everything, no kubeconfig needed") and Mode B
later requires rewriting rather than configuration.

Mode B is therefore implemented **first**; Mode A follows as the degenerate
case.

### 5. Child tenants sharing a parent's control plane

A child tenant may use the parent's platform instance instead of receiving its
own. This is permitted **only for trusted child tenants** — organisational
units of the same customer, not a provider-to-customer relationship.

Constraints:

- Isolation is per namespace, using Crossplane v2 namespaced composite and
  managed resources with a namespaced `ProviderConfig` carrying that child's
  own credentials.
- **Escalation vector:** a child creating a managed resource that references
  the parent's cluster-scoped `ClusterProviderConfig` obtains the parent's
  credentials. Crossplane does not prevent this. An admission policy
  (Kyverno or ValidatingAdmissionPolicy) forbidding references to cluster-scoped
  provider configurations from child namespaces is **mandatory from day one**,
  not a later hardening step.
- The extension-installation problem is not eliminated, only relocated to the
  parent/child boundary: a child cannot add a provider or bundle the parent has
  not installed, and is pinned to the parent's platform version and catalog.
  For a platform-team-to-product-team relationship this is the desired
  governance model; for a provider-to-customer relationship it is not
  acceptable and the child must get its own instance.

## Consequences

### Resolved by this ADR

- **ADR-0012 (execution layer) is decided in favour of Pods (ADR-0011).** The
  principal argument for Temporal was durability; the principal argument
  against it was that an outage suspends all tenants simultaneously. With a
  per-tenant plane, a Temporal cluster per tenant is not economically
  defensible, and the shared-blast-radius argument against Pods no longer
  applies. ADR-0011 is promoted, ADR-0010 rejected.
- **ADR-0015 is largely superseded.** Both placement axes collapse — everything
  runs in the instance's own plane. Removed with them: `host-system` placement
  and its trust gating, credential bridging between runner and target planes,
  host scheduler hardening, the shared-etcd scale concern, and the open
  question on output materialisation across placements.
- **ADR-0015 §2's rejection of Crossplane is void.** It rested on installing
  extensions into a *shared* plane. The plane is no longer shared, and
  requirement R6 states the catalog is curated by the Customer rather than by a
  community. See ADR-0019.

### Positive

- A large amount of unimplemented complexity is deleted rather than built.
- Security posture is simpler to reason about: the trust boundary is the
  cluster, which Cozystack already establishes.
- Per-customer divergence (different bundles, different provider versions)
  becomes possible at all, which the shared plane could not offer.

### Negative

- **The fleet view is lost.** ADR-0015 justified the shared plane partly by the
  single pane over all instances. Cross-instance governance now requires an
  explicit aggregation layer, which is not designed yet.
- **Fixed overhead per tenant.** Every instance carries platform controllers,
  kpack, and a registry. A project running one small application pays for a
  whole platform footprint. Who is billed for it — absorbed into the service
  price or shown to the customer — is a product decision that interacts with
  requirement R7 and is **not yet made**.
- Two installation modes mean two installation paths, two test matrices and two
  sets of documentation, mitigated but not eliminated by §4.

## Open questions

- Cross-instance governance and fleet view — needed at all, and if so, at what
  layer?
- Billing of the platform instance overhead itself (interacts with R7 / gap G2).
- Whether "project" in requirement 5.14 maps one-to-one onto a Cozystack
  tenant, which determines whether §5 (child tenants) is needed at all.

## Relationship to other ADRs

- **ADR-0010 / 0011 / 0012** — decision recorded here; Pods win.
- **ADR-0015** — superseded in its two-plane model and its rejection of
  Crossplane; its `execution.mode` gradation survives, minus the `stateful`
  tier (out of scope per requirement 5.14).
- **ADR-0019** — the role of Crossplane inside an instance.
- **ADR-0020** — what an installable bundle contains.
