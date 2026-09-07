# Research: Internal Developer Platform Landscape

## Status

Research note, 2026-09-07. Input to `docs/design/brick-model.md`.

Question asked: how are competing developer platforms actually built, and what
should be taken from each.

## The landscape splits into four categories

Conflating them is the most common analysis error. They solve different
problems and a real platform usually needs one from each column.

| Category | What it owns | Examples |
| --- | --- | --- |
| **Portal / catalog** | Presentation, discovery, ownership, self-service forms | Backstage, Port, Cortex, OpsLevel |
| **Application model** | What "an application" *is* as a typed object | OAM/KubeVela, Radius, Score |
| **Orchestrator** | Turning a request into real resources | Humanitec, Kratix, Crossplane, kro, Massdriver |
| **Runtime PaaS** | The whole vertical, opinionated end to end | Heroku, Railway, Northflank, Qovery, Coolify |

Cozyap as specified by requirement 5.14 must cover **all four**. That is the
scope reality, and it is why "just take Crossplane" only ever answers the
third column.

## Backstage — the portal, and its plugin lesson

Entity model: `Component` (a piece of software in source control), `Resource`
(infrastructure a component needs), `System` (a set of components and resources
exposing APIs), plus `API`, `Domain`, `Group`, `User`. Entities are declared in
`catalog-info.yaml` files discovered from repositories, and joined by explicit
relations plus annotations.

**What to take:**

- The catalog is an **aggregate joined by references and annotations**, not a
  single object owning its parts. This is exactly constraint V5, and Backstage
  is the proof that it scales organisationally.
- Annotations as the extension seam: a plugin looks for its own annotation on
  an entity and contributes a tab or a card. Nothing central has to know about
  it. This is the cheapest possible capability model.

**What to avoid:**

- Classic Backstage plugins are compiled into the application. Every plugin
  needs a frontend build, version-locks to the host, and an upgrade means
  recompiling everyone's plugins.
- Red Hat Developer Hub fixed this with **dynamic plugins**: frontend bundles
  are Module Federation remotes, loaded at runtime, distributed as NPM
  packages, tarballs or **OCI images**, declared in a config file, with a
  restart to pick up changes. This is precedent for constraint V3 — the model
  the user proposes is proven, not speculative. The price RHDH pays is a
  plugin/host version compatibility matrix and an isolated `node_modules` per
  plugin at build time.

## OAM / KubeVela — the application model to learn from

`Application` = `Components` + `Traits` + `Policies` + `Workflow`.

- **Component** — a deployable unit: a Helm chart, a Kubernetes workload, a
  Terraform module, a cloud database.
- **Trait** — an attachable operational behaviour overlaid on a component:
  autoscaling, ingress, sidecars, rollout strategy, security policy.
- **Policy** and **Workflow** — cross-cutting rules and a DAG of delivery steps.

Definitions (`ComponentDefinition`, `TraitDefinition`) are written in CUE and
expanded by an operator into Kubernetes manifests.

**What to take:** the component/trait split. Not everything attached to an
application is a node in the dataflow graph. A Grafana link, a backup policy, an
autoscaler, a documentation generator — these are **traits on a brick**, not
bricks with ports. Modelling them as graph nodes would make every application
graph unreadable. This is the single most useful distinction found in the
research and it directly shapes the graph editor.

**What to avoid:** CUE as the mandatory authoring language for every definition.
It is a real barrier and KubeVela's adoption suffers for it.

## Radius — the closest existing thing to the target design

Microsoft's Radius is the nearest neighbour to constraint V1, and its three
concepts map almost one-to-one:

- **Resource Types** — a contract: the properties a developer sets, without
  implementation detail. Since 2025 platform engineers can define their **own**
  resource types, not only built-ins.
- **Recipes** — the implementation of a contract, as Terraform or Bicep,
  published to git or an OCI registry and registered per environment. **One
  resource type can have different recipes in different environments.**
- **Connections** — relationships declared in the application, which are
  captured at authoring time and give Radius its **application graph**.

**What to take:**

- **Type/implementation separation with per-environment implementation
  binding.** A `Database` brick means the same thing everywhere; in a Cozystack
  environment it resolves to a Cozystack PostgreSQL, in an AWS environment to
  RDS. This is precisely constraint V7, already solved by someone else, and it
  is a stronger formulation than "Crossplane composition" because the binding
  is per environment rather than per definition.
- **Connections as first-class authored objects.** The graph is not inferred
  from labels after the fact; the edge is a thing the author writes, and it is
  what drives credential injection.

## Humanitec + Score — the "developer writes less" end

`Score` is an open workload specification: a container plus a set of abstract
`resources` it depends on (`postgres`, `redis`, `s3`), with placeholder
references into those resources' outputs. The Platform Orchestrator matches
abstract resource *types* to concrete **Resource Definitions** supplied by
platform engineers, per context, and injects the resolved values via ConfigMap
or Secret.

**What to take:** the developer-facing artefact declares *types and
placeholders*, never implementations or credentials. Placeholder resolution at
deploy time is exactly requirement R3 ("no manual copying of passwords and
addresses"), and Score is the cleanest published expression of it. Score is also
worth supporting as an **import format** — it is a small spec and accepting it
costs little.

**What to avoid:** Score deliberately has no notion of anything except a
workload and its resources. It cannot express WordPress, a CI pipeline or a
CS2 server. Constraint V4 rules it out as the native model.

## Kratix — packaging an API together with its implementation

A `Promise` bundles four things:

- `api` — a CustomResourceDefinition, the interface users request through;
- `dependencies` — what must be installed on destinations first: operators,
  Helm charts, **even Crossplane compositions**;
- `workflows` — ordered chains of **containers** that run to fulfil a request,
  split by lifecycle (promise vs. resource) and phase (configure vs. delete);
- `destinationSelectors` — where the results are scheduled.

**What to take:**

- This is the direct precedent for constraints V2 and V3: a package that
  carries an API *plus* the operators, compositions and machinery needed to
  serve it. The Cozyap package model should be recognisably this shape.
- **Imperative container workflows as a first-class citizen next to
  declarative composition.** Kratix does not pretend everything is a render
  step. Crossplane does, and that is its weakest point for a developer
  platform (build, migrate, scaffold a repository, generate a pipeline).
- `destinationSelectors` is the multi-cluster answer, and it is why Kratix is
  routinely described as complementing rather than competing with Crossplane.

## Massdriver — the graph editor precedent

The one competitor whose primary interface is the canvas: bundles are dragged
onto a diagram, connected, configured through generated forms, and deployed.
Bundles package an IaC module (Terraform, OpenTofu, Helm, Bicep) with an input
schema, an **output contract**, policies and documentation.

The load-bearing idea: **the diagram is the infrastructure**, and connections
on the canvas are what automatically configure credentials and IAM bindings —
the edge is not decoration, it is the authorisation and injection mechanism.

**What to take:** bundles with an explicit output contract are what makes
type-checked edges possible at all. Without declared, typed outputs, a graph
editor degenerates into a drawing tool.

**What to avoid:** Massdriver's canvas is the only interface. Any real platform
needs the YAML and the form to be equivalent and round-trippable, because
non-trivial changes and review both happen in text.

## Port — the pragmatic catalog

Blueprints (custom entity definitions), entities, relations, self-service
actions with forms and approvals, and scorecards grading entities against
standards.

**What to take:** scorecards. Requirement 5.14 says nothing about them, but for
"the Customer curates the catalog" (R6) a mechanism that grades applications
against the catalog's standards is the natural governance counterpart, and it
is cheap once the aggregate model exists.

## Runtime PaaS (Heroku, Railway, Northflank, Coolify)

Relevant only as a UX benchmark for requirement 5.14's core loop: connect a
repository, detect the stack, build, attach a database, get a URL, roll back.
Their lesson is negative and specific: **every one of them is opinionated to
the point of being unable to host WordPress or a game server well.** The
generality demanded by constraint V4 is a deliberate departure from the PaaS
playbook and it will cost UX simplicity somewhere; the place to spend that cost
is the blueprint catalog, not the model.

## Synthesis — what Cozyap should steal

| Source | Idea taken |
| --- | --- |
| Radius | Type/implementation split with **per-environment** binding; connections as authored, graph-forming objects |
| OAM/KubeVela | Component versus **trait** — not everything attached to an app is a graph node |
| Backstage | Aggregate-by-reference-and-annotation; annotations as the zero-cost extension seam |
| RHDH | Module Federation frontend plugins distributed as OCI images, loaded at runtime |
| Kratix | A package carrying API *plus* operators, compositions and **imperative container workflows** |
| Massdriver | Typed output contracts on bundles, making type-checked edges possible; connections drive credential wiring |
| Score | Developer artefact carries types and placeholders only; worth accepting as an import format |
| Port | Scorecards as the governance counterpart to a curated catalog |

## Sources

- [Backstage system model](https://backstage.io/docs/features/software-catalog/system-model/)
- [Backstage architecture overview](https://backstage.io/docs/overview/architecture-overview/)
- [Red Hat Developer Hub dynamic plugins](https://developers.redhat.com/articles/2025/11/20/how-build-your-dynamic-plug-ins-developer-hub)
- [KubeVela core concepts](https://kubevela.io/docs/getting-started/core-concept/)
- [OAM specification](https://oam.dev/)
- [Radius Resource Types](https://opensource.microsoft.com/blog/2025/06/30/expanding-platform-engineering-capabilities-with-radius-resource-types/)
- [Radius project](https://github.com/radius-project/radius)
- [Humanitec Score deployment](https://developer.humanitec.com/platform-orchestrator/docs/deploy/score/)
- [Humanitec resources overview](https://developer.humanitec.com/app-humanitec-io/docs/platform-orchestrator/resources/overview/)
- [Kratix Promise reference](https://docs.kratix.io/main/reference/promises/intro)
- [Kratix and Crossplane](https://docs.kratix.io/main/how-kratix-complements/crossplane)
- [Massdriver docs](https://docs.massdriver.cloud/)
- [Port product overview](https://docs.port.io/)
