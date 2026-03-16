# Cozy User Apps — Presentation Flow

## Slide 1: Title

**Cozy User Apps**
Application Platform for Cozystack

---

## Slide 2: Problem

**Current Cozystack Model Limitations:**

- One CRD per application type (Postgres, Redis, Minecraft, etc.)
- CRDs are cluster-wide resources — flood the API server
- Adding new application requires cluster-admin privileges
- Tenants cannot create their own application types
- Template authors need deep Kubernetes expertise

**Result:** Scaling application catalog is expensive and inflexible.

---

## Slide 3: Solution

**Cozy User Apps — Universal Application Abstraction**

Two simple resources replace hundreds of CRDs:

| Resource | Scope | Purpose |
|----------|-------|---------|
| ApplicationTemplate | Cluster | Describes how to install/manage an app |
| Application | Namespace | Deployed instance of a template |

**Benefits:**
- One CRD pair for ALL application types
- Templates are reusable across tenants
- No cluster-admin needed for new apps

---

## Slide 4: Core Concepts

```
┌─────────────────────────────────────────────────────────┐
│                 ApplicationTemplate                      │
│  (cluster-scoped, read-only for tenants)                │
│                                                          │
│  • Display name, description, icon                       │
│  • Parameter schema (forms for UI)                       │
│  • Lifecycle hooks (install, upgrade, remove)           │
│  • Custom actions (backup, restart, etc.)               │
└─────────────────────────────────────────────────────────┘
                          │
                          │ references
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    Application                           │
│  (namespace-scoped, per tenant)                         │
│                                                          │
│  • Template reference                                    │
│  • Target environment                                    │
│  • User-provided parameter values                        │
│  • Status and health                                     │
└─────────────────────────────────────────────────────────┘
```

---

## Slide 5: Environment Abstraction

**Environment = Managed Kubernetes Cluster**

- Tenant creates Environment (dev, staging, production)
- Platform provisions real Kubernetes cluster via Cozystack
- Pre-installed controllers (Argo Workflows, Ingress, Cert-Manager)
- Applications deploy INTO Environment cluster

```
Tenant Namespace (host cluster)
├── Environment: production
│   └── → ManagedKubernetes cluster
├── Application: my-minecraft
│   └── → deployed in production Environment
└── Application: my-webapp
    └── → deployed in production Environment
```

---

## Slide 6: Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Host Cluster                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │            cozy-apps-operator                    │    │
│  │  • Environment Controller                        │    │
│  │  • Application Controller                        │    │
│  │  • Action Controller                             │    │
│  └─────────────────────────────────────────────────┘    │
│                          │                               │
│                          │ creates workflows             │
│                          ▼                               │
│  ┌─────────────────────────────────────────────────┐    │
│  │           Environment Cluster                    │    │
│  │  • Argo Workflows (executes hooks)              │    │
│  │  • Ingress Controller                            │    │
│  │  • User Applications                             │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

---

## Slide 7: How It Works — Install Flow

1. **User creates Application** (via UI or kubectl)
2. **Controller validates** template and parameters
3. **Controller creates Argo Workflow** in Environment cluster
4. **Workflow executes install hook** (Helm install, kubectl apply, etc.)
5. **Application status updates** to Running
6. **User accesses application** via Ingress/LoadBalancer

```
User → Application CR → Controller → Argo Workflow → Helm/kubectl → App Running
```

---

## Slide 8: ApplicationTemplate Example

```yaml
apiVersion: apps.cozystack.io/v1alpha1
kind: ApplicationTemplate
metadata:
  name: minecraft
spec:
  displayName: "Minecraft Server"

  parameters:
    - name: serverName
      type: string
      required: true
      ui:
        label: "Server Name"
    - name: maxPlayers
      type: int
      default: 20
      ui:
        label: "Max Players"
        min: 1
        max: 100

  hooks:
    install:
      steps:
        - name: deploy
          image: alpine/helm:3.14
          args: [helm, install, "{{ .App.Name }}", ...]

  actions:
    - name: backup
      displayName: "Create Backup"
      steps:
        - name: backup
          image: restic/restic
          args: [...]
```

---

## Slide 9: User Personas

| Persona | What they do | Technical level |
|---------|--------------|-----------------|
| **End User** | Deploy apps from catalog | Low (UI only) |
| **Template Author** | Create new templates | High (K8s, Helm) |
| **ISP Operator** | Manage platform | High (K8s admin) |
| **Developer** | Use as dev platform | Medium (CI/CD) |

---

## Slide 10: End User Flow

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Browse  │ →  │ Select  │ →  │  Fill   │ →  │ Deploy  │
│ Catalog │    │  App    │    │  Form   │    │         │
└─────────┘    └─────────┘    └─────────┘    └─────────┘
                                                  │
                                                  ▼
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Access  │ ←  │   App   │ ←  │ Watch   │ ←  │Install  │
│  App    │    │ Running │    │Progress │    │Workflow │
└─────────┘    └─────────┘    └─────────┘    └─────────┘
```

**No Kubernetes knowledge required!**

---

## Slide 11: Template Author Flow

1. Study how application is normally installed
2. Define parameters (what users can configure)
3. Write install/upgrade/remove hooks
4. Add custom actions (backup, restart, etc.)
5. Test in development environment
6. Submit to catalog

**Hooks are just container steps** — use any tool (Helm, kubectl, Ansible, etc.)

---

## Slide 12: Multi-Tenancy

**Reuses Cozystack multi-tenancy model:**

- Tenant = Namespace in Cozystack
- ServiceAccount per tenant with RBAC
- Full access to own resources (Environment, Application, Action)
- No access to other tenants
- ApplicationTemplates are shared (read-only)

**No additional isolation implementation needed.**

---

## Slide 13: Networking

**Each Environment includes:**

| Component | Purpose |
|-----------|---------|
| Ingress Controller | HTTP/HTTPS routing |
| Cert-Manager | Automatic TLS (Let's Encrypt) |
| Cloud Controller | LoadBalancer provisioning |

**Applications use standard Kubernetes primitives:**
- `Ingress` for HTTP traffic
- `Service type: LoadBalancer` for TCP/UDP

---

## Slide 14: Observability & Debugging

**Centralized:**
- Grafana for logs and metrics (all Environments)
- Dashboards per tenant

**Direct access:**
- Kubeconfig available for each Environment
- kubectl exec, port-forward, logs
- Full debugging capabilities

---

## Slide 15: Custom Actions

**Beyond install/upgrade/remove:**

```yaml
actions:
  - name: backup
    displayName: "Create Backup"
    parameters:
      - name: backupName
        type: string
    steps:
      - name: backup
        image: restic/restic
        args: [backup, /data, --tag, "{{ .Action.Params.backupName }}"]
```

**User triggers via UI → Action CR → Workflow executes**

---

## Slide 16: Template Catalog

**Storage:** Aggregated API Server + PostgreSQL

**Features:**
- Version history for templates
- Native Kubernetes API experience
- Scales independently from etcd

**Future:** Sync from Git repositories

---

## Slide 17: Key Benefits

| For End Users | For Operators | For Platform |
|---------------|---------------|--------------|
| Simple UI | Controlled templates | Single CRD pair |
| No K8s knowledge | Multi-tenant isolation | Scalable catalog |
| Self-service | Centralized monitoring | Extensible actions |

---

## Slide 18: Summary

**Cozy User Apps transforms Cozystack into Application Platform:**

1. **Universal abstraction** — one template for any app
2. **Self-service** — users deploy without K8s expertise
3. **Extensible** — community can contribute templates
4. **Isolated** — multi-tenant by design
5. **Observable** — centralized logging and debugging

---

## Slide 19: Q&A

Questions?
