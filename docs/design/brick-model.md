# Design: The Brick Model

## Status

Design draft, 2026-09-07. Synthesises requirement 5.14
(`docs/requirements/5.14-developer-platform.md`), the product owner
constraints (`docs/requirements/platform-vision.md`) and the two research
notes in `docs/research/`.

Refines ADR-0019 and corrects ADR-0020. Not yet an ADR — this document exists
to be argued with.

## 1. The central problem this model solves

Constraint V1 says an application is a set of bricks. Constraint V4 says the
same model must express a Java service, a WordPress install, and a CS2 game
server. Requirement 5.14 says the user does not know Kubernetes and must be
given good status, rollback and a form to fill in.

These pull in opposite directions. A model general enough for a game server is
usually too abstract to give a Java developer a good form; a model tuned to
twelve-factor apps cannot host WordPress. Every runtime PaaS in the research
resolved this by giving up generality. We cannot.

The resolution used throughout this document:

> **Uniform contract, plural implementations, curated presentation.**
>
> Every brick honours the same contract — typed ports, readiness, status,
> billing metadata, facets. *How* a brick is implemented varies per brick type
> and is invisible to the graph. *What the user sees* is a blueprint from the
> catalog, not the raw model.

This also resolves the tension between V1 and ADR-0019 flagged in the vision
document. ADR-0019's boundary — "core in an own controller, dependencies in
Crossplane" — survives, but as a **property of particular brick types**, not
as a split in the model. A `Build` brick is implemented by a platform-native
controller because it needs revisions and log surfacing; a `PostgresDatabase`
brick is implemented by a composition because it is pure provisioning. Both are
bricks.

## 2. The five layers

```text
┌──────────────────────────────────────────────────────────────┐
│ 5. Presentation   Application view = aggregate + facets       │
│                   (links, panels, actions, health)            │
├──────────────────────────────────────────────────────────────┤
│ 4. Aggregate      Application: label/ref-scoped set of        │
│                   brick instances + Releases (revisions)      │
├──────────────────────────────────────────────────────────────┤
│ 3. Instance       BrickInstance CRs, wired by typed           │
│                   Connections. The graph is *derived*.        │
├──────────────────────────────────────────────────────────────┤
│ 2. Type           BrickDefinition: ports, schema, traits,     │
│                   facets, billing, implementation binding     │
├──────────────────────────────────────────────────────────────┤
│ 1. Implementation composition │ native controller │ task      │
│                   pipeline │ operator CR │ recipe per target  │
└──────────────────────────────────────────────────────────────┘
```

Layer 1 is plural on purpose. Layers 2–5 are singular and are what the whole
product is built on.

## 3. Layer 2 — `BrickDefinition`

The type. Shipped in a package, versioned with semver, cluster-scoped.

```yaml
apiVersion: bricks.cozyap.io/v1alpha1
kind: BrickDefinition
metadata:
  name: postgres-database.v1
spec:
  # --- identity -------------------------------------------------------
  type: core.cozyap.io/PostgresDatabase
  version: 1.4.2

  # --- the contract ---------------------------------------------------
  parameters:                       # OpenAPI schema → generated form
    type: object
    properties:
      size:    { type: string, enum: [small, medium, large] }
      version: { type: string, enum: ["15", "16", "17"] }
    required: [size]

  ports:
    in: []
    out:
      - name: connection
        type: core.cozyap.io/sql
        # what the platform injects downstream, and under which keys

  # --- how it becomes real, per target class -------------------------
  implementations:
    - match: { targetClass: cozystack }
      kind: Composition
      ref:  { name: xpostgres-cozystack, version: "1.4.x" }
    - match: { targetClass: aws }
      kind: Composition
      ref:  { name: xpostgres-rds, version: "1.4.x" }
    - match: { targetClass: any }         # fallback
      kind: TaskPipeline
      ref:  { name: postgres-generic }

  # --- what it needs present in the target ---------------------------
  targetRequirements:
    - package: cnpg-operator
      version: ">=1.24"
      scope: dataplane            # ref-counted, see §11

  # --- operational overlays this type accepts ------------------------
  acceptsTraits: [backup, monitoring, resource-limits]

  # --- what it contributes to the application view -------------------
  facets:
    - kind: link
      name: Grafana
      urlTemplate: "{{ .platform.grafana }}/d/pg/{{ .instance.name }}"
    - kind: action
      name: "Manual backup"
      task: postgres-backup

  # --- how it is counted --------------------------------------------
  billing:
    unit: instance-hour
    dimensions: [size, storageGiB]

  # --- UI beyond the generated form (optional) -----------------------
  ui:
    descriptor: { configMapRef: postgres-view }     # level 1, default
    plugin:     { image: registry/…/pg-ui:1.4.2 }   # level 2, opt-in
```

Three fields carry most of the design weight:

- **`implementations` matched per target class** — Radius's recipe idea. One
  contract, several realisations, chosen by *where it lands* rather than by who
  authored it. This is constraint V7 in one field.
- **`ports`** — the typed output contract. Without it, edges cannot be
  type-checked and the graph editor is a drawing tool (Massdriver's lesson).
- **`facets`** — how a brick contributes to the presentation layer without the
  UI knowing the brick exists (Backstage's annotation seam, made explicit).

### Implementation kinds

| Kind | Runs as | Use for | Revisions | Imperative |
| --- | --- | --- | --- | --- |
| `Composition` | Crossplane XR pipeline, `function-kro` by default | Provisioning: databases, buckets, DNS, repos | no | no |
| `Native` | Platform controller (controller-runtime) | Build, workload, release — anything needing revisions and log surfacing | yes | yes |
| `TaskPipeline` | Ordered containers, Kratix-shaped | Scaffolding, migration, generation, one-shot work | via Release | yes |
| `OperatorResource` | A CR of an operator shipped in the package | Off-the-shelf software with a good operator | operator's | operator's |
| `External` | Reference to something not managed here | Adopting pre-existing resources into an application | no | no |

`External` matters more than it looks: constraint V5 requires that resources
joined by annotation or label — including ones the platform did not create —
appear in the application. `External` makes adoption a first-class brick rather
than a special case in the UI.

### How a `Composition` implementation actually executes

The `implementations` field above is a binding, not a call. Crossplane's engine
is driven by **composite resources**: an XR is a CRD generated from an XRD, and
it selects its Composition through `compositionRef` / `compositionRevisionRef`.
For a Composition to run, something must create an XR.

That something is the **brick controller**. It is a translator, not a
composition engine:

```text
BrickInstance                      ← what the user and the editor see
      │  brick-controller: resolve BrickDefinition + Target.class,
      │  pick implementation, map parameters and resolved inputs
      ▼
XPostgresCozystack  (an XR)        ← the executable object, XRD from the package
      │  Crossplane: pinned CompositionRevision → function pipeline
      ▼
composed managed resources          ← provider-kubernetes Object, or a Cozystack CR
      ▼
Cozystack operator → CNPG → pods    ← the real database
      │
      │  connection secret + Ready condition
      ▼
brick-controller reads the XR back → BrickInstance.status.outputs / conditions / usage
```

`BrickInstance` is therefore a **facade** over an implementation object, and the
binding needs more than a reference:

```yaml
implementations:
  - match: { targetClass: cozystack }
    kind: Composition
    composite:
      apiVersion: cozystack.platform.io/v1alpha1
      kind: XPostgresCozystack              # XRD ships in the same package
      compositionRevisionRef: xpostgres-cozystack-a1b2c3
    parameterMapping: passthrough           # brick parameter schema == XR spec
    inputMapping: {}                        # resolved connections → XR spec fields
    outputMapping:
      connection:                           # our port name
        from: connectionSecret              # the XR's writeConnectionSecretToRef
        keys: { host: host, port: port, database: dbname,
                user: username, password: password }
    readiness:
      from: conditions[Ready]
```

Four decisions live in that block:

1. **`composite`** names the XR kind to create. The XRD arrives in the same
   package; if it is absent the brick is marked unavailable and the blueprint
   linter rejects anything using it. Installation-time validation, not a
   runtime surprise.
2. **`parameterMapping: passthrough`** — by default the brick's parameter
   schema *is* the XR's spec schema, and the package author writes the XRD to
   match. Cheaper than introducing a second templating language; an explicit
   map exists only for the cases that cannot line up.
3. **`outputMapping` is mandatory.** Connection-secret keys are the composition
   author's choice, while a port type has fixed keys. Without the mapping,
   typed ports do not actually type-check anything.
4. **`compositionRevisionRef`, never `compositionRef`.** A floating reference
   means upgrading a package silently changes the behaviour of every existing
   instance. Pinning a revision plus an explicit upgrade action is what makes
   requirement R6's "versioned by the Customer" real rather than nominal.

#### Worked trace

1. A `BrickInstance` of type `PostgresDatabase` is created with
   `targetRef: prod-eu` and `parameters: {size: medium, version: "17"}`.
2. The brick controller loads the `BrickDefinition`, loads the `Target`, reads
   `class: cozystack`, matches the implementation, satisfies
   `targetRequirements` (installing `cnpg-operator` and incrementing its
   refcount if absent), and creates the XR with an owner reference back to the
   `BrickInstance`.
3. Crossplane reconciles the XR against the pinned composition revision; the
   function pipeline renders composed resources; providers create them.
4. Crossplane writes the connection secret and sets the XR `Ready`.
5. The brick controller watches the XR and fills the facade back in:
   `status.outputs.connection` (secret reference plus non-secret fields inline
   for the UI), human-readable conditions, and `status.usage` for metering.
6. A downstream `Workload` with a connection from `api-db.connection` reads
   `status.outputs.connection.secretRef` and projects it into the pod as
   `DATABASE_*` variables and a `/bindings/db` mount.

Deletion mirrors this: a finalizer holds the `BrickInstance` until the XR is
gone.

`Native` and `TaskPipeline` implementations have no XR at all — the controller
does the work, or creates Jobs. Uniformity exists at the `BrickInstance` level
only, which is the entire point of the facade.

#### The cost of the facade, stated plainly

Two objects per brick, two reconcile loops, status that must be propagated,
errors that must be translated, and debugging that goes through an extra hop.

Paid for three things:

- **Constraint V7.** Changing target class changes the XR *kind*
  (`XPostgresCozystack` → `XPostgresRds`). Without the facade the application's
  own object would change; with it, the `BrickInstance` does not change at all.
- **Uniformity.** The editor, the aggregate, metering and facets deal with one
  kind rather than N unrelated XR kinds.
- **Non-composition bricks.** `Native` and `TaskPipeline` bricks have no XR;
  the facade is the only thing that makes them peers of composition-backed ones.

The alternative worth knowing: **make the brick type *be* the XRD**, with
`BrickDefinition` reduced to metadata *about* an existing XRD (ports, facets,
billing, ui). Half the objects, and `crossplane beta trace` works directly. It
is rejected because it breaks V7 — a per-target implementation swap would
change the kind the user authored — and because every `Native` brick would
still need a CRD of its own.

## 4. Layer 3 — `BrickInstance` and derived graphs

One CR per brick instance, namespaced. **The graph is never stored; it is
derived from typed references between instances.** This is kro's rule and it is
the single most important structural decision in this document: two sources of
truth for a graph will always drift.

```yaml
apiVersion: bricks.cozyap.io/v1alpha1
kind: BrickInstance
metadata:
  name: api-db
  namespace: team-payments
  labels:
    app.cozyap.io/application: payments-api
  annotations:
    ui.cozyap.io/position: "480,220"
spec:
  type: core.cozyap.io/PostgresDatabase
  targetRef: { name: prod-eu }
  parameters:
    size: medium
    version: "17"
  traits:
    - kind: backup
      params: { schedule: "0 2 * * *", retention: 14d }
status:
  phase: Ready
  conditions: [...]
  outputs:
    connection:
      secretRef: { name: api-db-conn }
      # non-secret fields inline for the UI: host, port, database
  facets:
    - { kind: link, name: Grafana, url: "https://…" }
  usage:
    - { unit: instance-hour, dimensions: { size: medium, storageGiB: 50 } }
```

### Connections

An edge is an authored object, not an inference (Radius's lesson). It lives on
the consumer:

```yaml
spec:
  connections:
    - name: db
      from: { instance: api-db, port: connection }
      inject:
        env:   { prefix: DATABASE_ }        # DATABASE_HOST, DATABASE_PASSWORD…
        files: { path: /bindings/db }       # Service Binding shape
```

A connection does three things, and the third is the one people forget:

1. **Data flow** — outputs become inputs.
2. **Injection** — env vars and mounted files, resolved at deploy time, never
   copied by a human (requirement R3).
3. **Authorisation** — creating the connection is what causes a database user,
   a bucket policy, or an IAM binding to be created and scoped. This is
   Massdriver's model and it is why a connection cannot be "just a label".

### Type checking

A connection is valid only if the producer's port type is assignable to the
consumer's declared input type. The port type catalogue is small and closed;
adding a type is a platform release, adding a *brick* is not.

| Port type | Carries |
| --- | --- |
| `git-source` | repo URL, ref, resolved commit, credentials ref |
| `oci-image` | image reference, digest, resolved tag |
| `sql` | host, port, database, user, password ref, sslmode |
| `redis` | host, port, password ref, tls |
| `queue` | protocol, endpoint, vhost/topic, credentials ref |
| `s3-bucket` | endpoint, region, bucket, access key ref, path style |
| `http-endpoint` | URL, scheme, optional auth ref |
| `k8s-service` | cluster ref, namespace, service name, ports |
| `tls-cert` | secret ref, DNS names, not-after |
| `dns-name` | FQDN, record type, zone |
| `secret` | opaque secret ref plus key schema |
| `config` | opaque structured values |
| `openapi-spec` | ConfigMap ref, format, version |
| `docs-site` | URL, source ref |
| `target` | a deployment destination (see §10) |

## 5. Layer 4 — `Application` as an aggregate, plus `Release`

Constraint V5 says the application is a set of resources joined by reference,
label selector or annotation. The `Application` object is therefore **thin**:

```yaml
apiVersion: apps.cozyap.io/v1alpha1
kind: Application
metadata:
  name: payments-api
  namespace: team-payments
spec:
  blueprintRef: { name: java-service, version: 2.1.0 }   # optional
  selector:
    matchLabels: { app.cozyap.io/application: payments-api }
  owners: [team-payments]
status:
  health: Degraded          # worst-of over members
  members: 7
  facets: [...]             # aggregated from members
```

It owns no bricks. It **selects** them. Consequences, all wanted:

- A resource created outside the platform can be adopted by labelling it.
- RBAC and status are per brick instance, not one coarse blob.
- The graph editor operates on real objects, not on a section of a large CR.

### `Release` — the answer to rollback (gap G1)

Requirement R4's rollback has no home in a purely declarative model. A
`Release` is an immutable snapshot of *resolved* inputs across the
application's bricks:

```yaml
kind: Release
spec:
  applicationRef: payments-api
  number: 42
  resolved:
    build.image:  "registry/…@sha256:abc…"
    source.commit: "9f2c1a…"
    blueprintVersion: "2.1.0"
    parameters: { … }          # every brick's resolved parameters
status:
  promotedAt: "2026-09-07T10:00:00Z"
  active: true
```

Rollback is "make release N active again". What a release does **not** capture
must be stated to the user plainly: database schema migrations are not
reversible, and a rollback that crosses a migration is a data operation, not a
deployment operation. This belongs in the UI copy, not only in the docs.

## 6. Traits — what is *not* a node

Taken from OAM, and this is what keeps graphs readable. An autoscaler, a backup
schedule, a network policy, a sidecar, a PDB, a resource limit — these attach
to a brick instance and never appear as graph nodes. Modelling them as nodes
would triple node count for zero information.

Rule of thumb, worth enforcing in the linter:

> If it has an output another brick consumes, it is a **brick**.
> If it only modifies the behaviour of one brick, it is a **trait**.

Initial trait set: `resource-limits`, `autoscaling`, `backup`, `monitoring`,
`rollout` (strategy), `network-policy`, `sidecar`, `pdb`, `schedule`.

## 7. Layer 5 — facets, the presentation contract

Constraint V5's "capabilities". A facet is a typed contribution from a brick to
the application view. The UI aggregates facets across the aggregate's members
and knows nothing about brick types.

| Facet kind | Renders as | Example |
| --- | --- | --- |
| `link` | An external link with icon | Grafana dashboard, docs site, SCM repository, log query |
| `metric` | A summary number on the app card | request rate, DB size, player count |
| `panel` | An embedded view | build log tail, recent deployments |
| `action` | A button that triggers a `Task` | manual backup, restart, rotate credentials, re-run scaffolding |
| `health` | A condition contributing to app health | "TLS certificate expires in 5 days" |
| `badge` | A small status marker | "image update available" |

Facets are declared in `BrickDefinition.spec.facets` with templates, and
materialised into `BrickInstance.status.facets` with values resolved. The UI
reads only status.

### UI extensibility, revised

ADR-0020 chose declarative view descriptors (level 1) and rejected Module
Federation plugins (level 2). Constraint V3 asks for level 2. Revised position:

- **Level 0** — forms generated from the parameter schema. Default, free.
- **Level 1** — declarative view descriptors in the package. **Default for
  facets and panels.** Covers most bricks.
- **Level 2** — a `UIPlugin` custom resource pointing at an image serving a
  Module Federation remote, loaded at runtime. **Available, opt-in, and the
  brick must declare it.**

```yaml
kind: UIPlugin
spec:
  image: registry/…/cs2-console-ui:1.2.0
  exposes:
    - { scope: brickPanel, brickType: game.cozyap.io/CS2Server, module: ./Console }
    - { scope: graphNode,  brickType: game.cozyap.io/CS2Server, module: ./Node }
```

This is a proven pattern — Red Hat Developer Hub loads Backstage frontend
plugins exactly this way, from OCI images, at runtime. The costs are known and
must be accepted explicitly:

- a plugin/host API compatibility matrix, forever;
- a host restart or re-fetch to pick up plugin changes;
- a security boundary that is *not* a sandbox — a level-2 plugin runs in the
  operator's browser session with its privileges, so plugin images must be
  signed and allowlisted, exactly as ADR-0020 §3.5 requires for packages;
- a plugin that fails to load must degrade to level 1, never break the page.

The `graphNode` scope above is the interesting one: it lets a package supply
the *renderer for its own node* in the graph editor, which is only possible
because the editor is built from React components (§ graph-editor doc).

## 8. The brick catalogue — first cut

Grouped by role. `Native` implementations are ours; the rest are package
content.

### Source and SCM

| Brick | In | Out | Impl | Notes |
| --- | --- | --- | --- | --- |
| `GitSource` | — | `git-source` | Native | Points at the customer's repository. Covers requirement R1's "fetch source". |
| `GitRepository` | — | `git-source` | Composition | *Creates* a repo (GitLab/GitHub/Gitea provider). Roadmap, not v1. |
| `SourceTrigger` | `git-source` | — | Native | Webhook registration plus polling fallback; emits change events. |
| `PipelineGenerator` | `git-source` | `config` | TaskPipeline | Writes CI config into the repo (GitLab CI, GH Actions) so the platform later sees produced images. Constraint V5. |

### Build and image

| Brick | In | Out | Impl | Notes |
| --- | --- | --- | --- | --- |
| `Build` | `git-source` | `oci-image` | Native | kpack / Cloud Native Buildpacks. No build scripts — requirement R1. |
| `DockerfileBuild` | `git-source` | `oci-image` | Native | Escape hatch when a Dockerfile exists. |
| `ImageRef` | — | `oci-image` | Native | An externally built image. Makes WordPress and CS2 expressible without a build. |
| `ImagePolicy` | `oci-image` | `oci-image` | Native | Watches a registry for new tags matching a policy; emits an updated image. Closes "the platform sees the image and its updates". |

### Workload

| Brick | In | Out | Impl | Notes |
| --- | --- | --- | --- | --- |
| `Workload` | `oci-image`, `config`, connections | `k8s-service` | Native | The stateless case. Owns revisions. |
| `StatefulWorkload` | as above + volumes | `k8s-service` | Native | WordPress, CS2, anything with disk identity. |
| `HelmWorkload` | `config` | `k8s-service` | OperatorResource | Off-the-shelf charts. The pragmatic path for WordPress. |
| `VirtualMachine` | `config` | `k8s-service` | Composition | Cozystack/KubeVirt. Constraint V4's outer edge. |
| `ScheduledWorkload` | `oci-image` | — | Native | CronJob-shaped. |

### Data and dependencies (requirement R2's closed set, extensible)

`PostgresDatabase`, `MySQLDatabase`, `RedisCache`, `MessageQueue`,
`ObjectBucket`, `ClickHouseDatabase`, `MongoDatabase` — all `Composition`,
all with per-target implementations, all emitting exactly one typed connection
port. These are where the Cozystack provider earns its place, and where a
different target class swaps in RDS or an external service without the
application changing.

### Networking and exposure

| Brick | In | Out | Impl | Notes |
| --- | --- | --- | --- | --- |
| `HttpRoute` | `k8s-service`, `tls-cert`, `dns-name` | `http-endpoint` | Composition | Requirement R4's "publish with an address". |
| `Certificate` | `dns-name` | `tls-cert` | Composition | cert-manager. |
| `DnsRecord` | `k8s-service` | `dns-name` | Composition | external-dns or a provider. |
| `L4Endpoint` | `k8s-service` | `http-endpoint` | Composition | UDP/TCP LoadBalancer. Required for CS2; not expressible as an Ingress. |

### Platform services (attached by default — requirement R5)

| Brick | Impl | Notes |
| --- | --- | --- |
| `ObservabilityBinding` | Composition | Scrape config, log routing, default dashboards. Injected by the blueprint, not chosen by the developer. |
| `SecretStore` | Composition | Vault/OpenBao path plus scoped policy for the application. |
| `BackupPolicy` | trait, not a brick | Attaches to stateful bricks. |

### Integration and metadata

| Brick | In | Out | Impl | Notes |
| --- | --- | --- | --- | --- |
| `OpenApiSpec` | `http-endpoint` or `git-source` | `openapi-spec` | TaskPipeline | Fetches and versions the spec. Constraint V5. |
| `DocsSite` | `git-source` | `docs-site` | TaskPipeline | Documentation generator. |
| `ExternalResource` | — | any | External | Adoption of something the platform did not create. |

## 9. Service bricks, and the `PythonEval` question

Constraint request: a `Kind: PythonEval` with an inline script that does work
and writes status to its own resource.

**This conflates two different primitives, and shipping them as one produces a
worse version of both.**

### Pure computation does not need a resource

Deriving values, reshaping parameters, computing a name — this is *rendering*.
Crossplane already has `function-python` with the full Python standard library
inside the composition pipeline, plus `function-kcl` (sandboxed) and
`function-kro` (YAML + CEL with a reference-derived DAG). A resource for this
would be strictly worse: an extra object, an extra reconcile, an extra failure
mode, and no sandbox.

**Rule: computation that produces desired state belongs in a composition
function, never in a CR.**

### Side-effecting work does need a resource

Calling an external API, running a migration, scaffolding a repository, taking
a backup — this needs its own lifecycle, retry policy, timeout, log surface,
idempotency key and status. That is a real primitive, and Crossplane does not
have it. Kratix's container workflows are the right shape.

So the primitive is **`Task`**, and Python is one runtime of several:

```yaml
kind: Task
spec:
  runtime:
    python:
      inline: |
        import os, json
        # inputs arrive as JSON on stdin, outputs are written to stdout
        …
      requirements: [requests==2.32.*]     # resolved at build, cached
  # or: shell.inline, or image + args
  inputs:
    fromConnections: [db]
    values: { retention: 14 }
  idempotencyKey: "{{ .spec.inputs.hash }}"
  policy:
    timeout: 15m
    retries: { limit: 3, backoff: exponential }
  placement: platform          # never the data plane — constraint V6
status:
  phase: Succeeded
  attempts: 1
  outputs: { … }               # becomes a typed port if declared
  logsRef: { … }
```

`ScheduledTask` is the same with a `schedule`. **`PythonEval` is a profile of
`Task`** — sugar for `runtime.python.inline` with no connections and a short
timeout — offered because it is a genuinely convenient shorthand, not because
it is a distinct concept.

### Non-negotiables for inline code

Inline code in a CR is remote code execution with a YAML front end. It is
acceptable, but only with all of the following, and they belong in v1, not in a
hardening pass:

1. **Runs in the platform plane, never the data plane.** Constraint V6, and
   also the difference between "a task" and "a foothold in a tenant cluster".
2. **A dedicated, minimal service account per task**, scoped to what the brick
   declared. Not the platform's identity.
3. **Explicit RBAC on creating tasks with inline code.** A developer filling in
   a form must not be able to create one; a package author and a platform
   operator may. Inline code from a *blueprint* is reviewed content; inline code
   from a user is an escalation.
4. **Network policy default-deny**, with declared egress only.
5. **Resource limits and a hard timeout**, always set, no unbounded default.
6. **No dependency resolution at run time.** `requirements` are resolved and
   cached when the package is built; a task that pip-installs at runtime is
   both a supply-chain hole and a flake.
7. **Idempotency key required** for anything with side effects, with the same
   discipline ADR-0011 called `transitionId`.

If any of these is inconvenient enough to be skipped, the honest answer is to
ship the work as a package image instead of inline code.

## 10. Targets — constraint V7

`Target` replaces ADR-0002's `Environment` and generalises it:

```yaml
kind: Target
metadata: { name: prod-eu }
spec:
  class: cozystack | kubeconfig | cloud       # selects implementations (§3)
  cozystack: { tenantRef: acme, cluster: prod-eu }
  # or kubeconfig: { secretRef: … }
  capabilities: [ingress, loadbalancer-l4, block-storage, gpu]
  defaults:
    observability: platform-stack
    secretStore: openbao-prod
```

Two things this buys:

- `class` is what `BrickDefinition.implementations` matches on, so Cozystack
  becomes one provider rather than the platform (constraint V7).
- `capabilities` lets the blueprint linter reject an application before deploy —
  a CS2 blueprint requiring `loadbalancer-l4` cannot be placed on a target
  without it. Failing at authoring time rather than at deploy time is the
  difference between a platform and a pile of controllers.

## 11. Data plane hygiene — constraint V6

Default posture: a data plane runs **workloads and nothing else**. No
Crossplane, no platform operator, no task runner. Reconciliation, composition
and tasks all execute in the platform plane and act on the data plane through a
scoped credential.

When a brick genuinely needs machinery inside the target — CNPG for a
PostgreSQL brick, a game-server operator, an ingress controller — it declares
it:

```yaml
targetRequirements:
  - package: cnpg-operator
    version: ">=1.24"
    scope: dataplane
```

The platform installs it on demand and **reference-counts** it: the operator is
removed when the last brick instance requiring it is gone. Refcounting is the
non-obvious part and the reason this cannot be left to "the blueprint installs
what it needs" — two applications sharing an operator, one deleted, must not
break the other.

Two further rules:

- **Version conflicts are a target-level concern.** Two bricks requiring
  incompatible versions of the same operator on one target is a rejection at
  authoring time, not a runtime surprise.
- **Requirements are declared, never implicit.** A brick that quietly assumes
  something is installed is a bug the linter must catch.

## 12. Packaging — constraints V2 and V3

**Correction to ADR-0020:** do not invent a bundle format. Cozystack v1.0
already ships `Package` / `PackageSource` driven by `cozystack-operator`, with
git and OCI sources, flavors, and a marketplace surface via
`ApplicationDefinition`. Constraint V2 says that is the vehicle.

What changes: the **content model** is ours and must stay portable, because
constraint V7 requires a non-Cozystack installer to consume the same artefact.

A package may contain:

| Content | Consumed by |
| --- | --- |
| `BrickDefinition`s | Platform |
| Crossplane XRDs, Compositions, function and provider dependencies | Crossplane |
| Operators (Helm charts, manifests) | Target, via `targetRequirements` |
| `TaskPipeline` definitions and their images | Platform |
| `ApplicationBlueprint`s | Catalog (requirement R6) |
| UI view descriptors (level 1) | Portal |
| `UIPlugin` (level 2, Module Federation image) | Portal |
| Dashboards, alert rules | Observability stack |
| RBAC presets, policies, quotas | Platform |
| Billing metadata | Metering pipeline |

`flavors` in the Cozystack package model is the same idea as per-target
implementations and should be used rather than duplicated.

## 13. Worked examples — does it actually cover constraint V4?

### Java service (requirement 5.14's core case)

```text
GitSource ──git-source──▶ Build(kpack) ──oci-image──▶ Workload
                                                        │
PostgresDatabase ──sql──────────────────────────────────┤
ObjectBucket ─────s3-bucket─────────────────────────────┤
                                                        ▼
                                          k8s-service ──▶ HttpRoute ──▶ url
Traits: autoscaling, resource-limits, monitoring
Facets: Grafana link, build-log panel, "rollback" action
```

Six bricks. The developer's form asks for: repository, size, which
dependencies, hostname. Everything else comes from the blueprint.

### WordPress

```text
ImageRef(wordpress:6.x) ──oci-image──▶ StatefulWorkload ──▶ HttpRoute
MySQLDatabase ──sql────────────────────┤
ObjectBucket  ──s3-bucket──────────────┘   (media offload)
Traits: backup (on StatefulWorkload and MySQLDatabase)
```

No build brick at all. This is why `ImageRef` exists as a first-class brick
rather than as a field on `Build` — an application without a build stage must
not be a special case.

Alternative, and probably better in practice: a single `HelmWorkload` brick
wrapping the upstream chart, with the database still a separate brick so it can
be backed up and sized independently.

### CS2 game server

```text
ImageRef(cs2) ──oci-image──▶ StatefulWorkload ──k8s-service──▶ L4Endpoint(UDP)
ConfigMap-ish `config` brick for server settings
Traits: resource-limits, schedule (nightly restart)
Facets: player-count metric, RCON console panel (level-2 UIPlugin), "restart" action
Target must advertise capability: loadbalancer-l4
```

This is the case that proves the model is not a twelve-factor PaaS in disguise:
no build, no HTTP, no database, a UDP endpoint, and a bespoke UI panel — and it
uses the same five layers.

## 14. What this model deliberately does not do

- **No dynamic fan-out, no higher-order bricks** (ADR-0017 stays out of scope).
  A brick instance is created by a blueprint or a human, never by another brick
  generating N of them. Revisit only with a concrete requirement.
- **No cross-application graph.** Applications may reference each other's
  `http-endpoint` via `ExternalResource`, but there is no global graph. That is
  a catalog concern (Backstage-style), not a dataflow concern.
- **No arbitrary user-authored inline code by default.** See §9's RBAC rule.

## 15. Open questions

1. **Does `Application` also need to be able to *own* bricks**, not only select
   them, so that deleting an application deletes its parts? Probably yes, via a
   `reclaimPolicy`, but selection and ownership must not be conflated.
2. **Blueprint rendering** — does a blueprint produce brick instances once
   (scaffold, then they diverge) or continuously (a change to the blueprint
   updates every application using it)? These are very different products.
   Requirement R6's "versioned by the Customer" implies the second; the first is
   far simpler. **This needs a decision before implementation.**
3. **Where a connection's authorisation work runs** when the producer and
   consumer sit on different targets.
4. **Port type versioning** — adding a field to `sql` is a platform release
   affecting every brick that emits it. Needs a compatibility rule.
5. **Trait conflict resolution** — two traits both editing a pod spec.
6. **Task log retention and its cost**, given tasks are how everything
   imperative happens.
