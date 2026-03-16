# ADR-0006: Template Catalog Management

## Status

Proposed

## Context

ApplicationTemplates need to be stored, versioned, and made available to tenants. The platform needs a catalog system that:

- Provides a source of truth for available templates
- Supports versioning for template updates
- Integrates with Kubernetes API for native access
- Scales with the number of templates and tenants

## Decision

### Storage Options

Two approaches are considered for storing ApplicationTemplates:

#### Option A: Native etcd (CRD in etcd)

ApplicationTemplate is a standard Kubernetes CRD stored in etcd.

```
┌─────────────────────────────────────────┐
│              Kubernetes API              │
├─────────────────────────────────────────┤
│  ApplicationTemplate CRD                 │
│  ┌─────────────────────────────────────┐│
│  │  etcd storage                       ││
│  │  /registry/apps.cozystack.io/...    ││
│  └─────────────────────────────────────┘│
└─────────────────────────────────────────┘
```

**Pros:**
- Native Kubernetes experience
- No additional infrastructure
- Built-in RBAC, watch, list operations
- Works with kubectl, client-go, controllers

**Cons:**
- etcd has size limits (~1.5MB per object)
- Large number of templates may impact etcd performance
- No built-in versioning (only current state)

#### Option B: Aggregated API Server with PostgreSQL

Custom API server using Kubernetes Aggregation Layer, backed by PostgreSQL.

```
┌─────────────────────────────────────────┐
│              Kubernetes API              │
├─────────────────────────────────────────┤
│  APIService: v1alpha1.apps.cozystack.io │
│  ┌─────────────────────────────────────┐│
│  │  cozy-apps-apiserver                ││
│  │  (aggregated API server)            ││
│  └──────────────┬──────────────────────┘│
└─────────────────┼───────────────────────┘
                  │
                  ▼
         ┌────────────────┐
         │   PostgreSQL   │
         │  (Cozystack    │
         │   managed)     │
         └────────────────┘
```

**Pros:**
- No etcd size limits
- Can store version history natively
- Better query capabilities (SQL)
- Scales independently from etcd
- Uses existing Cozystack PostgreSQL

**Cons:**
- Additional component to maintain
- More complex deployment
- PostgreSQL dependency

### Recommended Approach

**Option B (Aggregated API Server)** is recommended because:

1. Template versioning is critical for production
2. PostgreSQL is already available in Cozystack
3. Avoids etcd pressure from potentially large catalog
4. Enables future features (search, filtering, analytics)

### API Server Design

```yaml
# APIService registration
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1alpha1.apps.cozystack.io
spec:
  group: apps.cozystack.io
  version: v1alpha1
  service:
    name: cozy-apps-apiserver
    namespace: cozy-system
  groupPriorityMinimum: 1000
  versionPriority: 100
```

The aggregated API server handles:
- `ApplicationTemplate` — cluster-scoped, read-only for tenants
- Potentially `TemplateCatalog` — for catalog sources (future)

Other resources (`Application`, `Environment`, `Action`) remain as standard CRDs in etcd.

### Template Versioning

Each ApplicationTemplate has immutable versions stored in PostgreSQL:

```sql
CREATE TABLE application_templates (
    id UUID PRIMARY KEY,
    name VARCHAR(253) NOT NULL,
    version VARCHAR(64) NOT NULL,
    spec JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL,
    deprecated BOOLEAN DEFAULT FALSE,
    UNIQUE(name, version)
);

CREATE INDEX idx_templates_name ON application_templates(name);
CREATE INDEX idx_templates_version ON application_templates(name, version);
```

### Version Selection

Application references template by name only (no version pinning in v1):

```yaml
apiVersion: apps.cozystack.io/v1alpha1
kind: Application
metadata:
  name: my-minecraft
spec:
  templateRef: minecraft  # Always uses latest version
```

Future enhancement: version pinning via `templateRef: minecraft@v1.2.3`.

### Template Lifecycle

```
Template Author submits new version
        │
        ▼
┌───────────────────┐
│  Version created  │
│  (latest = true)  │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  Previous version │
│  (latest = false) │
└─────────┬─────────┘
          │ Optional: mark deprecated
          ▼
┌───────────────────┐
│    Deprecated     │
│  (hidden in UI)   │
└───────────────────┘
```

### Catalog Sources (Future)

For now, templates are manually created via API. Future enhancement:

```yaml
apiVersion: apps.cozystack.io/v1alpha1
kind: TemplateCatalog
metadata:
  name: community
spec:
  source:
    type: git
    git:
      url: https://github.com/cozystack/templates.git
      branch: main
      path: templates/
  syncInterval: 1h
```

## Consequences

### Positive

- Native Kubernetes API experience preserved
- Version history enables rollback and audit
- PostgreSQL handles large catalogs efficiently
- Clear separation: catalog in PostgreSQL, instances in etcd

### Negative

- Aggregated API server adds operational complexity
- PostgreSQL becomes critical dependency for template access
- Two storage backends to manage

### Risks

- API server availability affects template access
- PostgreSQL migration complexity if schema changes
- Version compatibility between API server and controllers
