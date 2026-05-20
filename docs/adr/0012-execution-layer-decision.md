# ADR-0012: Execution Layer Decision

## Status

Proposed. Decision **pending**.

This ADR compares ADR-0010 (Option A: Temporal) and ADR-0011 (Option B:
Pods with stdin/stdout) and will record the chosen execution layer once the
decision is made. All three ADRs (0010, 0011, 0012) remain Proposed until
the decision is committed here.

## Context

ADR-0008 established the reactive reconcile model and identified two viable
execution layers. ADR-0010 and ADR-0011 specify each option in full. This ADR
makes the choice.

The decision is load-bearing: the execution layer affects atom author
experience, operational complexity, blast radius, and the platform's ability
to support long-running side-effect operations (DB migrations, basebackups).
Changing the execution layer after atoms have shipped requires rewriting every
atom image's integration layer.

The atom contract (ADR-0009) is executor-neutral and stays unchanged
regardless of this decision.

## Comparison

### Head-to-head matrix

| Dimension | Option A — Temporal | Option B — Pods stdin/stdout |
|-----------|--------------------|-----------------------------|
| **Durability** | Native — workflows survive worker restart; activities replay from last checkpoint | DIY — atom manages `transitionId`; crash before writing result = side-effect repeats |
| **Ops complexity** | High — Temporal cluster (HA, PostgreSQL backend, upgrades, retention policies) | Low — only Kubernetes Jobs; no external server |
| **Blast radius** | Wide — Temporal outage suspends all tenants' reconciles simultaneously | Narrow — broken atom fails its own Job only |
| **Atom author complexity** | Medium — must learn Temporal determinism constraints, SDK, activity model | Low — read JSON from stdin, write JSON to stdout; any language, stdlib sufficient |
| **Long-running operations** | First-class — activities can run for hours; workflow replays from last activity | Timeout-limited — re-entrant logic across Job invocations required for ops > `reconcile.timeout` |
| **Side-effect tracking** | Built-in activity IDs; SDK `Transitions.Attempt` is safe by construction | Manual `transitionId`; atom author must write correctly or risk double-execution |
| **Idle resource cost** | Persistent atom worker Deployments (scale-to-zero via KEDA, but KEDA = extra dep) | Zero — Jobs are ephemeral; no idle processes |
| **Startup latency per reconcile** | ~0 ms (worker already running, task dispatched over gRPC) | ~1–5 s cold / <1 s warm (Pod scheduling + image pull) |
| **Debugging** | Temporal UI: full workflow history, activity inputs/outputs, retry counts | `kubectl logs job/<name>`; standard K8s tooling |
| **Vendor / tech dependency** | Temporal (OSS, self-hosted, but non-trivial ops) | None beyond Kubernetes |
| **Multi-tenant isolation** | Temporal namespace per tenant; strong history isolation | K8s namespace + RBAC; standard isolation |
| **Language ecosystem** | Go/Java/Python/TypeScript SDKs official; other langs via gRPC | Any language with stdin/stdout; no SDK needed |
| **Catalog linter burden** | Must detect non-deterministic workflow code (time.Now, rand, etc.) | Must verify transitionId discipline and stdout hygiene |
| **Scale-to-zero** | KEDA `ScaledObject` on Temporal task queue depth | Free — Jobs created on demand |

### Detailed trade-off analysis

#### Durability: the decisive dimension

The reactive model's key scenario is a database major-version upgrade:

1. Reconcile triggers (user bumped `version: "15"` → `"16"`).
2. Atom takes a basebackup (15 min).
3. Atom starts the upgrade migration (10 min).
4. Worker crashes at step 3, half-way through migration.
5. Next reconcile trigger fires.

**Temporal** (Option A): the workflow replays from the last committed
activity. Step 2 is already in history; the workflow resumes at step 3,
finding the partially applied migration via the Temporal activity ID. With
`Transitions.Attempt("migrate-pg15-to-pg16")`, step 3 either resumes or is
reported as already done — no double-execution.

**Pods** (Option B): the Job died at step 3. The next Job receives
`transitionId: "basebackup-done"` (written after step 2 succeeded), so it
skips step 2. But step 3 was mid-execution — the atom must detect partial
migration state itself (e.g. by querying the DB) and resume or roll back.
The `transitionId` protects committed steps; it does not automatically handle
partially-committed steps. The atom author must write this detection logic.

For casual catalog atoms (Minecraft, a Node.js app, a Redis cache), the
durability difference does not matter — their reconcile is fast and
idempotent. For database atoms and anything with long side effects, the
difference is significant.

#### Operational cost vs platform complexity

Temporal requires:

- HA Temporal server deployment (3+ replicas recommended).
- A PostgreSQL database backend (Cozystack can provide this, but it becomes
  a dependency chain: Cozyap → Temporal → Cozystack PostgreSQL).
- Operational runbooks: Temporal upgrades, history database vacuuming,
  namespace retention policies, disaster recovery.
- The `cozy-system` namespace runs a non-trivial stateful service visible
  to all tenants.

Pods require:

- Nothing beyond the Kubernetes API already in use.
- The operator's Job-management logic is a few hundred lines.
- Failure modes are familiar (Pod OOMKilled, ImagePullBackOff) and have
  existing tooling.

For an ISP offering Cozystack to customers, Temporal represents a meaningful
ops burden increase. For a platform team already running distributed systems,
it is manageable.

#### Author experience

Temporal's SDK enforces workflow determinism. Atom authors must understand:

- Which operations can happen in workflow code vs. activity code.
- Why `time.Now()` inside a workflow function is illegal.
- How to structure activities for proper replay.

This is not prohibitive for a professional SDK author, but it is a barrier
for a community contributor writing their first atom.

Pods' stdin/stdout contract is immediately understandable: shell script, Python,
Go, or Rust — read JSON, do work, write JSON, exit 0. The `transitionId`
pattern has a learning curve but can be documented with a simple example.

### When each option wins

**Choose Temporal (Option A) if:**

- A significant fraction of catalog atoms are expected to have long-running
  side-effect operations (multi-hour database upgrades, large data migrations).
- The platform team has the operational capacity to run and maintain a
  Temporal cluster.
- Strong auditability (full workflow history per tenant) is a requirement.

**Choose Pods (Option B) if:**

- Most catalog atoms are lightweight (game servers, web apps, caches) with
  fast, idempotent reconciles.
- Minimising operational complexity is the primary constraint (early-stage
  platform, small ops team).
- Wide language support for atom authors is important.
- A Temporal outage affecting all tenants simultaneously is an unacceptable
  risk.

## Decision

**Pending.**

The decision will be recorded here with the following fields:

```
Chosen option: [A — Temporal | B — Pods]
Decision date: YYYY-MM-DD
Rationale: <one paragraph>
Conditions: <any conditions or phased plan, e.g. "start with B, migrate to A
             when database atoms are needed">
```

Once committed, ADR-0010 or ADR-0011 is promoted to **Accepted** and the
other to **Rejected** (kept for historical context). ADR-0015 (Platform
Integration) is written based on the chosen option.

## Consequences of deferring

While the decision is pending:

- ADR-0009 (Atom Contract) is executor-neutral and can be implemented
  immediately.
- ADR-0013 (Application Graph) is executor-neutral and can be implemented
  immediately.
- ADR-0014 (Status and Conditions API) is executor-neutral.
- ADR-0015 (Platform Integration) blocks on this decision — execution
  placement and wiring injection differ between the two options.
- No atom images can be published to the catalog until the execution layer
  is decided and the SDK contract is stable.

The decision should be made before any atom SDK work begins.
