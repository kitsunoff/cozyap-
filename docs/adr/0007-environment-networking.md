# ADR-0007: Environment Networking

## Status

Proposed

## Context

Applications running in Environment clusters need external access. Users need to expose their applications via HTTP(S) ingress and TCP/UDP load balancers without understanding underlying infrastructure.

## Decision

### Networking Components in Environment

Each Environment cluster includes networking controllers managed by the platform:

```
┌─────────────────────────────────────────────────────────┐
│                    Environment Cluster                   │
├─────────────────────────────────────────────────────────┤
│  Platform-managed (installed during provisioning):       │
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │ Ingress         │  │ Cloud Controller│               │
│  │ Controller      │  │ Manager (CCM)   │               │
│  │ (e.g., nginx)   │  │                 │               │
│  └─────────────────┘  └─────────────────┘               │
│                                                          │
│  User applications:                                      │
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │ App namespace   │  │ App namespace   │               │
│  │ ┌─────────────┐ │  │ ┌─────────────┐ │               │
│  │ │ Deployment  │ │  │ │ Deployment  │ │               │
│  │ │ Service     │ │  │ │ Service     │ │               │
│  │ │ Ingress     │ │  │ │ Ingress     │ │               │
│  │ └─────────────┘ │  │ └─────────────┘ │               │
│  └─────────────────┘  └─────────────────┘               │
└─────────────────────────────────────────────────────────┘
```

### Ingress Controller

Platform installs and manages Ingress Controller in each Environment:

| Component | Purpose |
|-----------|---------|
| Ingress Controller | HTTP(S) routing, TLS termination |
| Cert-Manager | Automatic TLS certificates (Let's Encrypt) |

Applications create standard Kubernetes Ingress resources:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: my-app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - my-app.tenant-acme.example.com
      secretName: my-app-tls
  rules:
    - host: my-app.tenant-acme.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

### Load Balancer Services

Environment supports `type: LoadBalancer` services for non-HTTP traffic:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minecraft
  namespace: minecraft
spec:
  type: LoadBalancer
  ports:
    - port: 25565
      targetPort: 25565
      protocol: TCP
  selector:
    app: minecraft
```

Cloud Controller Manager provisions external IP from underlying infrastructure (Cozystack managed).

### DNS Management

DNS is handled at Environment level:

```
┌─────────────────┐     ┌─────────────────┐
│  Environment    │     │   DNS Provider  │
│  ┌───────────┐  │     │                 │
│  │ External  │  │     │  *.env.tenant.  │
│  │ DNS       │──┼────▶│  example.com    │
│  │ Controller│  │     │                 │
│  └───────────┘  │     └─────────────────┘
└─────────────────┘
```

Options:
- Wildcard DNS for Environment (`*.production.tenant-acme.example.com`)
- Per-application DNS records via External-DNS controller

### Application Namespace Isolation

Each application runs in its own namespace within Environment:

```
Environment Cluster
├── argo-workflows/        # Platform namespace
├── cert-manager/          # Platform namespace
├── ingress-nginx/         # Platform namespace
├── my-minecraft/          # App namespace
├── my-webapp/             # App namespace
└── my-database/           # App namespace
```

Network policies are NOT enforced between namespaces (for now). Applications can communicate freely within Environment.

### Template Networking Hints

ApplicationTemplate can specify networking requirements:

```yaml
apiVersion: apps.cozystack.io/v1alpha1
kind: ApplicationTemplate
metadata:
  name: minecraft
spec:
  parameters:
    - name: enableIngress
      type: bool
      default: false
      ui:
        label: "Enable Web Map"
        description: "Expose Dynmap web interface"

    - name: hostname
      type: string
      required: false
      ui:
        label: "Hostname"
        description: "Custom hostname for web access"
        placeholder: "map.example.com"

  hooks:
    install:
      steps:
        - name: deploy
          image: alpine/helm:3.14
          args:
            - helm
            - install
            - "{{ .App.Name }}"
            - minecraft/minecraft
            - --set
            - "ingress.enabled={{ .Params.enableIngress }}"
            - --set
            - "ingress.hostname={{ .Params.hostname }}"
```

### Fixed Controller Set (Updated)

From ADR-0002, Environment controllers now include networking:

| Controller | Purpose |
|------------|---------|
| Argo Workflows | DAG execution engine |
| Argo Events | Event-driven triggers |
| Ingress Controller | HTTP(S) routing |
| Cert-Manager | TLS certificate management |
| External-DNS | DNS record management (optional) |
| CCM | Load balancer provisioning |

## Consequences

### Positive

- Standard Kubernetes networking primitives (Ingress, Service)
- Automatic TLS with Let's Encrypt
- Load balancer support for non-HTTP protocols
- Each app in separate namespace for organization

### Negative

- Additional controllers increase Environment overhead
- No network isolation between apps (future enhancement)
- DNS configuration depends on infrastructure

### Risks

- Load balancer IP exhaustion in large deployments
- Certificate rate limits from Let's Encrypt
- DNS propagation delays affect user experience
