# ADR-0001: CRD Design

## Status

Proposed

## Context

Cozy User Apps needs a universal abstraction for managing applications in tenant environments. The current Cozystack model uses one CRD per application type, which is expensive: CRDs are cluster-wide, flood the API server, and require cluster-admin privileges for tenants.

We need two abstractions:
- A template that describes how to install/manage an application
- An instance that represents a deployed application

Both resources live in the host cluster. The controller creates Argo Workflow resources in the target tenant environment.

## Decision

### ApplicationTemplate

Cluster-scoped resource on the host cluster. Read-only for tenants. Describes application lifecycle via hooks.

```yaml
apiVersion: apps.cozystack.io/v1alpha1
kind: ApplicationTemplate
metadata:
  name: minecraft
spec:
  # Human-readable metadata
  displayName: "Minecraft Server"
  description: "Vanilla Minecraft Java Edition server"
  icon: "https://example.com/minecraft.png"

  # Parameter schema (flat, not nested)
  parameters:
    - name: serverName
      type: string
      required: true
      default: "My Server"
      ui:
        label: "Server Name"
        description: "Display name for your Minecraft server"
        placeholder: "Enter server name"
        order: 1

    - name: maxPlayers
      type: int
      required: false
      default: 20
      ui:
        label: "Max Players"
        description: "Maximum number of concurrent players"
        min: 1
        max: 100
        order: 2

    - name: enableWhitelist
      type: bool
      required: false
      default: false
      ui:
        label: "Enable Whitelist"
        description: "Only allow whitelisted players to join"
        order: 3

    - name: gameMode
      type: enum
      required: true
      default: "survival"
      enum:
        - survival
        - creative
        - adventure
        - spectator
      ui:
        label: "Game Mode"
        description: "Default game mode for players"
        order: 4
        widget: dropdown  # dropdown | radio

    - name: allowedIPs
      type: list
      items:
        type: string
      required: false
      minItems: 0
      maxItems: 50
      ui:
        label: "Allowed IPs"
        description: "List of IP addresses allowed to connect"
        order: 5

    - name: dbCredentials
      type: ref
      ref:
        kind: Secret
      required: false
      ui:
        label: "Database Credentials"
        description: "Secret containing database connection details"
        order: 6

    - name: appConfig
      type: ref
      ref:
        kind: ConfigMap
      required: false
      ui:
        label: "Application Config"
        description: "ConfigMap with additional configuration"
        order: 7

  # Lifecycle hooks (called by platform)
  hooks:
    install:
      steps:
        - name: deploy
          image: alpine/helm:3.14
          args:
            - helm
            - install
            - "{{ .App.Name }}"
            - "{{ .Template.ChartURL }}"
            - --namespace
            - "{{ .App.Namespace }}"
            - --set
            - "serverName={{ .Params.serverName }}"

    upgrade:
      steps:
        - name: upgrade
          image: alpine/helm:3.14
          args:
            - helm
            - upgrade
            - "{{ .App.Name }}"
            - "{{ .Template.ChartURL }}"
            - --namespace
            - "{{ .App.Namespace }}"

    remove:
      steps:
        - name: uninstall
          image: alpine/helm:3.14
          args:
            - helm
            - uninstall
            - "{{ .App.Name }}"
            - --namespace
            - "{{ .App.Namespace }}"

  # Custom actions (user-triggered via Action CR)
  actions:
    - name: restart
      displayName: "Restart Server"
      description: "Restart the Minecraft server process"
      # Action-specific parameters (same schema as template parameters)
      parameters:
        - name: gracePeriod
          type: int
          required: false
          default: 30
          ui:
            label: "Grace Period"
            description: "Seconds to wait before force kill"
      steps:
        - name: restart
          image: bitnami/kubectl:latest
          args:
            - kubectl
            - rollout
            - restart
            - deployment/{{ .App.Name }}
            - --namespace
            - "{{ .App.Namespace }}"

    - name: backup
      displayName: "Create Backup"
      description: "Create a backup of world data"
      parameters:
        - name: backupName
          type: string
          required: true
          ui:
            label: "Backup Name"
            description: "Name for this backup"
      steps:
        - name: backup
          image: restic/restic:latest
          args:
            - restic
            - backup
            - /data
            - --tag
            - "{{ .Action.Params.backupName }}"
```

### Application

Namespace-scoped resource on the host cluster. Represents a deployed instance of an ApplicationTemplate in a target environment.

```yaml
apiVersion: apps.cozystack.io/v1alpha1
kind: Application
metadata:
  name: my-minecraft
  namespace: tenant-acme
spec:
  # Reference to template (name only, no version)
  templateRef: minecraft

  # Target environment (abstraction over k8s cluster)
  targetEnvironment: production

  # User-provided parameter values
  values:
    serverName: "ACME Gaming"
    maxPlayers: 50
    enableWhitelist: true
    gameMode: survival
    allowedIPs:
      - "192.168.1.0/24"
      - "10.0.0.0/8"
    dbCredentials:
      name: minecraft-db-secret
    appConfig:
      name: minecraft-config

status:
  # Current state
  phase: Running  # Pending | Installing | Running | Upgrading | Failed | Removing

  # Conditions for detailed status
  conditions:
    - type: Ready
      status: "True"
      lastTransitionTime: "2024-01-15T10:30:00Z"
      reason: InstallSucceeded
      message: "Application successfully installed"

    - type: WorkflowReady
      status: "True"
      lastTransitionTime: "2024-01-15T10:29:00Z"
      reason: WorkflowCompleted
      message: "Install workflow completed successfully"

  # Reference to current/last workflow
  currentWorkflow:
    name: my-minecraft-install-abc123
    namespace: argo-workflows
    phase: Succeeded

  # Observed generation for reconciliation
  observedGeneration: 1
```

### Parameter Types

| Type | Description | Example |
|------|-------------|---------|
| `string` | UTF-8 string | `"My Server"` |
| `int` | Integer number | `20` |
| `bool` | Boolean | `true` |
| `enum` | One of predefined values | `"survival"` |
| `list` | Array of base type items (supports `minItems`/`maxItems` constraints) | `["192.168.1.0/24", "10.0.0.0/8"]` |
| `ref` | Reference to Secret or ConfigMap in same namespace (name only) | `{name: "my-secret"}` |

### UI Hints

Each parameter can have a `ui` block with rendering hints:

| Field | Type | Description |
|-------|------|-------------|
| `label` | string | Human-readable field label |
| `description` | string | Help text for the field |
| `placeholder` | string | Placeholder text for input |
| `order` | int | Display order in UI |
| `min` | int | Minimum value (for int type) |
| `max` | int | Maximum value (for int type) |
| `widget` | string | Widget type for enum: `dropdown` or `radio` |

### Hooks

Platform calls these hooks at specific lifecycle points:

| Hook | When Called |
|------|-------------|
| `install` | Application created |
| `upgrade` | Application spec changed |
| `remove` | Application deleted |

### Step Specification

Steps are a subset of Argo Workflows container spec:

```yaml
steps:
  - name: step-name           # Required: unique step identifier
    image: alpine:3.19        # Required: container image
    args: [...]               # Optional: command arguments
    command: [...]            # Optional: override entrypoint
    env:                      # Optional: environment variables
      - name: FOO
        value: bar
    workingDir: /app          # Optional: working directory
    dependsOn:                # Optional: DAG dependencies
      - previous-step
```

Platform automatically provides:
- `/platform/kubeconfig` — kubeconfig to access host Cozystack tenant (for ordering managed services: Postgres, S3, etc.)
- `/platform/values.yaml` — all Application values
- `/platform/app.yaml` — Application metadata (name, namespace, etc.)
- ServiceAccount with cluster-admin privileges in the Environment cluster (where workflow runs)

### Actions

Custom user-triggered operations defined in `spec.actions`. Each action:
- Has its own parameters (same schema as template parameters)
- Has its own steps (same format as hooks)
- Is triggered via Action CR

### Action CR

```yaml
apiVersion: apps.cozystack.io/v1alpha1
kind: Action
metadata:
  name: my-minecraft-backup-20240115
  namespace: tenant-acme
spec:
  # Reference to target application
  applicationRef: my-minecraft

  # Action name from ApplicationTemplate.spec.actions
  action: backup

  # Action-specific parameter values
  values:
    backupName: "before-update"

status:
  phase: Succeeded  # Pending | Running | Succeeded | Failed

  # Reference to Argo Workflow
  workflowRef:
    name: my-minecraft-backup-20240115-xyz
    namespace: argo-workflows
    phase: Succeeded

  startedAt: "2024-01-15T10:30:00Z"
  completedAt: "2024-01-15T10:35:00Z"
```

**Retention policy:** Completed Action CRs are automatically deleted after 24 hours.

### Environment

Environment is an abstraction over a Kubernetes cluster with:
- Strong autoscaler
- Required controllers for apps (Argo Workflows, Argo Events, etc.)

Application references environment by name via `spec.targetEnvironment`.

## Consequences

### Positive

- Single CRD pair for all application types (no CRD proliferation)
- Clear separation: templates are cluster-wide, instances are namespace-scoped
- Host cluster stays clean — workflows execute in target environments
- Flat parameter schema is simple to validate and render in UI
- Ref type allows secure secret/configmap passing without exposing values
- Actions provide extensible user-triggered operations without hardcoding
- Step spec is subset of Argo Workflows — familiar to operators, easy to map

### Negative

- No nested parameters — complex configurations may need multiple flat params
- Template versioning deferred — upgrades are manual image bumps for now
- Environment abstraction needs separate design (out of scope for this ADR)

### Risks

- Argo Workflows dependency in every environment adds operational overhead
- Template changes affect all applications using that template (no pinning)
- Action CR creates additional resources to track — may need cleanup policy
