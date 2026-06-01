# GitOps Platform: Helm App-of-Apps Structure

This repository has been refactored into a **Helm-based app-of-apps** architecture. All platform components are now templated and configured through a single centralized values file.

## Overview

- **Bootstrap**: `bootstrap/root-app.yaml` → `bootstrap/app-of-apps` (Helm chart)
- **Centralized Config**: `bootstrap/app-of-apps/values.yaml` contains all OCI IDs, regions, enable/disable flags
- **Templated Manifests**: Individual Helm charts for each infrastructure component
- **Single Source of Truth**: All configuration is managed from one place

## Architecture

```
bootstrap/root-app.yaml (Application CR)
  ↓
bootstrap/app-of-apps/ (Helm chart)
  ├── Chart.yaml
  ├── values.yaml (centralized configuration)
  └── templates/
      ├── appproject.yaml (platform AppProject)
      ├── karpenter.yaml (Karpenter Helm app)
      ├── karpenter-nodepools.yaml (NodePools with templated values)
      ├── crossplane-system.yaml (Crossplane Helm app)
      ├── crossplane-providers.yaml (Crossplane providers)
      ├── crossplane-compositions.yaml (Crossplane compositions)
      ├── gatewayapi-iam.yaml (Gateway API IAM with templated values)
      ├── envoy-gateway.yaml (Envoy Gateway Helm app)
      ├── gateway-objects.yaml (Gateway API objects)
      ├── stacks.yaml (Stack applications)
      └── demo-stack-helm.yaml (Demo stack with templated Redis claim)
```

## Centralized Values

Edit `bootstrap/app-of-apps/values.yaml` to manage:

### OCI Infrastructure
```yaml
oci:
  region: eu-frankfurt-1
  compartmentId: ocid1.compartment...
  
  karpenter:
    nodeCompartmentId: ocid1.compartment...
    primarySubnetId: ocid1.subnet...
    secondarySubnetId: ocid1.subnet...
    primaryNsgId: ocid1.nsg...
    secondaryNsgId: ocid1.nsg...
  
  crossplane:
    compartmentId: ocid1.compartment...
    clusterId: ocid1.cluster...
  
  stacks:
    demo:
      compartmentId: ocid1.compartment...
      redisSubnetId: ocid1.subnet...
      redisNsgId: ocid1.nsg...
```

### Component Enablement
```yaml
components:
  karpenter:
    enabled: true
  crossplaneSystem:
    enabled: true
  gatewayapiIam:
    enabled: true
  # etc.
```

## Templated Manifest Charts

Each infrastructure component has its own Helm chart that receives values from the parent app-of-apps:

### 1. Karpenter NodePools
**Chart**: `manifests/karpenter-nodepools-helm/`
- **Templates**: `nodeclass.yaml` (NodePool + OCINodeClass)
- **Values injected**: compartmentId, subnetIds, NSG IDs, node shapes, limits
- **Application**: Created by `bootstrap/app-of-apps/templates/karpenter-nodepools.yaml`

### 2. Gateway API IAM
**Chart**: `manifests/gatewayapi-iam/`
- **Templates**: `policy.yaml` (OCI Identity Policy)
- **Values injected**: compartmentId, compartment name
- **Application**: Created by `bootstrap/app-of-apps/templates/gatewayapi-iam.yaml`

### 3. Demo Stack
**Chart**: `manifests/stacks/demo-helm/`
- **Templates**: `redis-claim.yaml` (Crossplane RedisInstance claim)
- **Values injected**: compartmentId, subnetId, nsgIds
- **Application**: Created by `bootstrap/app-of-apps/templates/demo-stack-helm.yaml`

## How Values Flow

1. **Root App** (`bootstrap/root-app.yaml`) syncs `bootstrap/app-of-apps` Helm chart
2. **Parent Helm Chart** renders all child `Application` CRs from `values.yaml`
3. **Child Applications** sync their respective Helm charts/manifests
4. **Child Helm Charts** receive templated values from parent app-of-apps
5. **Final Manifests** are rendered with all OCI IDs, regions, and settings

Example for Karpenter NodePools:
```
values.yaml (oci.karpenter.compartmentId)
  ↓
app-of-apps templates (karpenter-nodepools.yaml)
  ↓
Application CR (passes values to karpenter-nodepools-helm)
  ↓
karpenter-nodepools-helm/templates/nodeclass.yaml
  ↓
Final manifest with injected OCIDs
```

## Using This Setup

### Deploy
```bash
# Bootstrap the Helm app-of-apps
kubectl apply -f bootstrap/root-app.yaml
```

Argo CD will:
1. Render `bootstrap/app-of-apps` with values.yaml
2. Create all child `Application` CRs
3. Sync each application in order (by sync-wave)

### Customize for Your Environment
Edit `bootstrap/app-of-apps/values.yaml`:
```yaml
oci:
  region: your-region
  compartmentId: your-compartment
  karpenter:
    nodeCompartmentId: your-node-compartment
    # ... etc
```

All templated resources will automatically use these new values.

### Enable/Disable Components
```yaml
components:
  karpenter:
    enabled: true  # or false to disable
  crossplaneSystem:
    enabled: true
  # etc.
```

### Adjust Sync Waves
```yaml
components:
  karpenter:
    syncWave: "5"
  crossplaneSystem:
    syncWave: "10"
```

## Component Sync Order

| Wave | Component | Notes |
|------|-----------|-------|
| -10 | AppProject | Platform project definition |
| 5 | Karpenter | Base infrastructure |
| 7 | Karpenter NodePools | Nodepool resources (depends on Karpenter CRDs) |
| 10 | Crossplane System | Crossplane core (via Helm chart) |
| 11 | Gateway API IAM | Gateway IAM policies |
| 12 | Crossplane Providers | OCI + Kubernetes providers |
| 15 | Crossplane Compositions | XRD + Composition definitions |
| 15 | Envoy Gateway | Envoy Gateway controller (via Helm chart) |
| 25 | Gateway Objects | Gateway/GatewayClass/HTTPRoute resources |
| 100 | Stacks | Application workloads |

## Manifest Structure

```
manifests/
├── karpenter-nodepools-helm/         # Templated Karpenter NodePool chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       └── nodeclass.yaml
├── gatewayapi-iam/                   # Templated Gateway API IAM chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       └── policy.yaml
├── stacks/
│   └── demo-helm/                    # Templated demo stack chart
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           └── redis-claim.yaml
├── crossplane/
│   ├── providers/                    # Raw manifests (synced as-is)
│   └── compositions/
├── gatewayapi-objects/               # Raw manifests
└── karpenter-nodepools/              # Legacy (replaced by demo-helm version)
```

## Key Benefits

1. **Single source of truth**: All OCI IDs and configuration in one place
2. **Easy to customize**: Change values for different environments
3. **Component control**: Enable/disable components with flags
4. **DRY**: No repeated OCID values across manifests
5. **GitOps**: Everything is version controlled and declarative
6. **Flexible**: Mix templated Helm charts with raw manifests

## Adding New Components

To add a new templated component:

1. Create a new Helm chart in `manifests/` (e.g., `my-component/`)
2. Add values to `bootstrap/app-of-apps/values.yaml`
3. Create a template in `bootstrap/app-of-apps/templates/my-component.yaml` that creates an `Application` CR
4. Update the sync wave and enable flag in values.yaml

## Notes

- Secrets and sensitive credentials should still be managed separately (Vault, sealed secrets, etc.)
- The parent app-of-apps chart is simple and intentionally does minimal processing
- Child Helm charts handle the actual manifest templating
- Argo CD will auto-sync when values change

---

For detailed deployment instructions, see [README.md](../README.md) in the root.
