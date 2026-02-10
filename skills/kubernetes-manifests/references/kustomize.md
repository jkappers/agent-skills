# Kustomize Deep Dive

Template-free configuration customization built into kubectl. Customize raw YAML without forking, using overlays and patches.

## Table of Contents

- [Core Concepts](#core-concepts)
- [Directory Structure](#directory-structure)
- [Base Configuration](#base-configuration)
- [Overlays](#overlays)
- [Generators](#generators)
- [Patches](#patches)
- [Transformers](#transformers)
- [Components](#components)
- [Replacements](#replacements)
- [GitOps Integration](#gitops-integration)
- [Anti-Patterns](#anti-patterns)
- [Validation](#validation)

## Core Concepts

Kustomize works with plain YAML — no template language, no special syntax. Files remain valid Kubernetes manifests.

**Base and Overlays Pattern**: Common configuration (base) separated from environment-specific modifications (overlays). The base remains untouched; customizations layer on top.

```
base/          → Environment-agnostic resources
overlays/dev/  → Dev-specific patches
overlays/prod/ → Prod-specific patches
```

## Directory Structure

```
manifests/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
├── components/
│   ├── monitoring/
│   │   ├── servicemonitor.yaml
│   │   └── kustomization.yaml
│   └── caching/
│       ├── redis.yaml
│       └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── ingress-patch.yaml
    └── production/
        ├── kustomization.yaml
        ├── replica-patch.yaml
        ├── resource-patch.yaml
        └── hpa.yaml
```

| Practice | Description |
|----------|-------------|
| Keep base generic | No hardcoded environment values |
| One overlay per environment | Separate dev, staging, production |
| Consistent naming | `replica-patch.yaml` not `patch1.yaml` |
| Group related resources | Keep deployment + service + configmap together |
| Use components for features | Package optional features as reusable components |

## Base Configuration

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

labels:
- pairs:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/managed-by: kustomize
  includeSelectors: true

commonAnnotations:
  app.kubernetes.io/part-of: my-platform
```

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:latest  # Tag overridden via images transformer
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
```

## Overlays

### Development Overlay

```yaml
# overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

namespace: myapp-dev
namePrefix: dev-

images:
- name: myapp
  newName: myregistry/myapp
  newTag: dev-latest

patches:
- path: replica-patch.yaml

configMapGenerator:
- name: app-config
  behavior: merge
  literals:
  - LOG_LEVEL=debug
  - ENABLE_PROFILING=true
```

### Production Overlay

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base
- hpa.yaml
- pdb.yaml

namespace: myapp-prod
namePrefix: prod-

images:
- name: myapp
  newName: myregistry/myapp
  newTag: v1.2.3

patches:
- path: replica-patch.yaml
- path: resource-patch.yaml
- path: security-patch.yaml

labels:
- pairs:
    environment: production
    tier: backend

configMapGenerator:
- name: app-config
  behavior: merge
  literals:
  - LOG_LEVEL=warning
  - ENABLE_PROFILING=false
```

## Generators

### ConfigMapGenerator

```yaml
configMapGenerator:
# From literals
- name: app-config
  literals:
  - DATABASE_HOST=postgres.default.svc
  - CACHE_TTL=3600

# From files
- name: nginx-config
  files:
  - nginx.conf

# From env file
- name: env-config
  envs:
  - .env.production
```

### SecretGenerator

Use external secret managers in production. SecretGenerator for development only:

```yaml
secretGenerator:
- name: app-secrets
  literals:
  - username=admin
  - password=secret123
- name: tls-secrets
  files:
  - tls.crt
  - tls.key
  type: kubernetes.io/tls
```

### Hash Suffix Behavior

Generators append a content hash by default (e.g., `app-config-8mbdf7882g`). When content changes, the hash changes, triggering Deployment rollouts. Disable only when managing rollouts manually:

```yaml
generatorOptions:
  disableNameSuffixHash: true
```

## Patches

### Strategic Merge Patches

Best for adding or modifying fields:

```yaml
patches:
- path: resource-patch.yaml
```

```yaml
# resource-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
```

Or inline:

```yaml
patches:
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: myapp
    spec:
      replicas: 3
```

### JSON Patches (RFC 6902)

Best for precise operations, deletions, and list manipulation:

```yaml
patches:
- target:
    kind: Deployment
    name: myapp
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
    - op: add
      path: /spec/template/spec/containers/0/env/-
      value:
        name: NEW_VAR
        value: "new-value"
    - op: remove
      path: /spec/template/spec/containers/0/resources/limits/cpu
```

### Targeting Multiple Resources

```yaml
patches:
- target:
    kind: Deployment
    labelSelector: "app=myapp"
  patch: |-
    - op: add
      path: /spec/template/spec/securityContext
      value:
        runAsNonRoot: true
        runAsUser: 1000
```

### Deleting Resources from Base

```yaml
patches:
- patch: |-
    $patch: delete
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: unwanted-config
```

| Patch Type | Best For |
|------------|----------|
| Strategic Merge | Adding/merging fields, large changes |
| JSON Patch (RFC 6902) | Precise operations, deletions, list manipulation |

## Transformers

### Images

```yaml
images:
- name: myapp
  newName: myregistry.azurecr.io/myapp
  newTag: v2.0.0
- name: critical-app
  newName: myregistry/critical-app
  digest: sha256:abc123def456...
```

### Namespace, Name Prefix/Suffix

```yaml
namespace: production
namePrefix: prod-
nameSuffix: -v2
```

### Labels

```yaml
labels:
- pairs:
    environment: production
    team: platform
  includeSelectors: true
```

### Replicas

```yaml
replicas:
- name: myapp
  count: 5
- name: worker
  count: 3
```

## Components

Reusable configuration units for optional features (Kustomize v3.7.0+):

```yaml
# components/monitoring/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
- servicemonitor.yaml

patches:
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: myapp
    spec:
      template:
        metadata:
          annotations:
            prometheus.io/scrape: "true"
            prometheus.io/port: "8080"
```

Include selectively per overlay:

```yaml
# overlays/production/kustomization.yaml
components:
- ../../components/monitoring
- ../../components/caching
```

| Scenario | Solution |
|----------|----------|
| Feature needed in all environments | Include in base |
| Feature needed in some environments | Use components |
| Feature varies per environment | Use overlays |

## Replacements

Use `replacements` instead of deprecated `vars` (Kustomize v5.0+):

```yaml
replacements:
- source:
    kind: Service
    name: myapp
    fieldPath: metadata.name
  targets:
  - select:
      kind: Deployment
      name: myapp
    fieldPaths:
    - spec.template.spec.containers.[name=myapp].env.[name=SERVICE_NAME].value
```

## GitOps Integration

### ArgoCD

ArgoCD automatically detects Kustomize:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  source:
    repoURL: https://github.com/org/repo
    path: manifests/overlays/production
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

### FluxCD

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: my-repo
  path: ./manifests/overlays/production
  prune: true
  targetNamespace: production
```

**Pin remote bases to specific versions:**

```yaml
# Bad
resources:
- github.com/org/repo//base

# Good
resources:
- github.com/org/repo//base?ref=v1.2.3
```

## Anti-Patterns

1. **Duplicating configuration across overlays** — Keep a single base, patch only differences
2. **Hardcoding environment values in base** — Keep base generic, override in overlays
3. **Using unstable remote references** — Pin to version tags or commit SHAs
4. **Overly complex patches** — If a resource needs major changes per environment, make it environment-specific
5. **Mixing CRDs with instances** — Apply CRDs in a separate kustomization to avoid race conditions
6. **Disabling generator hash suffixes globally** — Use hash suffixes (default) for automatic rollout triggers

## Validation

```bash
# Build and view output
kubectl kustomize overlays/production

# Client-side dry-run
kubectl apply -k overlays/production --dry-run=client

# Server-side dry-run
kubectl apply -k overlays/production --dry-run=server

# Validate with kubeconform
kubectl kustomize overlays/production | kubeconform -strict -summary
```

CI/CD integration:

```yaml
- name: Validate Kustomize manifests
  run: |
    for overlay in manifests/overlays/*/; do
      echo "Validating $overlay"
      kubectl kustomize "$overlay" | kubeconform -strict -summary
    done
```
