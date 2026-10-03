# OKE Workshop Demo Runbook

This branch is an instructor-owned OKE demonstration environment. Argo CD is
used to deploy and reset topic labs; it is not itself a customer requirement.

## Bootstrap

Install Argo CD separately, then apply the workshop root application:

```bash
kubectl apply -f bootstrap/workshop-root-app.yaml
argocd app list
```

The workshop applications use manual sync intentionally. Sync only the topic
being demonstrated:

```bash
argocd app sync workshop-governance
argocd app sync workshop-hpa
```

## Governance lab

```bash
kubectl get resourcequota,limitrange -n workshop-governance
kubectl get pods -n workshop-governance
kubectl apply -f labs/governance/noncompliant-pod.yaml
```

Expected result: the invalid Pod is rejected by Pod Security Admission. Kyverno
is an optional follow-on and must be based on a specific customer requirement.

## HPA lab

Confirm that the metrics API is available:

```bash
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl top pods -n workshop-hpa
```

Generate load from a temporary Pod:

```bash
kubectl run load-generator -n workshop-hpa --rm -it \
  --image=busybox:1.36.1 --restart=Never -- \
  sh -c 'while true; do wget -q -O- http://hpa-demo; done'
```

Observe scaling in another terminal:

```bash
kubectl get hpa,pods -n workshop-hpa -w
```

## Future topic applications

Add KPO and Envoy Gateway as separate Applications only after their versions,
IAM requirements, OCI network values and manifests have been validated for the
demo cluster. Do not place environment-specific OCIDs or broad IAM policies in
the reusable workshop baseline.
