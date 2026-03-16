# k8s — Local Kubernetes Learning

Hands-on Kubernetes reps on M4 Mac, mapped to AWS/ECS concepts I already own. Aligned to the KCNA certification domains.

## Stack

- **kind** (Kubernetes in Docker)
- **Colima** + Docker
- MacBook M4 Pro

## Learning Phases

| Phase | Topic | Status |
|---|---|---|
| 1 | Local cluster setup | ✓ |
| 2 | Core objects: Pods, Deployments, Services | ✓ |
| 3 | Config and Secrets (ConfigMap, Secret) | ✓ |
| 4 | Ingress | In progress |
| 5 | Namespaces and RBAC | — |
| 6 | Real app end-to-end | — |
| 7 | Cloud Native App Delivery (GitOps, Helm, Argo CD) | — |
| 8 | Cloud Native Architecture (CNCF, observability) | — |

Full plan and progress tracked in `plan.md`.

## Repo Structure

```
configs/       ConfigMaps and Secrets
deployments/   Deployment manifests
pods/          Standalone Pod manifests
services/      Service manifests
docs/          Cheatsheet and manifest templates
```

## AWS Translation Layer

| AWS / ECS | Kubernetes |
|---|---|
| ECS Task Definition | Pod spec |
| ECS Task | Pod |
| ECS Service | Deployment + Service |
| ALB | Ingress |
| Parameter Store | ConfigMap |
| Secrets Manager | Secret |
| IAM Role (task) | ServiceAccount + RBAC |

## Prerequisites

```bash
# Start Docker (Colima)
colima start

# Verify cluster
kubectl get nodes
```
