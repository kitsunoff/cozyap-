# Research: Crossplane as a Developer Platform Base

## Status

Research note, 2026-09-07. Input to `docs/design/brick-model.md`. Refines
ADR-0019.

Question asked: how are developer platforms actually built on Crossplane, and
what does Crossplane cover versus what everyone ends up building themselves.

## The dominant industry pattern

Practically every published Crossplane-based IDP is the same three-layer stack,
usually called the golden triangle:

```text
Backstage        →  portal, catalog, self-service forms (Scaffolder templates)
      ↓ writes YAML into git
Argo CD / Flux   →  reconciles git into the cluster
      ↓ applies claims / composite resources
Crossplane       →  XRDs + Compositions + Providers create the real thing
```

The build order everyone recommends is Argo first, then Crossplane, then
Backstage, because each layer depends on the previous.

**What this tells us:** Crossplane occupies exactly one third of a developer
platform, and it is the third that is furthest from the developer. Nobody
ships Crossplane as the platform; everybody wraps it. The recurring warning in
the literature — *do not hand developers the XRD directly, the composition
exists to hide implementation and enforce standards* — is the same conclusion
ADR-0019 reached from a different direction.

## What Crossplane genuinely covers

| Capability | Coverage |
| --- | --- |
| Reconciling external systems (cloud, SaaS, VCS, DBaaS) | **Excellent** — the provider fleet is the product |
| Typed self-service APIs backed by policy | **Good** — XRD is a real, versioned API |
| Composing several resources into one abstraction | **Good** — compositions plus functions |
| Multi-tenancy inside one control plane | **Improved in v2** — namespaced composites and managed resources, namespaced `ProviderConfig` alongside `ClusterProviderConfig` |
| Generating providers from existing Terraform providers | **Excellent** — Upjet, powering the official AWS/Azure/GCP providers and 50+ community ones |
| Ordering and dependency resolution inside a composition | **Weak** — expressed implicitly through readiness; kro's approach is better and is now available *inside* Crossplane via `function-kro` |
| Imperative work (build, migrate, scaffold, generate) | **Absent** — functions are render steps, not tasks |
| Revisions and rollback of a workload | **Absent** |
| Developer-facing status and error surfacing | **Weak** — conditions on managed resources, no human-readable propagation |
| Presentation, catalog, forms | **Absent by design** |
| Consumption accounting | **Absent** |

The bottom five rows are the developer platform. That is the calibration:
**Crossplane saves you controllers, not a platform.**

## The composition function ecosystem is more interesting than expected

Compositions are no longer YAML patch-and-transform only. The pipeline accepts
functions written in several languages, distributed as OCI packages:

- `function-patch-and-transform` — the classic model, still the baseline
- `function-go-templating` — Helm-like templating
- `function-kcl` — KCL; fast and sandboxed, good for dynamic logic
- `function-python` — **full Python standard library inside a composition**
- `function-cue` — CUE
- `function-auto-ready`, `function-environment-configs` — plumbing
- `function-kro` — **kro's YAML+CEL graph model embedded as a Crossplane
  function**, with its graph builder, CEL evaluator and runtime; no separate
  kro installation

Two consequences for our design:

1. **`function-python` already exists.** The proposed `PythonEval` resource
   must therefore be justified against it, not assumed. See the split in
   `docs/design/brick-model.md`: pure computation belongs in `function-python`
   inside a composition; a `PythonEval`-style *resource* is only warranted for
   side-effecting work that needs its own lifecycle, status and retry
   semantics. Conflating the two produces a worse version of both.
2. **`function-kro` removes the "kro or Crossplane" fork.** kro's genuinely
   better idea — never declare ordering, derive the DAG from CEL references
   between resources — is available without adopting kro as a separate system.
   That reference-derived DAG is also exactly the right source of truth for a
   graph editor.

## kro — the model worth copying even without adopting it

A `ResourceGraphDefinition` declares a schema plus resource templates wired
with `${}` CEL expressions. **Ordering is never declared:** kro reads the
expressions, builds a DAG, and creates resources in the derived order.

This is the correct answer to a question the graph editor otherwise has to
answer badly. If the graph is *derived from typed references*, there is exactly
one source of truth and the editor cannot drift from reality. If the graph is
stored separately from the references, there are two, and they will disagree.

Its stated limits are also useful: kro orchestrates within a single cluster and
does not attempt cross-cluster management or **workflows requiring imperative
actions** — the same two gaps Crossplane has, which is why Kratix exists.

## Upbound Spaces — the multi-instance precedent

Upbound's commercial answer to running many control planes: a Space is a
self-managed slice hosting **50 or more Crossplane control planes on one
Kubernetes cluster**, with git-synced configurations pushed to a control plane
without managing a build pipeline.

Two things this validates for ADR-0018:

- Per-tenant control planes are an established production shape, not an
  extravagance — but note that Upbound achieves density by making a control
  plane much lighter than a full cluster. Our Mode B rests on Cozystack tenant
  clusters, which are heavier. If instance density ever becomes a cost problem,
  this is the direction to look.
- Git-sync of the control plane's own API definitions is the natural way to
  ship bundles, and matches Cozystack `PackageSource` pointing at git or OCI.

## Cozystack's own package model already exists — do not reinvent it

Cozystack v1.0 replaced HelmRelease bundle deployments with a declarative
`Package` / `PackageSource` model driven by `cozystack-operator`:

- `PackageSource` references a **git or OCI** repository;
- `Package` expresses the intent to install a particular package;
- sources are pulled and built into installation-ready artefacts via Flux's
  source machinery;
- packages support **flavors** — several implementations of the same package
  selectable at install time (the networking package ships `kubeovn-cilium`,
  `cilium`, `cilium-kilo`, `noop`);
- `CozystackResourceDefinition` was renamed `ApplicationDefinition`, and the
  platform chart registers apps through it so they appear in the dashboard and
  marketplace automatically.

**This directly corrects ADR-0020.** That ADR specified inventing an OCI bundle
format. Constraint V2 says the vehicle is the Cozystack `Package`, and the
vehicle already does what ADR-0020 was going to build: OCI and git sources,
declarative install, and a marketplace surface. The *content model* stays ours
— `Package` says nothing about bricks, ports, UI plugins or billing metadata —
but the **distribution mechanism must not be reinvented**.

The `flavors` mechanism is worth noting separately: it is the same idea as
Radius recipes-per-environment, already present in the platform we are
building on.

## Where every Crossplane platform ends up writing its own code

Consistently, across every published account:

1. A portal and catalog (usually Backstage, always customised).
2. Self-service forms and their validation (Scaffolder templates, hand-written).
3. Git-writing glue — the portal produces YAML, a pipeline commits it.
4. Status translation from conditions into something a developer understands.
5. Build and CI integration — entirely outside Crossplane.
6. Anything imperative or one-shot.
7. Cost and consumption reporting.

Items 4 through 7 are exactly requirement 5.14's hard parts. The research
supports ADR-0019's boundary rather than challenging it.

## Corrections to ADR-0019 arising from this research

1. **`function-kro` should be the default composition authoring model** for
   bricks implemented as compositions, rather than patch-and-transform or Go
   templating. Reference-derived ordering is both less error-prone and the
   right substrate for the graph editor.
2. **`function-python` exists**, which changes the justification required for a
   `PythonEval` resource — it must be argued as a *task* primitive, not as a
   templating primitive.
3. **Imperative workflows need a first-class primitive** next to compositions.
   Kratix proves the shape (ordered containers per lifecycle phase); Crossplane
   will not provide it.
4. **ADR-0020's bundle format must be rebased onto Cozystack `Package` /
   `PackageSource`** rather than inventing distribution.

## Sources

- [Crossplane composition functions in production](https://blog.crossplane.io/composition-functions-in-production/)
- [Crossplane v2 — what's new](https://docs.crossplane.io/latest/whats-new/)
- [function-kro announcement](https://blog.crossplane.io/function-kro-yaml-cel/)
- [function-kro repository](https://github.com/crossplane-contrib/function-kro)
- [function-python / function-kcl / function-go-templating](https://github.com/crossplane-contrib/function-go-templating)
- [kro overview](https://kro.run/docs/overview/)
- [Building platforms using kro for composition (CNCF)](https://www.cncf.io/blog/2025/12/15/building-platforms-using-kro-for-composition/)
- [Upbound Spaces](https://www.upbound.io/blog/announcing-spaces)
- [Cozystack v1.0 — package-based architecture](https://cozystack.io/blog/2026/03/cozystack-1-0-release/)
- [Building an IDP with Backstage, Argo CD and Crossplane](https://www.freecodecamp.org/news/how-to-build-an-internal-developer-platform-a-complete-guide-to-backstage-argocd-and-crossplane/)
