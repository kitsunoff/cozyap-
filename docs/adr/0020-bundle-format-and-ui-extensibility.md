# ADR-0020: Bundle Format and UI Extensibility

## Status

Proposed, **amended 2026-09-07** — see the Amendments section at the end.
Two decisions below are superseded: the invention of a distribution format,
and the rejection of Module Federation UI plugins.

Depends on ADR-0018 (instance model) and ADR-0019 (Crossplane's role).
Reshapes ADR-0016 (Catalog and Versioning).

## Context

Requirement R6 asks for a catalog of applications and blueprints extended and
versioned by the Customer. Beyond the requirement, the operational driver is
divergence: each customer wants a different set of dependency types,
integrations, and — the expensive part — different things shown in the UI.

The natural first instinct is "a bundle is a Crossplane Configuration package".
It does not work.

**A Crossplane Configuration package may contain only XRDs and Compositions.**
This is a hard restriction of the package specification. Everything else a
bundle must carry has no place in it:

| Bundle content | Fits in a Crossplane Configuration? |
| --- | --- |
| XRD (typed dependency or application API) | yes |
| Composition (how it renders) | yes |
| Provider and function dependencies | yes, via `crossplane.yaml` |
| UI view descriptors | **no** |
| Default Grafana dashboards and alerts | **no** |
| RBAC presets and roles | **no** |
| Policies and quotas | **no** |
| Consumption-accounting metadata | **no** |

The community workaround — wrapping arbitrary objects in `provider-kubernetes`
`Object` resources inside a Composition — inverts the architecture: static
installation artefacts masquerade as composed resources of some composite.
Debugging is painful and versioning drifts.

## Decision

### 1. The bundle is our format; a Crossplane Configuration is one layer inside it

A bundle is a single versioned OCI artefact. It contains a Crossplane
Configuration package as one layer, plus the layers Crossplane cannot carry.

This must be settled before the first bundle ships. Starting from "a bundle is
a Configuration" and discovering the limit after two customers means changing
the format once bundles are already distributed across installations.

### 2. Bundle contents

- XRDs and Compositions, with provider and function dependencies — the
  Crossplane Configuration layer.
- **UI view descriptors** (§4).
- Default observability: Grafana dashboards, alert rules.
- RBAC presets and roles.
- Policies and quotas.
- **Consumption-accounting metadata** — how each resource this bundle
  introduces is counted for billing.
- Optionally, tier-3 Terraform modules for `provider-terraform`
  (ADR-0019 §5), so a customer-specific long-tail integration ships as a
  bundle rather than as a platform release.

### 3. Bundle contract — settle now, while bundles number zero

Changing the contract is free today and expensive after the third customer.
The minimum to define before any implementation:

1. **Format and boundaries** — what a bundle may and may not contain (the list
   in §2), semver versioning, OCI distribution.
2. **Compatibility** — a bundle declares which platform versions it supports,
   and what happens to installed bundles when the platform is upgraded.
3. **UI contract** — the view-descriptor schema. Even if v1 implements only
   level 0 (§4), the schema must be laid down now.
4. **Consumption-accounting metadata** — how a bundle's resources reach the
   billing pipeline. This is the field that gets forgotten and cannot be
   retrofitted into bundles already distributed.
5. **Trust** — a bundle can pull an arbitrary provider, meaning an arbitrary
   controller with rights in the cluster. In Mode B that is the customer's own
   risk. In Mode A it is **ours**, and requires signing plus an allowlist.

### 4. UI extensibility: three levels, commit to level 1

"We would like the UI to show this and that" is the most expensive requirement
in the whole design, because it has no floor. The level must be chosen
deliberately.

| Level | Mechanism | Verdict |
| --- | --- | --- |
| 0 | Forms generated from the XRD/CRD OpenAPI schema (the role `cozyvalues-gen` already plays in Cozystack) | Free. Covers R2 ("declare dependencies and runtime parameters") entirely. Covers nothing about displaying state. |
| 1 | **Declarative view descriptors shipped in the bundle** | **Chosen.** |
| 2 | Real frontend plugins (Module Federation, Backstage/Headlamp style) | Rejected for now. |

**Level 1** is a bundle object — a `ViewDefinition` or similar — describing
which tabs a resource has, which columns appear in lists, which status fields
are surfaced where, and which action buttons map to which operations. No code
runs in the browser; it is versioned with the bundle and reviewed as YAML. It
covers the large majority of real "show this" requests.

**Level 2 is rejected** because it is what makes Backstage expensive to
operate: every plugin needs a frontend build, version-locks to the application,
and a platform upgrade turns into recompiling every customer's plugins. For
"N enterprise customers with their own wishes" that does not scale — it is N
forks of the UI.

If level 1 provably proves insufficient, the next step is a build-versus-buy
evaluation of an existing extensible Kubernetes UI (Headlamp) rather than
writing a plugin runtime. That evaluation is explicitly deferred until level 1
has been shown to fall short.

## Consequences

### Positive

- Modularity is real: a new dependency type or integration for a customer is a
  bundle, not a platform release.
- UI divergence is absorbed declaratively, with no per-customer frontend build.
- Billing metadata travels with the thing being billed, from the first bundle.

### Negative

- We own a package format, its installer, its compatibility rules and its
  signing story — none of which Crossplane provides.
- Level 1 will not satisfy every UI request, and there will be pressure toward
  level 2. The rejection must be re-argued from evidence, not eroded case by
  case.
- In Mode A, bundle trust is a genuine security surface requiring signing and
  an allowlist before the first third-party bundle is installed.

## Open questions

- Bundle installer: a dedicated controller, or an extension of the platform
  operator?
- How blueprints (R6's "typical applications") relate to bundles — content of a
  bundle, or a separate, lighter artefact referencing bundles?
- Whether view descriptors are versioned independently of the bundle, for UI
  fixes without a bundle release.

## Relationship to other ADRs

- **ADR-0016** — catalog and versioning; reshaped by this format.
- **ADR-0018** — Mode A shares one bundle set per installation; Mode B is
  per-tenant. Bundle trust differs accordingly.
- **ADR-0019** — the bundle carries the dependency half of the boundary; the
  core is not bundled.

## Amendments (2026-09-07)

Arising from `docs/requirements/platform-vision.md` and
`docs/research/crossplane-as-platform-base.md`.

### A1 — Do not invent the distribution mechanism

§1 concluded that the bundle must be our own OCI artefact. The **content
model** conclusion stands and is unaffected: a Crossplane Configuration package
carries only XRDs and Compositions, so UI descriptors, dashboards, RBAC and
billing metadata need a carrier of our own.

The **distribution** conclusion is withdrawn. Cozystack v1.0 already ships a
`Package` / `PackageSource` model driven by `cozystack-operator`, with git and
OCI sources, install-time `flavors`, and a marketplace surface through
`ApplicationDefinition`. Constraint V2 names it as the vehicle, and it already
does what this ADR proposed to build.

Revised position: **our content model, Cozystack's `Package` as the first
distribution backend.** Constraint V7 then requires the content model to carry
no Cozystack assumptions, so a non-Cozystack installer can consume the same
artefact. The package contents list is in `docs/design/brick-model.md` §12.

Cozystack's `flavors` — several implementations of one package, selected at
install time — is the same idea as Radius recipes-per-environment and as the
per-target `implementations` field in `BrickDefinition`. Use it rather than
duplicating it.

### A2 — Module Federation UI plugins are accepted, as an opt-in tier

§4 rejected level 2 outright. Constraint V3 asks for it explicitly, and the
research shows the pattern is proven rather than speculative: Red Hat Developer
Hub loads Backstage frontend plugins as Module Federation remotes at runtime,
distributed as NPM packages, tarballs or OCI images.

Revised position, detailed in `docs/design/brick-model.md` §7:

- level 0 (schema-generated forms) and level 1 (declarative view descriptors)
  remain the **default**, and cover most bricks;
- level 2 is **available and opt-in**, declared by the brick, via a `UIPlugin`
  resource pointing at an image serving a Module Federation remote;
- the costs this ADR cited are accepted knowingly, not waved away: a
  plugin/host compatibility matrix, a reload to pick up changes, and — the one
  that matters most — a plugin runs in the operator's browser session with
  their privileges, so plugin images require the same signing and allowlisting
  as packages (§3.5);
- a plugin that fails to load must degrade to level 1, never break the page.

### A3 — Blueprint rendering semantics are still undecided

Not addressed by this ADR and now blocking: does a blueprint render once
(scaffold, then applications diverge) or continuously (a blueprint change
updates every application using it)? Requirement R6's "versioned by the
Customer" implies the second; the first is far simpler. Recorded as an open
question in `docs/design/brick-model.md` §15.
