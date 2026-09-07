# Research: Cold Start Cost of Pipeline Bricks

## Status

Measurement note, 2026-09-07. Input to the decision on `kind: Pipeline` as the
primary brick implementation (`docs/design/brick-model.md` §3).

Question asked: if every brick reconcile is a Job, does the pod churn break the
platform plane?

## Method

Single-node kind cluster, Kubernetes 1.35, containerd 2.2, kindnet CNI,
10 vCPU / 8 GiB, 10 pre-existing pods, no competing load. The payload is a
realistic brick renderer: `python:3.13-alpine` running a script that emits a
manifest plus an `outputs.json` — verified to actually produce both, not just
to exit.

Measurement overhead (two `kubectl` invocations per run) was measured
separately at ~97 ms mean and is subtracted where noted.

## Results

| Measurement | Value |
| --- | --- |
| Sequential reconcile, warm image cache | **~3.0 s** (median 3.2 s, minus 0.19 s harness overhead) |
| Burst of 30 jobs, wall clock to all complete | 9.3 s |
| **Sustained pod rate, one node** | **3.2 pods/s** |
| Per-pod amortised cost in a burst | 309 ms |
| **API writes per reconcile** (pods + jobs, POST/PUT/PATCH/DELETE) | **10.6** |
| Cold image cache, 2–4 MB image | **7.5 s** — the pull adds ~4.5 s |
| **Warm alternative: exec into a running pod** | **~53 ms** net of harness overhead |

The last two rows carry the design consequences.

**Warm versus cold is 57×.** 53 ms against 3.0 s.

**A trivially small image still costs 4.5 s to pull.** That is registry
round-trip plus extraction plus containerd bookkeeping, not bandwidth — a
200 MB brick image would be far worse.

### Caveats, stated so the numbers are not over-trusted

- One node, on a macOS VM, with kindnet. Cilium or kube-ovn do more work per
  pod (IPAM, eBPF program attach) and will be slower.
- No competing load. A real platform plane runs monitoring, other controllers
  and its own workloads.
- Pod-churn capacity is kubelet-bound, so it scales roughly per node; a
  three-node plane gets roughly three times the ceiling.

**Design against ~1 pod/s per node, not 3.2.** Treat the measured ceiling as
optimistic.

## The model

Steady-state pod rate from drift checking is `applications × bricks ÷ interval`.
With 8 bricks per application:

| Applications | Bricks | drift 24 h | drift 6 h | drift 1 h | drift 15 min | **status @ 60 s** |
| --- | --- | --- | --- | --- | --- | --- |
| 10 | 80 | 0.001/s | 0.004/s | 0.02/s | 0.09/s | **1.3/s** |
| 50 | 400 | 0.005/s | 0.02/s | 0.11/s | 0.44/s | **6.7/s** |
| 200 | 1 600 | 0.02/s | 0.07/s | 0.44/s | 1.8/s | **26.7/s** |
| 1 000 | 8 000 | 0.09/s | 0.37/s | 2.2/s | 8.9/s | **133/s** |

Measured ceiling: 3.2 pods/s per node; safe budget ~1 pod/s per node.

## Findings

### 1. Reconcile churn is a non-issue

At a 6-hour drift interval, **1 000 applications produce 0.37 pods/s** — about
a third of the safe budget for a single node, and roughly 4 API writes per
second. There is no throughput problem to solve.

At a 1-hour interval, 1 000 applications reach 2.2 pods/s, which is past the
safe budget for one node and near the measured ceiling. So the interval is a
real design parameter, but the usable range is wide.

### 2. What actually breaks is the status loop

ADR-0008 specified status as a **separate workflow per atom on a 30–60 second
cron**. If status is a pod, the numbers above move to the last column:

- at 60 s, **50 applications already exceed the single-node ceiling** (6.7/s
  against 3.2/s);
- at 30 s it breaks at roughly **12 applications**;
- at 200 applications it is 26.7 pods/s and ~283 API writes per second of pure
  overhead, competing with the platform's real work.

**Rule: status must never be a pod.** It is derived in-process from watching the
conditions of the resources the brick applied. Cost: zero pods, and it is also
fresher than a 60-second cron.

This is the single most valuable output of the measurement, and it invalidates
a decision that has been sitting in ADR-0008 unchallenged.

### 3. Image pull is a package-upgrade problem, not a steady-state one

The 4.5 s pull penalty is paid on the first reconcile of a brick version **on
each node**. Consequences:

- pre-pull brick images when a package is installed or upgraded, rather than
  discovering the cost on a user's first action;
- keep brick images small — a script on Alpine, never a full language SDK;
- pin by digest so the cache is actually reused.

### 4. Where warm mode is genuinely needed — and it is not load

Interactive latency is chain depth × 3 s:

| User action | Chain | Cold | Warm |
| --- | --- | --- | --- |
| Change an environment variable | 1 hop | 3 s | 0.05 s |
| Rotate a database password | DB → Workload, 2 hops | 6 s | 0.1 s |
| Push a commit | Trigger → Build → Workload → Route, 4 hops | 12 s | 0.2 s |

For creation and build flows the pod overhead is noise next to the real work —
a build takes minutes, provisioning a database takes tens of seconds. For small
configuration changes, 3–6 s is noticeable but tolerable.

**Warm mode is therefore an optimisation for interactive chains, not a
prerequisite for scale.** It can arrive later and target only the bricks on the
critical path of a user action, which is a much smaller job than making it
mandatory from day one.

## Conclusion

`kind: Pipeline` as the primary implementation is affordable, subject to three
rules that follow directly from the numbers:

1. **Status is derived from watches, never a pod.** Without this the design
   breaks at tens of applications.
2. **Drift checking runs on hours, not minutes**; the primary trigger is change,
   not cron.
3. **Brick images are small and pre-pulled at package install**, pinned by
   digest.

Warm execution is deferred, and scoped to interactive chains when it arrives.

## Reproducing

The benchmark is three parts: sequential latency with a warm cache, a 30-job
burst for sustained rate, and an exec into a running pod for the warm
comparison, with `apiserver_request_total` sampled either side of the burst for
the write count. It needs any throwaway cluster and takes about two minutes.
