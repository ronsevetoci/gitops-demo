# Crossplane Setup Guide

## Overview

This guide explains the optimized **lean, production-ready Crossplane** integration for provisioning OCI infrastructure through Kubernetes claims.

**Current Use Case:** Redis provisioning via Crossplane compositions + claims.

## Architecture

```
┌─────────────────────────────────────────────────┐
│ Developer: Creates RedisInstance Claim          │
│ (manifests/stacks/demo-helm/templates/...)      │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│ XRD: XRedisInstance                             │
│ (manifests/crossplane/compositions/redis-xrd)   │
│ • Defines parameters (compartmentId, subnetId)  │
│ • Defines status (primaryFqdn, endpoint)        │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│ Composition: oci-redisinstance                  │
│ (manifests/crossplane/compositions/...)         │
│ • Pipeline mode with patch-and-transform        │
│ • Creates RedisCluster resource (OCI)           │
│ • Creates ConfigMap (connection details)        │
└────────────────┬────────────────────────────────┘
                 │
        ┌────────┴────────┐
        ▼                 ▼
  ┌──────────────┐  ┌──────────────┐
  │ OCI Redis    │  │ Kubernetes   │
  │ Cluster      │  │ ConfigMap    │
  │ (via OCI API)│  │ (connection) │
  └──────────────┘  └──────────────┘
```

## Components

### 1. Providers (`manifests/crossplane/providers/providers.yaml`)

**Minimal Provider Set:**

| Provider | Purpose | Package |
|----------|---------|---------|
| **family-oci** | All OCI resource types (Redis, compute, network, etc) | ghcr.io/oracle/provider-family-oci:v1.1.0 |
| **kubernetes** | In-cluster resources (ConfigMaps, Secrets) | ghcr.io/crossplane-contrib/provider-kubernetes:v0.14.0 |

**Removed (Redundant):**
- ~~provider-oci-redis~~ (covered by family-oci)
- ~~provider-oci-core~~ (covered by family-oci)
- ~~provider-oci-identity~~ (covered by family-oci)
- ~~provider-oci-objectstorage~~ (covered by family-oci)

### 2. Composition Function

**Enables Pipeline Mode Compositions:**
```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: ghcr.io/crossplane/function-patch-and-transform:v0.2.1
```

Required for:
- Patch-based field transformations
- Dynamic value composition
- Advanced schema validation

### 3. RBAC

```yaml
ServiceAccount: crossplane-provider-oci
├── ClusterRole: permissions to read secrets, write events
└── ClusterRoleBinding: binds SA to ClusterRole
```

**Enables:** Provider to authenticate with OCI via workload identity.

### 4. DeploymentRuntimeConfig

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: oci-workload-identity-runtime
spec:
  deploymentTemplate:
    spec:
      serviceAccountName: crossplane-provider-oci
      nodeSelector: oke.oraclecloud.com/pool.name: karpenter
      containers:
        - env:
            - OCI_RESOURCE_PRINCIPAL_VERSION: "2.2"
            - OCI_RESOURCE_PRINCIPAL_WORKLOAD_IDENTITY: "1"
```

**Benefits:**
- OKE workload identity (no static credentials)
- Scheduled on Karpenter nodes only
- Automatic tolerance for critical addons

### 5. ProviderConfigs

#### OCI ProviderConfig

```yaml
apiVersion: oci.m.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      name: oci-creds
      namespace: crossplane-system
      key: credentials
```

**Features:**
- Uses workload identity (minimal secret required)
- Referenced by all OCI resources in compositions
- Region handled by OKE workload identity provider

#### In-Cluster ProviderConfig

```yaml
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: in-cluster
spec:
  credentials:
    source: InjectedIdentity
```

**Features:**
- Kubernetes provider accesses the same cluster
- Uses injected service account credentials
- No external configuration needed

### 6. XRD (Composite Resource Definition)

**File:** `manifests/crossplane/compositions/redis-xrd.yaml`

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xredisinstances.platform.example.org
spec:
  group: platform.example.org
  names:
    kind: XRedisInstance
    plural: xredisinstances
  claimNames:
    kind: RedisInstance
    plural: redisinstances
  connectionSecretKeys:
    - REDIS_HOST
    - REDIS_PORT
  versions:
    - name: v1alpha1
      schema:
        openAPIV3Schema:
          properties:
            spec:
              properties:
                parameters:
                  properties:
                    compartmentId: string (required)
                    subnetId: string (required)
                    displayName: string
                    nodeCount: integer (default: 1)
                    nodeMemoryInGbs: integer (default: 2)
                    softwareVersion: string (default: REDIS_7_0)
                    nsgIds: array (optional NSG IDs)
                    connectionConfigMapName: string (default: demo-redis-connection)
                    connectionConfigMapNamespace: string (default: demo)
```

**Key Features:**
- `parameters`: inputs from claims (compartment, subnet, sizing)
- `status`: outputs from composition (primaryFqdn, endpoint)
- `connectionSecretKeys`: which data to expose in connection secret

### 7. Composition

**File:** `manifests/crossplane/compositions/redis-composition.yaml`

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: oci-redisinstance
spec:
  compositeTypeRef:
    apiVersion: platform.example.org/v1alpha1
    kind: XRedisInstance
  mode: Pipeline
  pipeline:
    - step: patch-and-transform
      functionRef:
        name: function-patch-and-transform
      input:
        resources:
          # Managed Resource 1: OCI Redis Cluster
          - name: redis-cluster
            base:
              apiVersion: redis.oci.m.upbound.io/v1alpha1
              kind: RedisCluster
              spec:
                managementPolicies:
                  - "*"  # All lifecycle operations allowed
                providerConfigRef:
                  name: default
            patches:
              - FromCompositeFieldPath: compartmentId → spec.forProvider.compartmentId
              - FromCompositeFieldPath: subnetId → spec.forProvider.subnetId
              - ToCompositeFieldPath: status.atProvider.primaryFqdn → status.primaryFqdn

          # Managed Resource 2: Kubernetes ConfigMap
          - name: redis-connection-configmap
            base:
              apiVersion: kubernetes.crossplane.io/v1alpha1
              kind: Object
              spec:
                managementPolicies:
                  - "*"
                providerConfigRef:
                  name: in-cluster
                forProvider:
                  manifest:
                    kind: ConfigMap
                    data:
                      REDIS_HOST: ""
                      REDIS_PORT: "6379"
                      REDIS_URL: ""
            patches:
              - FromCompositeFieldPath: connectionConfigMapName → manifest.metadata.name
              - FromCompositeFieldPath: primaryFqdn → manifest.data.REDIS_HOST
              - Transform: primaryFqdn → redis://primaryFqdn:6379 → manifest.data.REDIS_URL
```

**Key Features:**
- **managementPolicies:** controls which operations are allowed (create/update/delete/observe)
- **Patches:** link claim inputs to managed resource specs, and resource outputs back to claim status
- **Transforms:** format values (e.g., FQDN → Redis URL)

### 8. Claims

**File:** `manifests/stacks/demo-helm/templates/redis-claim.yaml`

```yaml
apiVersion: platform.example.org/v1alpha1
kind: RedisInstance
metadata:
  name: demo-redis
  namespace: demo
spec:
  compositionRef:
    name: oci-redisinstance
  parameters:
    compartmentId: "ocid1.compartment.oc1..."
    subnetId: "ocid1.subnet.oc1..."
    displayName: "demo-redis"
    nodeCount: 1
    nodeMemoryInGbs: 2
    softwareVersion: REDIS_7_0
    nsgIds:
      - "ocid1.nsg.oc1..."
    connectionConfigMapName: demo-redis-connection
    connectionConfigMapNamespace: demo
```

**Workflow:**
1. Claim specifies parameters (compartment, subnet, sizing)
2. Composition creates RedisCluster resource in OCI
3. Composition creates ConfigMap with connection details
4. Application reads ConfigMap to connect to Redis

## Deployment Order (Sync Waves)

```
Wave 0:  ServiceAccount, RBAC, RuntimeConfig, Secret
    ↓
Wave 1:  (initial sync-wave annotations in secret/runtime)
    ↓
Wave 5:  Providers (family-oci, kubernetes), Functions (patch-and-transform)
    ↓
Wave 10: ProviderConfigs (default, in-cluster)
    ↓
Wave 15: XRD + Composition
    ↓
Wave 20+: Composition is ready
```

## Configuration

### values.yaml (Helm App-of-Apps)

```yaml
components:
  crossplaneSystem:
    enabled: true
    syncWave: "10"
  crossplaneProviders:
    enabled: true
    syncWave: "12"
  crossplaneCompositions:
    enabled: true
    syncWave: "15"

charts:
  crossplane:
    repoURL: https://charts.crossplane.io/stable
    chart: crossplane
    version: ">=1.0.0"

providers:
  familyOci:
    package: ghcr.io/oracle/provider-family-oci:v1.1.0
  kubernetes:
    package: ghcr.io/crossplane-contrib/provider-kubernetes:v0.14.0
  functions:
    patchAndTransform: ghcr.io/crossplane/function-patch-and-transform:v0.2.1
```

### Updating Provider Versions

To update provider versions, edit:

```yaml
# bootstrap/app-of-apps/values.yaml
providers:
  familyOci:
    package: ghcr.io/oracle/provider-family-oci:v1.2.0  # Update version here
  kubernetes:
    package: ghcr.io/crossplane-contrib/provider-kubernetes:v0.15.0
  functions:
    patchAndTransform: ghcr.io/crossplane/function-patch-and-transform:v0.3.0
```

Argo CD will automatically update the providers.yaml during sync.

## Deletion Policies

### Automatic Cleanup (Current)

When a RedisInstance claim is deleted:

1. Composition's `managementPolicies: ["*"]` allows deletion
2. RedisCluster in OCI is deleted (via Crossplane reconciliation)
3. ConfigMap is deleted (Kubernetes Object lifecycle)
4. XRD status cleared

**Time to cleanup:** ~2-5 minutes for OCI resources

### Optional: Retain on Claim Deletion

To keep Redis cluster even if claim is deleted:

```yaml
# In redis-composition.yaml, redis-cluster resource:
managementPolicies:
  - create
  - update
  - observe
  # Removed 'delete' → cluster retained
```

## Troubleshooting

### Check Provider Status

```bash
kubectl get providers -n crossplane-system
kubectl describe provider provider-family-oci -n crossplane-system
```

### Check Composition and XRD

```bash
kubectl get xrd
kubectl get composition
```

### Debug a Failing Claim

```bash
kubectl describe redisinstance demo-redis -n demo
kubectl logs -n crossplane-system deployment/crossplane-provider-family-oci -f
```

### Check ProviderConfigs

```bash
kubectl get providerconfig -A
kubectl describe providerconfig default -n crossplane-system
```

## Security Considerations

### Credentials

- **Current:** OKE Workload Identity (no static credentials in Secret)
- **Secret Content:** Only `{"auth": "OKEWorkloadIdentity"}` placeholder
- **Real auth:** Handled by OKE workload identity provider and IRSA

### RBAC

- ServiceAccount limited to: reading secrets, writing events
- Only crossplane-provider-oci SA has cluster-level access
- Applications in demo namespace read ConfigMap only (no OCI access)

### Future Improvements

- Use [External Secrets Operator](https://external-secrets.io/) for credential rotation
- Implement [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) to isolate crossplane-system
- Add [Pod Security Policies](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) for provider pods

## Next Steps

1. **Monitor Crossplane:**
   ```bash
   kubectl logs -n crossplane-system deployment/crossplane -f
   ```

2. **Create a Claim:**
   ```bash
   kubectl apply -f manifests/stacks/demo-helm/templates/redis-claim.yaml
   ```

3. **Verify Composition:**
   ```bash
   kubectl get redisinstance -n demo -o yaml
   kubectl get rediscluster -A
   ```

4. **Check ConfigMap:**
   ```bash
   kubectl get configmap -n demo
   kubectl describe configmap demo-redis-connection -n demo
   ```

5. **Extend Compositions:**
   - Add new XRDs for other OCI resources (storage, networking, compute)
   - Reuse family-oci provider for all new compositions
   - Follow same patch-and-transform pattern

## References

- [Crossplane Documentation](https://docs.crossplane.io/)
- [Oracle Provider Family OCI](https://github.com/oracle/provider-family-oci)
- [Crossplane Composition Functions](https://docs.crossplane.io/latest/concepts/compositions/#composition-functions)
- [OKE Workload Identity](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengusingworkloadidentity.htm)
