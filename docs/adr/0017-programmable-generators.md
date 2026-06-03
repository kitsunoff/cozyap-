# ADR-0017: Programmable Generators and Higher-Order Atoms

## Status

Proposed. Follows ADR-0008 (Reactive Reconcile Model) and ADR-0009 (Atom
Contract). Refines the composition rules that ADR-0013 (Application Graph)
will formalize; this ADR owns the **dynamic fan-out** and **higher-order
composition** concerns specifically.

Execution semantics depend on the still-open ADR-0012 decision (Temporal vs
Pods/stdin-stdout); this ADR is written **executor-neutral** and notes where
the two options differ.

## Context

The reactive reconcile model (ADR-0008) and the atom contract (ADR-0009)
describe a **static** graph: an `ApplicationTemplate` is a fixed set of atom
nodes wired by typed ports, instantiated as a single `Application`. Three
recurring needs do not fit that static shape:

1. **Dynamic fan-out — "N instances from data".** One open pull request →
   one preview environment; one tenant row → one Application; one region →
   one deployment. The count is not known at authoring time; it is derived
   from data that arrives at runtime. The static graph has no primitive for
   "map a list onto N instances". A preview-per-PR feature today would need a
   bespoke controller sitting outside the model (see the preview-environment
   analysis in the project notes).

2. **Cross-Application sharing.** ADR-0008 left open whether
   `App-B.Container` may reference `App-A.Postgres.credentials`. Direct edges
   between independent Applications break tenant isolation and the graph's
   ownership model. But the underlying need — one component provisions
   things and feeds values into others — is real.

3. **Application as a composable unit.** The template→instance split stops at
   the Application boundary. There is no way for a graph to *produce* another
   Application, which blocks both fan-out (1) and structured sharing (2).

The unifying request from design discussion: a **programmable generator** —
data arrives, an author-supplied mapping turns it into a set of parameter
bundles, and each bundle instantiates an Application (or sub-graph). The
mapping must be programmable, not limited to a fixed list of generator kinds.

The tempting but wrong framing is "support cycles in the graph". Fan-out
(`map` over a list) is conflated with feedback (`A.out → … → A.in`). They are
different primitives. Feedback reintroduces exactly the reconcile-storm /
divergence problem the acyclic invariant (project notes §8 "Rate Limiting and
Cycles") was built to prevent. This ADR provides fan-out **without** relaxing
the acyclic invariant.

## Decision

### 1. Higher-order atom: `CreateApplication`

Introduce a platform-provided atom whose reconcile creates and continuously
reconciles a **child `Application`**, passing values into it:

- **inputs/params**: a `templateRef` (which `ApplicationTemplate` + version to
  instantiate) and a `values` bundle (the child's `spec.values`), plus a
  required **merge key** (stable identity, see §3).
- **outputs**: optionally re-exports selected child outputs (typed ports) so a
  parent graph can wire them — this is the sanctioned answer to cross-app
  sharing (need 2): the parent owns the child, so there is no cross-tenant
  edge.
- **ownership**: the child carries an `ownerReference` to the creating
  `AtomInstance`/`Application`. Teardown cascades; a parent finalizer holds
  until children are gone.
- **reconcile semantics**: unchanged from ADR-0008 — create / update / delete
  of the child are all "make actual match desired". Removing a `values` bundle
  removes the child.

This makes `Application` a composable unit: a graph can produce graphs.

### 2. Programmable generator + fan-out boundary

A **generator** is the source of the parameter bundles a fan-out consumes:

```
[data source]
      │
      ▼
┌──────────────────────┐   programmable mapping:  data -> []paramBundle
│  Generator           │   each bundle carries a STABLE merge key
└──────────────────────┘
      │  collection (boundary value — see §4)
      ▼
┌──────────────────────┐   for each bundle:
│  Fan-out boundary    │     CreateApplication(templateRef, values = bundle)
└──────────────────────┘
      │
      ▼
   N × Application      (each is an ordinary static graph per ADR-0009;
                         an instance does not know it is one of N)
```

The generator's mapping is **author-supplied logic**, in two tiers:

- **Declarative tier** — a JQ/template expression `data -> []paramBundle`.
  Pure, deterministic, sufficient for list/PR/git-style generators. Preferred
  default.
- **Code tier** — an arbitrary-language atom: JSON in (the data) → JSON array
  of parameter bundles out. For logic JQ cannot express. **Reuses the atom
  execution layer (ADR-0012)** — a generator is just another atom invocation,
  so no new execution mechanism is introduced. Under Pods/stdin-stdout it is
  a trivial `read JSON → write JSON`; under Temporal it is a workflow bound by
  the same determinism constraints as any atom.

### 3. Identity is mandatory (merge key)

Every emitted bundle MUST carry a **stable key** that identifies its
instance across reconciles (PR → PR number; history record → record id;
region → region name). The fan-out maps `key → instance` deterministically:

- new key → create instance,
- existing key, changed bundle → update instance (reconcile),
- key absent from the current emission → instance is a teardown candidate
  (subject to the lifecycle policy in §5).

Without a stable key, every reconcile churns create/destroy and tears down
the wrong instances. Assigning the key is the **author's responsibility** and
is the primary footgun of the programmable tier.

### 4. Collection is a boundary type, not a graph port type

The list produced by a generator is consumed **only** by a fan-out boundary.
It is NOT added to the typed port vocabulary (ADR-0009, 13 types). Ports
inside an instance graph remain scalar/single-reference; an instance is
unaware of the collection. Rationale: adding `list<T>` as a general port type
would spread collections across every graph and defeat the static
type-checking the port system exists to provide.

### 5. Generator contract — guardrails (normative)

A generator declaration MUST specify, and the platform MUST enforce:

1. **Determinism** — the mapping is pure with respect to its input data.
   External reads, if any, happen inside reconcile and are absorbed by the
   merge-key discipline.
2. **Cardinality cap** — a hard upper bound on emitted instances. Bounded
   sources (open PRs) are fine; unbounded/append-only sources (history,
   events) MUST also declare a **window/filter** (last N, by status). The cap
   is a backstop against fleet explosion.
3. **Lifecycle policy on disappearance** — what "a key left the set" means is
   explicit per generator: `teardown` (PR closed → destroy preview) or
   `retain` (history record aged out of the window → keep the instance).
   There is no implicit default.
4. **Blast radius controls** — a generator that creates Applications creates
   arbitrary workloads via author logic. Required: execution sandbox (no
   ambient network/host access beyond declared `executor.requires`, mirroring
   the Cortex-style scaffolder sandbox), a per-tenant quota on child count, a
   whitelist of instantiable `templateRef`s, and a **nesting depth cap**
   (Application creating Application creating … is bounded).

### 6. The acyclic invariant stays

Fan-out (`map`) is not a cycle. The data graph remains a DAG; cycle detection
at builder time (project notes §8) is unchanged. Feedback edges
(`A.out → … → A.in`) remain **forbidden**. If a genuine feedback need appears
later (e.g. an autoscaler reading its own metric), it requires a separate
contract — fixpoint iteration with an oscillation detector and an iteration
cap — and is explicitly **out of scope** here. Generators are not a backdoor
to cycles.

## Prior art

This pattern is established; two systems are near-exact analogs and both
deliberately stay acyclic:

- **Crossplane Composition Functions** — functions packaged as OCI
  containers, in any language, arranged in a **pipeline** where each
  accumulates desired state (reads the composite's observed state, adds
  composed resources, returns a `RunFunctionResponse`). This is literally a
  programmable generator of resources, and Compositions are nestable. The
  model is a DAG; it has no cycles.
- **Argo CD ApplicationSet** — `List` / `Git` / **`Pull Request`** /
  **`Plugin`** / **`Matrix`** generators. The **Plugin** generator is the
  programmable case (an external service returns arbitrary-typed parameters);
  **Matrix** composes generators; the **Pull Request** generator is exactly
  the preview-per-PR case. No cycles.

Both validate the decisive split: **programmability — yes; feedback cycles —
intentionally no.**

## Consequences

### Positive

- Dynamic fan-out ("N from data") is expressed **inside** the model, not by a
  bespoke side controller.
- Cross-Application sharing has a sanctioned, ownership-respecting form
  (parent creates and wires children) instead of isolation-breaking edges.
- `Application` becomes composable; higher-level templates can be built from
  whole sub-Applications.
- Reuses the existing execution layer (ADR-0012) — no new runtime concept.
- Stays acyclic — the reconcile-storm guarantees of ADR-0008 are preserved.

### Negative

- Programmable generators raise the authoring bar again: a correct, stable
  **merge key** and a declared lifecycle policy are now author obligations.
- A new conceptual surface (generator, fan-out boundary, merge key,
  cardinality/lifecycle policy, depth cap) to document and to render in the
  builder UI.
- The code tier widens the blast-radius surface; sandboxing and quotas become
  load-bearing rather than nice-to-have.

### Risks

- **Fleet explosion** from an unbounded or buggy generator. Mitigation: hard
  cardinality cap + mandatory window for unbounded sources (§5.2). The cap
  must be enforced at the platform, not trusted to the author.
- **Churn / thrash** from an unstable or non-deterministic key. Mitigation:
  determinism requirement (§5.1); catalog linter should run a generator twice
  on identical input and assert identical key sets.
- **Privilege escalation** via a generator instantiating powerful templates.
  Mitigation: templateRef whitelist + per-tenant quota + sandbox (§5.4).
- **Determinism under Temporal** (if ADR-0012 picks Temporal): a code-tier
  generator inherits workflow determinism constraints; the linter burden from
  ADR-0010/0012 extends to generators.

## Relationship to other ADRs

- **ADR-0008** — preserves the acyclic, reconcile-only model; fan-out is a new
  composition primitive, not a new lifecycle.
- **ADR-0009** — does not add a port type; the collection is a boundary value.
  `CreateApplication` is a platform atom that conforms to the atom contract.
- **ADR-0012** — generators reuse whichever execution layer is chosen; this
  ADR adds no new executor.
- **ADR-0013 (Application Graph, pending)** — should incorporate the
  generator/fan-out boundary and the `CreateApplication` node into the formal
  composition + cycle-detection rules.

## Open questions

- **Where generator data comes from** — a typed input port feeding the
  generator atom, a watched external source inside the generator's reconcile,
  or a separate `GeneratorSource` resource? Affects change-detection
  granularity for re-fan-out.
- **Source of truth for fanned-out instances** — child `Application` CRs in
  the cluster vs Git-driven (ApplicationSet-style). Preview-per-PR leans
  toward Git-driven; ties into the unresolved spec-source question in
  ADR-0008 / project notes §16.
- **Default cardinality cap and depth cap values** — must be set conservatively
  and be tenant-overridable within operator-set limits.
