# EKS Observability Platform

A production-grade, hub-and-spoke observability platform built on AWS EKS.
A central EKS cluster monitors itself **and** any number of remote
workload clusters using Grafana Alloy + the LGTM stack (Loki, Grafana,
Tempo, Mimir/Prometheus), all delivered via GitOps with ArgoCD.

This is split across **three independent GitHub repositories** so each
layer can evolve at its own pace and you learn clean separation of
concerns.

---

## The big picture

```
                                Internet
                                   │
                                   ▼
                  ┌────────────────────────────────┐
                  │ AWS NLB (TLS terminated, ACM)  │
                  │ *.monitoring.<your-domain>     │
                  └────────────────┬───────────────┘
                                   │
                                   ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │              Hub:  observability-cluster (EKS)                  │
   │                                                                 │
   │   Envoy Gateway ─► HTTPRoutes ─► Grafana / Prom / Loki / Tempo  │
   │                                  Thanos / Alertmanager / ArgoCD │
   │                                                                 │
   │   Prometheus + Thanos sidecar (S3)     Grafana (EFS)            │
   │   Loki (S3)                            Grafana Operator         │
   │   Tempo (S3, OTLP)                     External Secrets         │
   │   Grafana Alloy DaemonSet (self-monitor)                        │
   │                                                                 │
   │   Karpenter, fck-nat, single-AZ workloads (cost-optimized)      │
   └─────────────────────────────────────────────────────────────────┘
                                   ▲
                Metrics, logs, traces (basic auth)
                                   │
                  ┌────────────────────────────────┐
                  │ Spoke EKS cluster(s)           │
                  │ Grafana Alloy DaemonSet only   │
                  └────────────────────────────────┘
```

---

## The three repositories

| Repo | What it owns | Tools |
|---|---|---|
| **[`eks-observability-iac`](./eks-observability-iac/)** | AWS infrastructure: VPC, EKS, IAM/IRSA, S3, EFS, ACM, Secrets Manager. Also: demo spoke cluster, operator scripts, Alloy values for spokes. | Terraform, AWS CLI, kubectl |
| **[`eks-observability-helm-charts`](./eks-observability-helm-charts/)** | Versioned Helm chart definitions. One umbrella chart per component (kube-prometheus-stack, loki, tempo, thanos, grafana, alloy, ...). Published as a Helm repo on GitHub Pages. | Helm, chart-releaser |
| **[`eks-observability-gitops`](./eks-observability-gitops/)** | What runs in the cluster: ArgoCD `Application` and `AppProject` CRs, plus native Kubernetes resources (Gateway API, PrometheusRule, GrafanaDashboard, ExternalSecret, ...). | ArgoCD, kustomize |

Each repo has its own `README.md` and `ARCHITECTURE.md`. **Read those for
details.** This file is just the map.

---

## Recommended reading order

If you're new to the platform, read in this order:

1. **This file** — understand the three-repo split (you are here).
2. **[`eks-observability-iac/README.md`](./eks-observability-iac/README.md)** — what's deployed on AWS.
3. **[`eks-observability-iac/ARCHITECTURE.md`](./eks-observability-iac/ARCHITECTURE.md)** — *why* those choices.
4. **[`eks-observability-helm-charts/README.md`](./eks-observability-helm-charts/README.md)** — how charts are versioned and published.
5. **[`eks-observability-helm-charts/ARCHITECTURE.md`](./eks-observability-helm-charts/ARCHITECTURE.md)** — umbrella chart pattern.
6. **[`eks-observability-gitops/README.md`](./eks-observability-gitops/README.md)** — how ArgoCD ties chart + cluster together.
7. **[`eks-observability-gitops/ARCHITECTURE.md`](./eks-observability-gitops/ARCHITECTURE.md)** — sync waves, secrets, DNS.

---

## End-to-end deployment

```
┌───────────────────────────────────────────────────────────────────────┐
│  ONE-TIME SETUP                                                       │
└───────────────────────────────────────────────────────────────────────┘

1. eks-observability-iac/bootstrap/tf-state/
       terraform init && terraform apply              ← creates state bucket

2. eks-observability-iac/environments/production/
       terraform init -backend-config=backend.tfbackend
       terraform plan && terraform apply              ← creates EKS + AWS infra

3. eks-observability-helm-charts/ (this happens via CI once the repo is on GitHub)
       git tag kube-prometheus-stack-1.0.0 && git push --tags
                                                      ← release workflow
                                                        publishes to gh-pages

4. eks-observability-gitops/bootstrap/
       kubectl apply -f argocd-install-values.yaml    ← install ArgoCD
       kubectl apply -f ../apps/root.yaml             ← hand keys to ArgoCD

5. Wait ~15 min for sync waves 0→8 to complete.

6. eks-observability-iac/scripts/update-dns.sh        ← point *.monitoring.<domain>
                                                        at the NLB

7. Log in to https://grafana.<domain> with the admin password from
   AWS Secrets Manager (observability/grafana).

┌───────────────────────────────────────────────────────────────────────┐
│  ADD A SPOKE                                                          │
└───────────────────────────────────────────────────────────────────────┘

8. eks-observability-iac/spokes/demo-spoke/
       terraform init && terraform apply              ← provision spoke EKS

9. eks-observability-iac/
       ./scripts/install-spoke-alloy.sh demo-spoke    ← install Alloy on spoke
                                                        (pulls password from
                                                         AWS Secrets Manager)

10. In Grafana, query:  up{cluster="demo-spoke"}      ← verify spoke is reporting
```

---

## Prerequisites

| Tool | Version | Why |
|---|---|---|
| Terraform | >= 1.10.0 | Native S3 state locking (`use_lockfile = true`) |
| AWS CLI | >= 2.15 | Pretty much everything |
| kubectl | >= 1.30 | Talk to EKS |
| helm | >= 3.14 | Local chart linting + ArgoCD install + spoke install |
| jq | any | Scripts |
| git | any | Obviously |

Plus an AWS account where you can create VPCs, EKS clusters, IAM roles,
S3 buckets, and use a Route53 hosted zone you already own.

---

## Cost (when running 24/7)

| Component | Approx. cost (eu-central-1) |
|---|---|
| EKS control plane (hub) | 73 USD/mo |
| Hub managed node group (1× t3.medium ON_DEMAND) | 30 USD/mo |
| Hub observability node group (2× t3.large SPOT) | 40 USD/mo |
| NLB (Envoy Gateway exposure) | 20 USD/mo |
| fck-nat hub | 3 USD/mo |
| S3 + EFS storage | 10 USD/mo |
| **Hub subtotal** | **~180 USD/mo** |
| Demo spoke (EKS + 1× t3.medium SPOT + fck-nat) | ~110 USD/mo |
| **Grand total** | **~290 USD/mo** |

For learning: run `eks-observability-iac/scripts/cluster-shutdown.sh`
when not in use. Brings it down to ~80 USD/mo (control plane + EBS only).

---

## CI/CD

Every repo has GitHub Actions workflows. Together they teach the patterns
a junior DevOps engineer will see at most companies:

| Repo | Workflows |
|---|---|
| iac | `pr-validate` (fmt + validate + tflint + tfsec + plan comment), `apply` (protected environment), `drift-detect` (nightly issue) |
| helm-charts | `pr-validate` (matrix helm-lint + helm-template + kubeconform), `release` (chart-releaser → gh-pages) |
| gitops | `pr-validate` (yamllint + kustomize + kubeconform), `argocd-diff` (PR comment showing what'll change) |

OIDC is used everywhere — no long-lived AWS keys in GitHub secrets.

---

## What's intentionally NOT included

- **Application workloads.** This is platform infrastructure. Apps that
  want to be observed install Alloy or use the OTLP endpoint.
- **OIDC / Cognito SSO.** Grafana and ArgoCD use local admin only by
  default. SSO is one of the documented "next steps."
- **Cross-cluster alerting.** Each Prometheus alerts locally. Thanos Ruler
  is disabled by default; enable in the helm-charts repo's `thanos/values.yaml`
  if you need global alerts.
- **external-dns.** DNS is updated by a one-shot script. external-dns is
  the right long-term answer.
- **Traefik on EC2.** The NLB is fine for ~250 USD/mo. The Traefik-on-EC2
  trick (saves ~800 USD/mo at scale) is documented as a future exercise
  in `eks-observability-iac/ansible/README.md`.

---

## Credits & references

- LGTM stack: [Grafana Labs](https://grafana.com/oss/)
- Prometheus stack: [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator) + [community Helm chart](https://github.com/prometheus-community/helm-charts)
- GitOps: [ArgoCD](https://argo-cd.readthedocs.io/)
- Gateway API: [envoyproxy/gateway](https://gateway.envoyproxy.io/)
- Karpenter: [karpenter.sh](https://karpenter.sh/)
- fck-nat: [RaJiska/terraform-aws-fck-nat](https://github.com/RaJiska/terraform-aws-fck-nat)
- External Secrets: [external-secrets.io](https://external-secrets.io/)
- Chart releaser: [helm/chart-releaser-action](https://github.com/helm/chart-releaser-action)
