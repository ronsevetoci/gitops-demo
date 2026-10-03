# Envoy Gateway lab prerequisites

These are instructor-cluster prerequisites, separate from the Kubernetes lab
manifests. Do not copy customer values into this repository.

## Controller and CRDs

Use the [Envoy Gateway v1.8 Helm installation guide](https://gateway.envoyproxy.io/v1.8/install/install-helm/)
for the pinned **v1.8.4** release. On a disposable instructor cluster where
the Gateway API CRDs are not already managed by another provider, install the
chart before syncing the lab:

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.8.4 -n envoy-gateway-system --create-namespace
```

The default chart installation includes Gateway API and Envoy Gateway CRDs.
If the cluster already has Gateway API CRDs, check their version and owner
against the versioned guide first. Do not install a second conflicting copy.
Keep installation and upgrades under the instructor's control; this lab does
not upgrade either CRDs or the controller.

## OCI networking and permissions

- Confirm the OKE cloud controller can provision a flexible OCI load balancer
  and that the selected subnet has capacity and a route to the demo client.
  This example disables load balancer NodePorts, so verify the cluster supports
  VCN-native pod networking and OCI load balancer pod backends.
- Decide whether the cluster-level default backend NSG should be used. If not,
  set `oci.oraclecloud.com/oci-backend-network-security-group` to
  `<BACKEND_NSG_OCID>` in an instructor-private overlay. Worker nodes or pods
  serving as LB backends must belong to that NSG, in the same VCN. If a
  dedicated LB subnet is needed, set
  `service.beta.kubernetes.io/oci-load-balancer-subnet1` to `<LB_SUBNET_OCID>`.
- Confirm frontend and backend traffic rules, OCI LB service limits, quota and
  the cluster's existing OCI IAM integration. Do not add broad IAM policies to
  the workshop baseline.

The [OKE load-balancer annotation guide](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingloadbalancer_topic-Summaryofannotations.htm)
lists the supported annotations. Never commit real OCIDs, tenancy details or
IAM credentials to this repository.
