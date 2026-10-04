# OKE Workshop Demo Repository Instructions

## Purpose

This repository is being prepared as an instructor-owned OKE demonstration
environment for a live workshop. Argo CD is used to deploy, reset and observe
the labs. Argo CD itself is not a customer requirement.

The customer will execute equivalent steps in their own environment. Do not
assume access to, or make changes in, the customer environment.

## Branch and change boundaries

- Work only on the `workshop` branch unless the user explicitly requests a
  different branch.
- Do not modify, reset, rebase or push `master` without explicit approval.
- Do not push changes to GitHub unless explicitly requested.
- Prefer small, additive changes. Preserve the existing platform boilerplate
  while the workshop layer is being developed.

## Workshop scope

The workshop topics are:

1. `תשתית ליבה`, `ניהול ותחזוקת`, `עדכונים ושדרוגים` and
   `ארכיטקטורת High Availability`
2. `מדיניות אבטחת Pods` and
   `ניהול והגבלת משאבים (Quotas, Limits & LimitRanges)`
3. `Workload Autoscaling` and `Cluster Autoscaling`
4. `ניהול חשיפת שירותים ורשת (LB & Ingress)` using Envoy Gateway
5. `אופטימיזציית משאבים ועלויות`, only where it directly relates to KPO

Do not add unrelated platform products or generic Kubernetes subjects unless
they are required by one of these labs.

## Repository layout

- `bootstrap/workshop-root-app.yaml`: manual entry point for the workshop
  Argo CD application.
- `apps/workshop/`: Argo CD `AppProject` and one Application per lab topic.
- `manifests/workshop/`: desired state for topic applications.
- `labs/`: deliberate manual scenarios, including negative tests that must not
  be continuously reconciled by Argo CD.
- `docs/workshop-runbook.md`: instructor and participant instructions.
- Existing `apps/core/`, `apps/infra/` and Crossplane content are legacy
  boilerplate. Do not pull them into the core workshop path without a specific
  reason.

## Argo CD rules

- Workshop Applications must use manual sync by default.
- Do not enable automated prune or self-heal for interactive lab resources.
- Use one Application per topic so the instructor can demonstrate topics
  independently and reset them independently.
- Use sync waves when CRDs/controllers must exist before custom resources.
- Do not manage generated resources such as `NodeClaim` or HPA status.
- Do not let Argo CD continuously enforce a fixed Deployment replica count when
  HPA is controlling replicas. Use `ignoreDifferences` where appropriate.
- Do not use Argo CD to perform OKE cluster or node-pool upgrades.

## OCI and security rules

- Never commit kubeconfigs, tokens, private keys, secrets or customer data.
- Never commit customer-specific OCIDs, tenancy details, subnet IDs, NSG IDs or
  region-specific values to reusable workshop manifests.
- Use clearly named placeholders, example values files or documented variables.
- Do not add broad IAM statements such as `Allow any-user` to the reusable
  workshop baseline.
- IAM, networking and KPO prerequisites must be documented separately from the
  Kubernetes lab manifests.
- Treat all topic labs as non-production and safe to delete.

## Topic-specific guidance

### Governance

Use native Kubernetes objects first: Pod Security Admission, `securityContext`,
`ResourceQuota` and `LimitRange`. Add Kyverno only for a concrete customer
requirement that native objects do not address. Do not install or teach Kyverno
as a generic product demonstration.

### HPA and KEDA

Provide one complete HPA example using resource metrics. Mention KEDA briefly
as a customer-operated option for event-driven or external metrics. Do not make
KEDA a full lab unless the workshop scope explicitly changes.

### KPO

KPO installation and IAM are environment prerequisites. The lab should focus on
`OCINodeClass`, `NodePool`, pending workload behavior, scheduling constraints,
capacity, quotas and consolidation. Pin the KPO version and validate all OCI
values before adding the Application to the workshop root.

### Envoy Gateway

Use the approved workshop repository/manifests and a pinned version. Keep the
lab focused on Gateway API, `Gateway`, `HTTPRoute` and OCI load-balancer
integration. Do not introduce an unrelated ingress controller.

### Maintenance and upgrades

Use a read-only inspection and controlled walkthrough. Do not perform a real
cluster upgrade during a shared demonstration unless the user explicitly
authorizes it for a disposable cluster.

## Required validation before handing off a change

Run as many of these as the local environment supports:

```bash
git diff --check
git status --short --branch
```

Validate YAML syntax and confirm that every Argo CD Application source path
exists. If `kustomize`, `kubectl`, or Argo CD CLI are available, also run the
appropriate offline build or dry-run. Do not connect to a customer cluster.

Every new lab must document:

1. prerequisites
2. sync/apply step
3. observation or expected result
4. cleanup/reset step
5. known ownership and support limitations

When reporting work, list changed files, validations performed, assumptions and
anything that still requires testing against the instructor's OKE cluster.
