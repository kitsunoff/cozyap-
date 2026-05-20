# ADR-0002: Environment Abstraction

## Status

Superseded by `docs/cozya-new-notes.md` (pending new ADR series, 0008+).

The Environment abstraction itself survives the reactive redesign, but the
fixed controller set, kubeconfig wiring, and "Argo per Environment" decisions
will be revisited in ADR-0013 (Platform Integration). Kept for historical
context.

## Context

Applications need a target cluster to run in. Instead of exposing raw Kubernetes cluster management to tenants, we introduce Environment — an abstraction that declaratively provisions and manages application runtime clusters.

Environment handles:
- Cluster provisioning via Cozystack API
- Required controller installation (Argo Workflows, etc.)
- Health monitoring and resource tracking
- Isolation between tenants

## Decision

### Multi-Tenancy Model

Cozy User Apps reuses the existing Cozystack multi-tenancy model. No additional tenant isolation is implemented at this layer.

```
┌─────────────────────────────────────────────────────────────┐
│                     Cozystack Cluster                        │
├─────────────────────────────────────────────────────────────┤
│  Cozystack Multi-Tenancy (existing):                         │
│  ┌─────────────────┐  ┌─────────────────┐                    │
│  │ tenant-acme     │  │ tenant-globex   │                    │
│  │ (namespace)     │  │ (namespace)     │                    │
│  │                 │  │                 │                    │
│  │ • SA with RBAC  │  │ • SA with RBAC  │                    │
│  │ • ResourceQuota │  │ • ResourceQuota │                    │
│  │ • NetworkPolicy │  │ • NetworkPolicy │                    │
│  └─────────────────┘  └─────────────────┘                    │
│                                                              │
│  Cozy User Apps resources (per tenant namespace):            │
│  ┌─────────────────┐  ┌─────────────────┐                    │
│  │ tenant-acme     │  │ tenant-globex   │                    │
│  │ ┌─────────────┐ │  │ ┌─────────────┐ │                    │
│  │ │ Environment │ │  │ │ Environment │ │                    │
│  │ │ Application │ │  │ │ Application │ │                    │
│  │ │ Action      │ │  │ │ Action      │ │                    │
│  │ └─────────────┘ │  │ └─────────────┘ │                    │
│  └─────────────────┘  └─────────────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

**Key points:**

- **Tenant = Cozystack namespace** — Each tenant operates within their Cozystack-provisioned namespace
- **ServiceAccount per tenant** — Tenant accesses resources via their namespace ServiceAccount
- **Full access within namespace** — Tenant can manage all Cozy User Apps resources (Environment, Application, Action) in their namespace
- **No cross-tenant access** — Cozystack RBAC prevents access to other tenant namespaces
- **ApplicationTemplates are cluster-scoped** — Read-only for all tenants, managed by platform operator

This design deliberately avoids reimplementing tenant isolation. Cozystack already provides:
- Namespace-based isolation
- RBAC for ServiceAccounts
- Resource quotas
- Network policies between tenant namespaces

### Environment CRD

Namespace-scoped resource. One tenant can have multiple environments (dev, staging, prod).

```yaml
apiVersion: apps.cozystack.io/v1alpha1
kind: Environment
metadata:
  name: production
  namespace: tenant-acme
spec:
  # Optional: cluster sizing hints
  # If omitted, platform decides automatically
  sizing:
    maxNodes: 10
    nodeSize: medium  # small | medium | large | xlarge

status:
  # Overall state
  phase: Ready  # Pending | Provisioning | Ready | Degraded | Deleting

  # Cluster health (updated every 30s)
  health:
    status: Healthy  # Healthy | Unhealthy | Unknown
    lastCheckTime: "2024-01-15T10:30:00Z"
    message: "All systems operational"

  # Cluster resources (updated every 30s)
  resources:
    nodes:
      total: 3
      ready: 3
    capacity:
      cpu: "12"
      memory: "48Gi"
    allocatable:
      cpu: "11"
      memory: "44Gi"

  # Generated kubeconfig for accessing the cluster
  kubeconfigSecretRef:
    name: production-kubeconfig

  # Controller status (fixed set, managed by platform)
  controllers:
    - name: argo-workflows
      ready: true
      version: "3.5.2"
    - name: argo-events
      ready: true
      version: "1.9.0"

  # Conditions for detailed status
  conditions:
    - type: ClusterReady
      status: "True"
      lastTransitionTime: "2024-01-15T10:00:00Z"
      reason: ClusterProvisioned
      message: "Kubernetes cluster is running"

    - type: ControllersReady
      status: "True"
      lastTransitionTime: "2024-01-15T10:05:00Z"
      reason: AllControllersHealthy
      message: "All required controllers are ready"

    - type: Healthy
      status: "True"
      lastTransitionTime: "2024-01-15T10:30:00Z"
      reason: HealthCheckPassed
      message: "Cluster health check passed"

  # Reference to underlying Cozystack resource
  clusterRef:
    apiVersion: cozystack.io/v1alpha1
    kind: ManagedKubernetes
    name: tenant-acme-production
```

### Lifecycle

```
User creates Environment
        │
        ▼
┌───────────────────┐
│     Pending       │
└─────────┬─────────┘
          │ Controller detects new Environment
          ▼
┌───────────────────┐
│   Provisioning    │ ◄── Creates ManagedKubernetes via Cozystack API
└─────────┬─────────┘     Waits for cluster ready
          │               Installs controllers (Argo Workflows, Events)
          │               Generates kubeconfig Secret
          ▼
┌───────────────────┐
│      Ready        │ ◄── Health checks every 30s
└─────────┬─────────┘     Updates resources in status
          │
          │ Health check fails
          ▼
┌───────────────────┐
│     Degraded      │ ◄── Applications may not work correctly
└─────────┬─────────┘     Controller attempts recovery
          │
          │ Health restored
          ▼
┌───────────────────┐
│      Ready        │
└───────────────────┘
```

### Fixed Controller Set

Every Environment gets the same set of controllers, managed by the platform:

| Controller | Purpose |
|------------|---------|
| Argo Workflows | DAG execution engine for application hooks |
| Argo Events | Event-driven triggers (future: webhooks, git events) |

Tenants cannot customize this list. Platform operator decides which controllers are required.

### Node Sizing

| Size | Description |
|------|-------------|
| `small` | 2 vCPU, 4Gi RAM |
| `medium` | 4 vCPU, 8Gi RAM |
| `large` | 8 vCPU, 16Gi RAM |
| `xlarge` | 16 vCPU, 32Gi RAM |

If `spec.sizing` is omitted, platform automatically selects appropriate node size based on workload.

### Kubeconfig Secret

Generated by controller after cluster provisioning:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: production-kubeconfig
  namespace: tenant-acme
  ownerReferences:
    - apiVersion: apps.cozystack.io/v1alpha1
      kind: Environment
      name: production
type: Opaque
data:
  kubeconfig: <base64-encoded-kubeconfig>
```

Secret is owned by Environment and deleted when Environment is deleted.

### Integration with Application

Application references Environment by name:

```yaml
apiVersion: apps.cozystack.io/v1alpha1
kind: Application
metadata:
  name: my-minecraft
  namespace: tenant-acme
spec:
  templateRef: minecraft
  targetEnvironment: production  # References Environment in same namespace
  values:
    serverName: "ACME Gaming"
```

Controller resolves `targetEnvironment` → Environment → `status.kubeconfigSecretRef` → kubeconfig to access target cluster.

### Storage

Environment cluster provides default StorageClass for persistent volumes. Applications request storage via standard Kubernetes PVC mechanisms.

- **Default StorageClass** — Provisioned by Cozystack, available in every Environment
- **No custom StorageClasses** — Applications use whatever default is configured
- **Storage sizing** — Defined in ApplicationTemplate parameters, passed to Helm/manifests

```yaml
# Example: ApplicationTemplate parameter for storage
parameters:
  - name: storageSize
    type: string
    default: "10Gi"
    ui:
      label: "Storage Size"
      description: "Persistent volume size for data"
```

### Observability

Observability is provided by centralized Cozystack infrastructure:

```
┌─────────────────────────────────────────────────────────────┐
│                   Cozystack Platform                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Centralized Grafana                     │    │
│  │  • Application logs (all Environments)              │    │
│  │  • Metrics dashboards                               │    │
│  │  • Alerting                                         │    │
│  └─────────────────────────────────────────────────────┘    │
│                          ▲                                   │
│                          │ logs/metrics                      │
│  ┌───────────────────────┼───────────────────────────────┐  │
│  │     Environment       │                               │  │
│  │  ┌─────────────────┐  │                               │  │
│  │  │ Monitoring Agent│──┘                               │  │
│  │  └─────────────────┘                                  │  │
│  │  ┌─────────────────┐                                  │  │
│  │  │  Applications   │                                  │  │
│  │  └─────────────────┘                                  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Logging:**
- Application logs collected by agents in Environment cluster
- Shipped to centralized Grafana/Loki
- Tenants access logs via Grafana dashboards

**Debugging:**
- Tenants can use Environment kubeconfig for direct cluster access
- `kubectl exec` into application containers
- `kubectl port-forward` for local debugging
- `kubectl logs` for real-time log streaming

Kubeconfig Secret (see above) provides full access to Environment cluster for debugging purposes.

## Consequences

### Positive

- Tenants get simple abstraction, no need to understand Kubernetes cluster management
- Platform controls cluster configuration and required controllers
- Health monitoring and resource visibility built-in
- Multiple environments per tenant (dev/staging/prod patterns)
- Clean separation: Environment manages infrastructure, Application manages workloads
- Reuses Cozystack multi-tenancy — no additional isolation implementation needed
- Tenant ServiceAccount provides natural access boundary

### Negative

- Additional abstraction layer adds complexity
- Fixed controller set may not fit all use cases (but keeps platform simple)
- 30s health check interval may be too slow for some scenarios

### Risks

- Cozystack API dependency — if Cozystack changes, Environment controller needs updates
- Cluster provisioning can be slow (minutes) — users need clear feedback
- Resource tracking may become stale if cluster is modified outside of Environment
- Cozystack multi-tenancy model changes would require updates to this layer
