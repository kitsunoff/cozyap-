# Design: The Graph Editor

## Status

**Deferred (2026-09-14.)** No requirement asks for the editor, and it is
recorded as D1 in `docs/decisions.md`. The model still derives topology from
references, so the editor stays possible. Kept for historical context.

Design draft, 2026-09-07. Companion to `docs/design/brick-model.md`.

The editor is not a visualisation bolted on at the end. It is the interface
that makes the brick model comprehensible, and several decisions in the brick
model exist because of it. Getting it wrong makes the whole model feel like
YAML with extra steps.

## 1. The two products sharing one canvas

They are routinely confused, and they have different rules.

| | **Blueprint editor** | **Application view** |
| --- | --- | --- |
| Edits | `ApplicationBlueprint` — a template | Live `BrickInstance` objects |
| Audience | Platform engineer, package author, Customer curating the catalog (R6) | Developer |
| Status | None — nothing is running | Live conditions on every node |
| Parameters | Declared as inputs, with defaults | Concrete values |
| Output | A versioned artefact in a package | A change to a running system |
| Default mode | Full edit | **Read-first**, edit behind an explicit affordance |

The same canvas component serves both; the difference is the data source and
the permission model. Shipping only one of them is a mistake in either
direction: the blueprint editor without the application view gives developers
nothing, and the application view without the blueprint editor means the
Customer cannot curate the catalog.

## 2. The load-bearing rule: the graph is derived

**Layout is stored. Topology is never stored.**

Topology comes from `spec.connections` on brick instances, exactly as kro
derives its DAG from CEL references. There is one source of truth, and the
editor cannot show something the cluster does not have.

The alternative — a `Graph` object holding nodes and edges alongside the
instances — creates two sources of truth that drift the first time anyone uses
`kubectl`. Every platform in the research that made the diagram authoritative
(Massdriver) also made the diagram the *only* interface, which constraint V5
and ordinary code review both rule out for us.

Layout is a different matter and is genuinely presentational:

- position lives in an annotation on the instance:
  `ui.cozyap.io/position: "480,220"`;
- when absent, layout is computed;
- writes are debounced (drag produces one write on drop, not per frame);
- a missing or stale position is never an error.

Storing position on the instance rather than in a side object means it travels
with the object, survives export, and needs no garbage collection. The cost is
that two people dragging the same node concurrently conflict — acceptable, and
noted in §9.

## 3. Auto-layout

Nodes have **typed ports on specific sides**, which rules out the naive
choices. Dagre has no port model; a force-directed layout produces something
that looks organic and reads terribly for a pipeline.

**ELK's `layered` algorithm** with port constraints is the right default:
left-to-right, ports honoured, edge crossings minimised. The natural flow of
these graphs — source → build → image → workload → route — is inherently
layered, and the layout should make that obvious at a glance.

Rules:

- Auto-layout runs when an instance has no stored position.
- "Re-layout" is an explicit user action, never automatic on data change —
  a graph that rearranges itself while being watched is disorienting.
- New nodes appear near their connection source, not at the origin.

## 4. What is and is not a node

From `brick-model.md` §6, and this is where the editor's readability is won or
lost:

- **Bricks are nodes.** They have ports and other bricks consume them.
- **Traits are badges on a node.** Autoscaling, backups, network policy,
  sidecars — small icons in the node's footer, expandable in the inspector.
- **Facets are not on the canvas at all.** Links, panels, actions and metrics
  belong to the application view's side panels, not to the graph.

Without this rule a six-brick Java service renders as twenty-plus nodes and
nobody reads it twice.

**Grouping** handles the remaining density: a brick whose implementation
composes many underlying resources is one node. Expanding it is a drill-down
into a read-only sub-graph — useful for debugging, never editable, because the
composition is the package author's business.

## 5. Edges: type checking is the feature

Dragging from an output port:

1. Immediately dims every input port whose type is not assignable, so the valid
   drop targets are the only lit ones. Type errors become unreachable rather
   than reported.
2. On drop, writes `spec.connections` on the **consumer**, with default
   injection settings derived from the port type (`sql` → `DATABASE_` env
   prefix plus a `/bindings/db` file mount).
3. Rejects an edge that would create a cycle, with the offending path
   highlighted rather than a message saying "cycle detected".

The inspector then exposes what the edge actually does — the three jobs from
`brick-model.md` §4: data, injection, authorisation. **The authorisation part
must be visible.** A user connecting a bucket to a workload is granting the
workload access to that bucket, and a UI that presents this as merely drawing a
line is training people to grant access carelessly.

## 6. Editing a live system: the GitOps problem

**This is the decision most likely to be got wrong, and it must be made before
implementation.**

If applications are reconciled from git by Flux or Argo — which is the norm in
every Crossplane platform in the research, and is how Cozystack itself operates
— then an editor that applies changes directly to the cluster is writing into a
system that will revert it. The user drags a node, it works for thirty seconds,
and then it disappears. This is worse than not having an editor.

Three modes, and the platform must support at least the first two:

| Mode | Editor writes | Suits |
| --- | --- | --- |
| **Direct apply** | Straight to the API server | Instance-owned applications with no GitOps upstream |
| **Propose** | Opens a merge request against the application's repository | GitOps-managed applications; the norm for anything production |
| **Ephemeral** | A dry-run overlay, never persisted | Previewing and exploring |

The mode is a property of the `Application` (does it have a git upstream?), not
a user preference, and the editor must **show which mode it is in at all
times**. In propose mode the primary button says "Propose change", not "Save".
Backstage's Scaffolder and Massdriver both landed on the same conclusion from
the opposite direction.

## 7. Dry-run before apply

Every mutation shows what will change before it happens: resources created,
updated, deleted, and the **credential grants a new connection implies**. The
underlying mechanism already exists — `crossplane render` for composition-backed
bricks, a controller dry-run for native ones.

For destructive changes (removing a database brick, changing a target) the
dialog must name what is lost and require typing the name. The audience does
not know Kubernetes; the confirmation carries the whole weight of preventing
data loss.

## 8. Technology

**React Flow (xyflow).** Chosen because every node is a real React component,
which is what makes §10 possible; the alternatives are all weaker on custom
node content. It handles pan, zoom, multi-select, keyboard, minimap and custom
handles out of the box, and re-renders only changed nodes.

- **ELK.js** for layout, in a worker (layout on the main thread janks at ~50
  nodes).
- **`onlyRenderVisibleElements`** beyond ~150 nodes. Below that, unnecessary.
- Realistic budget: applications are 5–30 bricks. Grouping and search cover the
  outliers. Do not engineer for a thousand-node canvas that will not exist.

## 9. Deliberate non-goals for v1

- **Real-time collaboration.** Storing layout in annotations makes concurrent
  drags conflict; that is an accepted cost, not a bug to fix now.
- **Free-form drawing** — notes, arrows, boxes that mean nothing. Every element
  on the canvas must correspond to a real object, or the canvas stops being
  trustworthy.
- **Editing compositions.** Drilling into a brick is read-only. Composition
  authoring is a package author's job in YAML, with tests.

## 10. Where the editor and the package system meet

The `UIPlugin` scope `graphNode` from `brick-model.md` §7 lets a package supply
the React component that renders its own node. This is only possible because
nodes are React components and plugins arrive as Module Federation remotes.

It is genuinely useful — a CS2 node showing live player count, a build node
showing a progress bar, a database node showing storage headroom — and it is
also the single easiest way to make the canvas slow or broken. Constraints:

- a node plugin renders **inside a fixed-size frame** and cannot alter layout;
- it gets read-only props (the instance's status), never a client;
- a render error falls back to the default node with a small badge, never an
  empty canvas;
- a node that fails to load in time renders the default and retries silently.

## 11. Build order

1. **Read-only application view** with auto-layout, live status and the
   inspector. This alone justifies the model to a developer and is the cheapest
   thing that makes the platform feel real.
2. **Blueprint editor**, full edit, no live status. The Customer needs this for
   requirement R6, and it is easier than editing a live system because nothing
   is running.
3. **Application editing**, propose mode first, direct apply second.
4. **Level-2 node plugins.** Last, once the default node has proven what the
   frame should be.

Shipping 1 and 2 without 3 is a coherent product. Shipping 3 before 2 is not.
