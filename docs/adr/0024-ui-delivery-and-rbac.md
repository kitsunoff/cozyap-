# ADR-0024: UI Delivery and the RBAC Model

## Status

Proposed, 2026-09-15. Depends on ADR-0021. Closes open questions O3 and O4 of
`docs/decisions.md` and revises ADR-0021's consequences.

## Context

`BlueprintInstance` and `Workload` are ordinary custom resources. `cozystack-api`
projects only HelmReleases carrying the `apps.cozystack.io/application.*`
labels, so the platform's own objects do not appear in the Cozystack dashboard
by default. Requirement 5.14 specifies a user who does not know Kubernetes, so
some surface has to exist.

Three things about `cozystack-ui` decide the shape of that surface:

- It is a **pure SPA that talks directly to the Kubernetes API**. There is no
  backend and no code generation, and that is a stated architectural property
  rather than an accident of the current version.
- Entities are **discovered at runtime** from `ApplicationDefinition` resources
  in the cluster.
- The stack is React with Vite and TypeScript in a pnpm workspace, with
  `packages/{k8s-client,ui,types}` already providing a watch layer, shared
  primitives and types, and RJSF rendering forms from OpenAPI schemas.

A separate application of our own would duplicate authentication, navigation
and the watch layer, and would leave the operator with two panels.

## Decision

### 1. The platform's surface is a plugin inside `cozystack-ui`

Not a fork, not a second application. A plugin mechanism is contributed upstream
first, and the developer-platform UI is its first consumer.

What the mechanism must provide, which is also the smallest useful patch:

1. **Registration of entity types not backed by an `ApplicationDefinition`** —
   today discovery assumes that resource shape; a plugin must be able to declare
   a CRD-backed type with its own list columns, detail tabs and actions.
2. **Route and navigation contribution** so a plugin can own a section.
3. **Access to the existing workspace packages** — `k8s-client` for the watch
   layer, `ui` for primitives, `types` for shared types. A plugin that reaches
   past them re-implements the console badly.

Forms need nothing new: `Blueprint.spec.parameters` is an OpenAPI schema and
RJSF already renders those.

### 2. No backend of our own

The direct-to-API model is adopted rather than worked around, and this is only
viable because ADR-0021 puts real status on real custom resources. Everything
the surface needs — graph state per node, replica counts, the active revision
and its predecessor, resolved bindings, the published URL — is in
`BlueprintInstance.status` and `Workload.status`, read with the viewing user's
own credentials and their own RBAC.

An earlier draft of this design carried a backend-for-frontend to aggregate
status scattered across HelmReleases, kpack and a remote cluster. Writing the
two controllers removed the thing it was aggregating. **The BFF is dropped from
the component list.**

Logs are the one thing this does not cover — pods in a tenant Kubernetes cluster
and kpack's collected build pods are not reachable from the browser. **ADR-0025
resolves it without breaking the model**, by narrowing the rule here to what it
was protecting: the platform adds no API surface *outside the Kubernetes API*.
A `logs` subresource served by an aggregated API server satisfies that; a
standalone REST service would not.

### 3. RBAC: controllers cluster-wide, tenants namespace-scoped

The platform's controllers are shared infrastructure and run cluster-wide in the
host cluster, exactly like every other Cozystack controller. They are not
per-tenant deployments.

Tenant permissions follow the scope of each type:

| Type | Scope | Tenant access |
| --- | --- | --- |
| `BlueprintInstance` | namespace | full within the tenant's own namespaces |
| `Workload` | namespace | read; written by `blueprint-controller` |
| `WorkloadRevision` | namespace | read |
| `NodeType`, `Blueprint`, `BindingProfile` | cluster | read-only |

Delivered as ClusterRoles aggregated into Cozystack's existing tenant roles, so
a tenant gains the platform types through the role it already has rather than
through a parallel mechanism.

The catalogue being cluster-scoped means it is common to the whole installation,
and per-tenant divergence is an RBAC-scoped subset of one catalogue rather than
a catalogue per tenant. This is the same limit recorded in `prior-art.md` §3 and
it must not be described as per-customer modularity.

### 4. Build-time plugin first, runtime loading only if version skew hurts

Two ways to load a plugin, and the cheaper one is not obviously wrong:

| | Build-time workspace package | Runtime Module Federation |
| --- | --- | --- |
| Upstream patch | a registry and a contribution API | the above plus a remote loader, a host/plugin API contract and a compatibility matrix |
| Releasing the platform UI | requires a `cozystack-ui` release | independent |
| Trust | reviewed in one repository | a plugin image runs in the operator's session with their privileges, so images need signing and an allowlist |
| Precedent | — | Red Hat Developer Hub loads Backstage plugins this way from OCI images |

**Start build-time**, with the contribution API designed so a remote loader can
be added behind it without changing plugins. The version-skew cost that argues
for runtime loading is a cost of *many* plugins from *many* authors; at one
plugin it is not yet real, and ADR-0020's rejection of Module Federation was
specifically about paying that cost before it is owed.

Revisit when there is a second plugin author, which is also when signing and
allowlisting stop being theoretical.

## Consequences

### Positive

- One panel, one login, one navigation. The operator does not learn a second
  tool.
- Authentication, the watch layer, form rendering and UI primitives are
  inherited rather than rebuilt.
- No backend to deploy, secure, scale or keep in sync with the CRDs.
- The plugin mechanism is useful to Cozystack independently of this platform,
  which makes it a contribution rather than a favour.

### Negative

- **The schedule now depends on upstream review.** The plugin mechanism must
  land in `cozystack-ui` before the platform has any surface at all. That is a
  dependency on other people's priorities and it is on the critical path.
- Until it lands, the platform is usable only through `kubectl`. This is
  acceptable for proving the architecture and unacceptable for demonstrating the
  product, so the two milestones must not be conflated.
- A build-time plugin ties the platform UI's release to the console's.
- Logs remain unsolved (§2).

## Open questions

1. The shape of the contribution API itself — this ADR states what it must do,
   not what it looks like. It should be designed in `cozystack-ui`, with its
   maintainers, not specified here.
2. Whether the plugin mechanism is acceptable upstream at all, and on what
   timeline. If it is not, the fallback is a separate panel with the costs in
   §1, and that decision should be taken early rather than after waiting.
3. Whether `cozystack-api` should instead learn to project CRD-backed types, so
   our resources appear through the existing discovery path. It would be a
   larger change to a more load-bearing component, but it would serve every
   future platform extension rather than only ours.

## Relationship to other ADRs

- **ADR-0020 §4** — its three-level UI extensibility model survives. Level 0,
  schema-generated forms, is what RJSF already does; level 1 is the plugin's own
  declarative views; level 2, runtime-loaded remotes, is deferred by §4 rather
  than rejected.
- **ADR-0021** — its "a UI of our own stops being optional" consequence is
  revised here, and the BFF it implied is dropped.
- **ADR-0022** — build and workload logs are question 5 there and are not solved
  by the no-backend model.
