# OKE workshop demo

The `workshop` branch is an instructor-owned, disposable OKE demonstration
environment. Argo CD is used to deploy and reset labs; it is not a customer
requirement. Participants reproduce the equivalent Kubernetes steps in their
own environments.

Install Argo CD on the workshop cluster separately. Then apply only the
workshop entry point and manually sync it:

```bash
kubectl apply -f bootstrap/workshop-root-app.yaml
argocd app sync workshop-root
```

`bootstrap/root-app.yaml` targets the legacy `master` environment and must not
be used for this workshop. The lab Applications use manual sync. See the
[workshop runbook](docs/workshop-runbook.md) for prerequisites, observations,
and cleanup.
