# OKE workshop runbook

This repository is the instructor's disposable demonstration environment. Argo CD
deploys and resets lab resources; participants perform equivalent steps in their
own environments.

## Bootstrap

Install Argo CD separately. From the `workshop` branch, apply and manually sync
the root application:

```bash
kubectl apply -f bootstrap/workshop-root-app.yaml
argocd app sync workshop-root
```

The root creates the `workshop` AppProject and one manual-sync Application for
each topic. Sync only the topic being demonstrated.

## Envoy Gateway: OCI load balancer and HTTPRoute

### Prerequisites

Review [Envoy Gateway prerequisites](envoy-gateway-prerequisites.md) before
syncing. The instructor's OKE cluster must have Envoy Gateway **v1.8.4** and
compatible Gateway API and Envoy Gateway CRDs installed. The controller is an
environment prerequisite, not owned by this lab Application. Check that the
controller is available:

```bash
kubectl -n envoy-gateway-system rollout status deployment/envoy-gateway
kubectl get crd envoyproxies.gateway.envoyproxy.io gateways.gateway.networking.k8s.io httproutes.gateway.networking.k8s.io
```

Confirm the cluster's LB subnet, NSG setup, OCI permissions and service LB
capacity. If there is no cluster-level default backend NSG, configure the
`<BACKEND_NSG_OCID>` annotation in an instructor-private overlay before syncing.
The reusable manifest intentionally contains no real OCID. The example uses a
10 Mbps flexible OCI load balancer, which can incur charges.

### Sync

```bash
argocd app sync workshop-envoy-gateway
```

The sync creates an `EnvoyProxy`, `GatewayClass`, `Gateway`, `HTTPRoute` and a
small echo backend. Sync waves create the namespace first, the proxy settings
and backend next, then the Gateway and route. Sync is manual, with no automated
pruning or self-healing.

### Observe

```bash
kubectl get gatewayclass workshop-envoy-oci
kubectl get gateway,httproute -n workshop-envoy
kubectl get service -n envoy-gateway-system -l gateway.envoyproxy.io/owning-gateway-name=workshop-gateway
```

Expect `Accepted=True` on the GatewayClass, `Programmed=True` on the Gateway,
and `Accepted=True` and `ResolvedRefs=True` on the HTTPRoute. Wait for an
external address on the Envoy Service, then send an HTTP request to port 80 and
expect `OKE Envoy Gateway workshop` in the response. OCI LB provisioning can
take several minutes.

### Reset and cleanup

To rerun a routing change, edit the instructor workshop branch and manually
sync `workshop-envoy-gateway` again. To end the lab and delete its resources:

```bash
kubectl delete -k manifests/workshop/envoy-gateway
```

Wait until the Envoy Service and its OCI load balancer have been deleted. The
Application will show `OutOfSync` until it is synced again. Keep the controller
and shared CRDs in place if other Gateways use them.

### Ownership and support limits

The instructor owns this non-production namespace, GatewayClass, route, demo
backend and the OCI load balancer created for the Gateway. OCI networking, IAM,
cluster configuration, shared CRDs and the Envoy Gateway controller are managed
outside this Application. The legacy `apps/infra/gatewayapi` Applications are
not part of this lab; do not sync them into the same cluster without checking
for controller and CRD ownership conflicts.
