# GitOps Demo: OCI Platform with Helm App-of-Apps

A single-environment GitOps setup using **Argo CD**, **Helm**, **Karpenter**, **Crossplane**, and **Gateway API** on Oracle Cloud Infrastructure.

## Quick Start

Deploy to your cluster with Argo CD:

```bash
kubectl apply -f bootstrap/root-app.yaml
```

This bootstraps the entire platform via a Helm-based app-of-apps pattern.

## Architecture

The repository is organized around a **Helm app-of-apps** architecture:

- **`bootstrap/root-app.yaml`** — Main Argo CD Application that syncs the app-of-apps chart
- **`bootstrap/app-of-apps/`** — Helm chart that templates all platform components
- **`bootstrap/app-of-apps/values.yaml`** — **Single source of truth** for all OCI configuration
- **`manifests/`** — Component manifests (both templated Helm charts and raw YAML)

## Single Source of Truth

All OCI infrastructure configuration is centralized in:

```
bootstrap/app-of-apps/values.yaml
```

This file contains:
- OCI region, compartment IDs
- Subnet and NSG OCIDs
- Cluster IDs and workspace identities
- Component enable/disable flags
- Sync-wave ordering

**Update this file to customize for your environment.**

## Templated Components

Infrastructure manifests are templated via Helm charts:

| Component | Chart | Templated Values |
|-----------|-------|------------------|
| Karpenter NodePools | `manifests/karpenter-nodepools-helm/` | Compartment, subnets, NSGs |
| Gateway API IAM | `manifests/gatewayapi-iam/` | Compartment ID |
| Demo Stack + Redis | `manifests/stacks/demo-helm/` | Compartment, subnets, NSGs |

All other components sync raw manifests or external Helm charts (Crossplane, Envoy Gateway).

## Repository Structure

```
bootstrap/
  ├── root-app.yaml                 # Argo CD bootstrap app
  └── app-of-apps/                  # Main Helm chart (generates all child apps)
      ├── Chart.yaml
      ├── values.yaml               # Centralized configuration
      └── templates/                # 13 Application templates

manifests/
  ├── karpenter-nodepools-helm/      # Templated Karpenter chart
  ├── gatewayapi-iam/                # Templated IAM policy chart
  ├── gatewayapi-objects/            # Raw Gateway API manifests
  ├── crossplane/
  │   ├── providers/                 # Raw Crossplane provider manifests
  │   └── compositions/              # Redis XRD + Composition
  └── stacks/
      ├── demo-helm/                 # Templated demo stack chart
      └── demo/app/                  # Demo application (Helm chart)

bootstrap/root-app.yaml
policy.yaml
values.yaml
notes.txt
```

## Configuration Example

Edit `bootstrap/app-of-apps/values.yaml` to customize:

```yaml
oci:
  region: eu-frankfurt-1
  compartmentId: ocid1.compartment.oc1..xxxxx
  
  karpenter:
    nodeCompartmentId: ocid1.compartment.oc1..xxxxx
    primarySubnetId: ocid1.subnet.oc1.xxxxx
    secondarySubnetId: ocid1.subnet.oc1.xxxxx
    primaryNsgId: ocid1.nsg.oc1.xxxxx
    secondaryNsgId: ocid1.nsg.oc1.xxxxx

  # ... more components

components:
  karpenter:
    enabled: true
  crossplaneSystem:
    enabled: true
  # Toggle components on/off here
```

All templated manifests will automatically use these values.

## Deployment Order

Components deploy in order via Argo CD sync-waves:

1. **Wave -10**: AppProject (permissions)
2. **Wave 5-7**: Karpenter (node orchestration)
3. **Wave 10-12**: Crossplane System + OCI Providers
4. **Wave 15**: Gateway API + Envoy Gateway
5. **Wave 25**: Gateway objects
6. **Wave 100**: Application stacks

See `HELM_ARCHITECTURE.md` for complete details.

## Documentation

- **`HELM_ARCHITECTURE.md`** — Complete architecture guide with all configuration options
- **`bootstrap/app-of-apps/values.yaml`** — Annotated centralized configuration

## Key Files

| File | Purpose |
|------|---------|
| `bootstrap/root-app.yaml` | Argo CD bootstrap; edit to point to your Git repo |
| `bootstrap/app-of-apps/values.yaml` | **Customize your environment here** |
| `bootstrap/app-of-apps/Chart.yaml` | App-of-apps chart metadata |
| `bootstrap/app-of-apps/templates/` | Argo CD Application templates for each component |

## Next Steps

1. **Fork this repository** to your GitHub account
2. **Update `bootstrap/root-app.yaml`** to point to your repo
3. **Edit `bootstrap/app-of-apps/values.yaml`** with your OCI infrastructure details
4. **Deploy**: `kubectl apply -f bootstrap/root-app.yaml`
5. **Monitor**: Watch Argo CD as the platform bootstraps

## Notes

- This is a single-demo environment setup; extend `values.yaml` for multiple environments
- Secrets (OCI credentials, etc.) should be managed separately via Vault or sealed secrets
- Sync policies use `prune: true` and `selfHeal: true` for automatic GitOps reconciliation

