
You are a presentation designer. Create an HTML presentation for Cozy User Apps —
an Application Platform built on top of Cozystack.

Target audience: technical people (DevOps, SRE, architects) evaluating the platform.
They understand Kubernetes but want to see value proposition and architecture
at a high level.

---

## FORMAT

- Single HTML file with all slides
- Each slide is a full viewport section (100vh)
- Navigation: keyboard arrows or scroll
- 19 slides in the specified order
- Replace ASCII diagrams with visual schemas (SVG or CSS boxes)

---

## DESIGN SYSTEM

### Colors

```css
--bg-slide:       #1a1a1e;
--bg-card:        #222226;
--bg-elevated:    #2a2a2e;
--bg-code:        #16161a;
--text-primary:   #e8e8ea;
--text-secondary: #9999a8;
--text-muted:     #666672;
--border:         rgba(255,255,255,0.08);
--accent-blue:    #4c7ef3;
--accent-green:   #3ecf8e;
--accent-amber:   #f5a623;
--accent-red:     #f87171;
--accent-purple:  #a78bfa;
```

### Typography

- Font: system-ui / -apple-system / Inter
- Slide title: 42px, font-weight 700, color text-primary
- Slide subtitle: 20px, font-weight 400, color text-secondary
- Section header: 24px, font-weight 600
- Body text: 18px, line-height 1.6
- Code/monospace: 14px, font-family monospace, bg bg-code, padding 2px 6px, border-radius 4px
- List items: 18px, line-height 2

### Slide Components

**Slide container**:
- Background: bg-slide
- Padding: 80px
- Height: 100vh
- Display: flex, align-items: center, justify-content: center
- Max content width: 1200px

**Title slide**:
- Logo: "COZY≡TACK" + "User Apps", font-weight 700, letter-spacing 0.1em
- Centered vertically and horizontally
- Subtitle below the title

**Content slide**:
- Title at top left
- Content area: max-width 1000px

**Diagram box** (for architecture diagrams):
- Background: bg-card
- Border: 1px solid border
- Border-radius: 12px
- Padding: 24px
- Nested boxes with bg-elevated

**Code block**:
- Background: bg-code
- Border-radius: 8px
- Padding: 20px
- Font: monospace 14px
- Syntax highlighting: keywords accent-blue, strings accent-green, comments text-muted

**Table**:
- Header: bg-elevated, font-weight 600
- Rows: border-bottom 1px border
- Cells: padding 12px 16px
- Hover: bg-elevated

**Badge/Tag**:
- Height: 24px
- Padding: 0 10px
- Border-radius: 6px
- Font-size: 13px, font-weight 500
- Colors by type (see below)

**Status badges**:
- Running/Ready:  bg rgba(62,207,142,0.15),  color #3ecf8e
- Installing:     bg rgba(76,126,243,0.15),  color #4c7ef3
- Failed:         bg rgba(248,113,113,0.15), color #f87171

**Flow diagram** (for user flows):
- Boxes: bg-card, border-radius 8px, padding 16px
- Arrows: 2px solid text-muted, with triangle at the end
- Horizontal layout with gap 24px

**Two-column layout**:
- Gap: 48px
- Left column: 60%
- Right column: 40%

**Bullet list**:
- Custom bullet: accent-blue circle 6px
- Indent: 24px
- Line-height: 2

**Numbered list**:
- Number: accent-blue, font-weight 600
- Same spacing as bullets

**Highlight box** (for key takeaways):
- Background: rgba(76,126,243,0.1)
- Border-left: 4px solid accent-blue
- Padding: 16px 20px
- Border-radius: 0 8px 8px 0

**Icon placeholders**:
- Use emoji for icons: 📦 🔧 🚀 ✅ ⚠️ 🎯 🔒 📊

---

## SLIDE CONTENT

### Slide 1: Title

**COZY≡TACK User Apps**

Application Platform for Cozystack

*(centered minimalist title slide)*

---

### Slide 2: Problem

**The Challenge**

Current Cozystack model limitations:

- ❌ One CRD per application type (Postgres, Redis, Minecraft...)
- ❌ CRDs are cluster-wide — flood the API server
- ❌ Adding new app requires cluster-admin privileges
- ❌ Tenants cannot create their own application types

**Result:** Scaling application catalog is expensive and inflexible.

---

### Slide 3: Solution

**Universal Application Abstraction**

Two resources replace hundreds of CRDs:

| Resource | Scope | Purpose |
|----------|-------|---------|
| **ApplicationTemplate** | Cluster | How to install/manage an app |
| **Application** | Namespace | Deployed instance |

**Benefits:**
- ✅ One CRD pair for ALL application types
- ✅ Templates are reusable across tenants
- ✅ No cluster-admin for new apps

---

### Slide 4: Core Concepts

**Diagram showing relationship:**

```
┌─────────────────────────────────────┐
│       ApplicationTemplate           │
│  (cluster-scoped, read-only)        │
│                                     │
│  • Parameters schema                │
│  • Lifecycle hooks                  │
│  • Custom actions                   │
└──────────────┬──────────────────────┘
               │ references
               ▼
┌─────────────────────────────────────┐
│          Application                │
│  (namespace-scoped, per tenant)     │
│                                     │
│  • Template reference               │
│  • Target environment               │
│  • User values                      │
└─────────────────────────────────────┘
```

*(visualize as two connected boxes with an arrow)*

---

### Slide 5: Environment Abstraction

**Environment = Managed Kubernetes Cluster**

```
Tenant Namespace (host cluster)
│
├── Environment: production
│   └── → ManagedKubernetes cluster
│
├── Application: my-minecraft
│   └── → runs in production
│
└── Application: my-webapp
    └── → runs in production
```

**What tenant gets:**
- Real Kubernetes cluster
- Pre-installed controllers
- Automatic scaling
- Health monitoring

---

### Slide 6: Architecture Overview

**Diagram:**

```
┌────────────────────────────────────────────┐
│              Host Cluster                   │
│  ┌──────────────────────────────────────┐  │
│  │        cozy-apps-operator            │  │
│  │  • Environment Controller            │  │
│  │  • Application Controller            │  │
│  │  • Action Controller                 │  │
│  └──────────────┬───────────────────────┘  │
│                 │ creates workflows         │
│                 ▼                           │
│  ┌──────────────────────────────────────┐  │
│  │       Environment Cluster            │  │
│  │  • Argo Workflows                    │  │
│  │  • Ingress Controller                │  │
│  │  • User Applications                 │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

*(visualize as nested blocks)*

---

### Slide 7: Install Flow

**How deployment works:**

```
User → Application CR → Controller → Argo Workflow → Helm/kubectl → App Running
```

**Steps:**
1. User creates Application (UI or kubectl)
2. Controller validates template & parameters
3. Controller creates Argo Workflow in Environment
4. Workflow executes install hook
5. Application status → Running
6. User accesses via Ingress/LoadBalancer

---

### Slide 8: Template Example

**ApplicationTemplate: Minecraft**

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
    - name: maxPlayers
      type: int
      default: 20

  hooks:
    install:
      steps:
        - name: deploy
          image: alpine/helm:3.14
          args: [helm, install, ...]

  actions:
    - name: backup
      displayName: "Create Backup"
      steps: [...]
```

---

### Slide 9: User Personas

**Four personas, different needs:**

| Persona | What they do | Technical level |
|---------|--------------|-----------------|
| 👤 **End User** | Deploy apps from catalog | Low (UI only) |
| 📝 **Template Author** | Create templates | High (K8s, Helm) |
| 🔧 **ISP Operator** | Manage platform | High (K8s admin) |
| 💻 **Developer** | Use as dev platform | Medium (CI/CD) |

---

### Slide 10: End User Flow

**Deploy app without Kubernetes knowledge:**

```
[Browse Catalog] → [Select App] → [Fill Form] → [Deploy]
                                                    │
                                                    ▼
[Access App] ← [App Running] ← [Watch Progress] ← [Install]
```

**No YAML. No kubectl. Just click.**

---

### Slide 11: Template Author Flow

**Create new template:**

1. Study how app is normally installed
2. Define parameters (what users configure)
3. Write install/upgrade/remove hooks
4. Add custom actions (backup, restart...)
5. Test in dev environment
6. Submit to catalog

**Hooks are container steps** — use any tool:
Helm, kubectl, Ansible, custom scripts

---

### Slide 12: Multi-Tenancy

**Reuses Cozystack multi-tenancy:**

```
┌─────────────────────────────────────┐
│         Cozystack Cluster           │
│                                     │
│  ┌─────────────┐  ┌─────────────┐  │
│  │ tenant-acme │  │tenant-globex│  │
│  │             │  │             │  │
│  │ Environment │  │ Environment │  │
│  │ Application │  │ Application │  │
│  │ Action      │  │ Action      │  │
│  └─────────────┘  └─────────────┘  │
└─────────────────────────────────────┘
```

- Tenant = Namespace
- Full access within namespace
- No cross-tenant access
- Templates shared (read-only)

---

### Slide 13: Networking

**Each Environment includes:**

| Component | Purpose |
|-----------|---------|
| Ingress Controller | HTTP/HTTPS routing |
| Cert-Manager | Auto TLS (Let's Encrypt) |
| Cloud Controller | LoadBalancer provisioning |

**Standard Kubernetes primitives:**
- `Ingress` for HTTP
- `Service type: LoadBalancer` for TCP/UDP

---

### Slide 14: Observability & Debugging

**Centralized:**
- 📊 Grafana for logs & metrics
- 📈 Dashboards per tenant

**Direct access:**
- 🔑 Kubeconfig per Environment
- Standard kubectl:
  - `kubectl exec`
  - `kubectl port-forward`
  - `kubectl logs`

---

### Slide 15: Custom Actions

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
        args: [backup, /data]
```

**Flow:** User triggers → Action CR → Workflow executes

---

### Slide 16: Template Catalog

**Storage: Aggregated API Server + PostgreSQL**

```
┌─────────────────────────────────┐
│       Kubernetes API            │
│  ┌───────────────────────────┐  │
│  │  cozy-apps-apiserver      │  │
│  │  (aggregated)             │  │
│  └─────────────┬─────────────┘  │
└────────────────┼────────────────┘
                 │
                 ▼
        ┌────────────────┐
        │   PostgreSQL   │
        │  (versioned    │
        │   templates)   │
        └────────────────┘
```

- Version history
- Native K8s API
- Scales independently

---

### Slide 17: Key Benefits

| For End Users | For Operators | For Platform |
|---------------|---------------|--------------|
| ✅ Simple UI | ✅ Controlled templates | ✅ Single CRD pair |
| ✅ No K8s knowledge | ✅ Multi-tenant | ✅ Scalable catalog |
| ✅ Self-service | ✅ Central monitoring | ✅ Extensible |

---

### Slide 18: Summary

**Cozy User Apps transforms Cozystack into Application Platform:**

1. **Universal abstraction** — one template for any app
2. **Self-service** — users deploy without K8s expertise
3. **Extensible** — community contributes templates
4. **Isolated** — multi-tenant by design
5. **Observable** — centralized logging & debugging

---

### Slide 19: Q&A

**Questions?**

*(minimalist slide with logo and contacts)*

---

## TECHNICAL REQUIREMENTS

### HTML Structure

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Cozy User Apps — Presentation</title>
  <style>/* design system */</style>
</head>
<body>
  <div class="presentation">
    <section class="slide" id="slide-1">...</section>
    <section class="slide" id="slide-2">...</section>
    <!-- ... -->
  </div>
  <script>/* navigation */</script>
</body>
</html>
```

### Navigation

```js
// Scroll snap for slides
.presentation {
  scroll-snap-type: y mandatory;
  overflow-y: scroll;
  height: 100vh;
}

.slide {
  scroll-snap-align: start;
  height: 100vh;
}

// Keyboard: up/down arrows
document.addEventListener('keydown', (e) => {
  if (e.key === 'ArrowDown' || e.key === ' ') nextSlide();
  if (e.key === 'ArrowUp') prevSlide();
});
```

### Diagrams

Replace ASCII diagrams with CSS boxes:

```html
<div class="diagram">
  <div class="box box--primary">
    <div class="box__title">ApplicationTemplate</div>
    <div class="box__content">cluster-scoped</div>
  </div>
  <div class="arrow arrow--down"></div>
  <div class="box box--secondary">
    <div class="box__title">Application</div>
    <div class="box__content">namespace-scoped</div>
  </div>
</div>
```

### Progress Indicator

Thin line at the bottom of the screen showing current slide / total:

```css
.progress {
  position: fixed;
  bottom: 0;
  left: 0;
  height: 3px;
  background: var(--accent-blue);
  transition: width 0.3s;
}
```

---

## IMPORTANT

- No external dependencies (vanilla HTML/CSS/JS only)
- All styles inline or in `<style>` block
- Presentation must work offline
- Responsive: minimum 1280px width
- Print: `@media print` for PDF export
