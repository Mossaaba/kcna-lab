# Kubernetes Local Learning Lab

A hands-on Kubernetes learning repo built for local development on an M-series Mac. Works through all four KCNA certification domains by building real things — not just reading docs.

Every phase produces something that runs. Concepts are mapped to AWS/ECS equivalents throughout, which makes the mental model click faster if you have a cloud background.

---

## What You'll Build

| Phase | Topic | What you end up with |
|---|---|---|
| 1 | Local cluster | A running kind cluster, `kubectl` connected |
| 2 | Core objects | Pods, Deployments, Services deployed and verified |
| 3 | Config & Secrets | ConfigMaps and Secrets injected as env vars and volume mounts |
| 4 | Ingress | External traffic routed into the cluster via nginx-ingress |
| 5 | Namespaces & RBAC | Isolated namespaces, ServiceAccounts, Roles verified with `kubectl auth can-i` |
| 6 | Real app end-to-end | Full app: Deployment, Service, ConfigMap, Secret, Ingress, RBAC, resource limits, health probes |
| 7 | GitOps (Helm + Argo CD) | App packaged as a Helm chart, deployed and auto-synced by Argo CD from GitHub |
| 8 | Observability | Prometheus + Grafana running in cluster, scraping real metrics |

---

## KCNA Coverage

| Domain | Weight | Phases |
|---|---|---|
| Kubernetes Fundamentals | 44% | 2, 3, 4, 5, 6 |
| Container Orchestration | 28% | 2, 4, 5, 6 |
| Cloud Native Application Delivery | 16% | 7 |
| Cloud Native Architecture | 12% | 8 |

---

## Prerequisites

### Install these tools

```bash
# Homebrew (if not already installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Colima — lightweight Docker runtime (no Docker Desktop needed)
brew install colima

# Docker CLI
brew install docker

# kind — Kubernetes in Docker
brew install kind

# kubectl — Kubernetes CLI
brew install kubectl

# Helm — Kubernetes package manager (needed for Phase 7+)
brew install helm
```

### Start the runtime

```bash
# Start Colima with enough resources for the full stack (Phases 7-8 are heavy)
colima start --cpu 4 --memory 6
```

---

## Cluster Setup

This repo includes a `kind-config.yaml` that configures the cluster with ingress support and port mapping pre-wired.

```bash
# Create the cluster
kind create cluster --config kind-config.yaml

# Verify it's up
kubectl get nodes
# Expected: k8s-learning-control-plane   Ready   control-plane
```

**kind-config.yaml:**
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: k8s-learning
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 8080
        protocol: TCP
```

> Note: `ingress-ready=true` is required by the nginx-ingress manifest for kind. Without it the controller pod stays `Pending`.

---

## Repo Structure

```
k8s/
├── pods/               Phase 2 — standalone Pod manifests
├── deployments/        Phases 2, 6 — Deployment manifests
├── services/           Phases 2, 6 — Service manifests
├── configs/            Phase 3 — ConfigMaps and Secrets
├── ingress/            Phase 4 — Ingress rules
├── namespaces/         Phase 5 — Namespaces, ServiceAccounts, Roles, RoleBindings
├── charts/
│   └── app/            Phase 7 — Helm chart for the Phase 6 app
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
├── argocd/
│   └── application.yaml  Phase 7 — Argo CD Application manifest
└── docs/
    └── k8s-cheatsheet.md  Full reference: manifests, commands, AWS mappings, troubleshooting
```

---

## Phase Walkthroughs

### Phase 1 — Local Cluster

Goal: get a cluster running and `kubectl` connected.

```bash
colima start --cpu 4 --memory 6
kind create cluster --config kind-config.yaml
kubectl get nodes
```

**AWS equivalent:** Creating a VPC and an ECS cluster — you're setting up the environment before deploying anything.

---

### Phase 2 — Core Objects

Deploy your first Pod, Deployment, and Service. Understand the label wiring that connects them.

```bash
kubectl apply -f pods/pod-hello.yaml
kubectl apply -f deployments/deployment-hello.yaml
kubectl apply -f services/service-hello.yaml

kubectl get pods
kubectl get deployments
kubectl get services
kubectl port-forward service/hello 8080:80
```

**Key concept:** `selector.matchLabels` in a Deployment must match `template.metadata.labels`. Same pattern in a Service. Labels are the glue — get them wrong and nothing connects.

**AWS equivalent:** Pod = ECS Task. Deployment = ECS Service (desired count, restarts). Service = ALB Target Group.

---

### Phase 3 — Config and Secrets

Inject config into Pods without baking it into the image. Two methods: env vars and volume mounts.

```bash
kubectl apply -f configs/configmap-hello.yaml
kubectl apply -f configs/secrets-hello.yaml

# Verify env injection
kubectl exec -it <pod-name> -- env | grep APP

# Verify volume mount
kubectl exec -it <pod-name> -- ls /etc/config
```

**AWS equivalent:** ConfigMap = Parameter Store. Secret = Secrets Manager (plaintext locally — not encrypted at rest in kind).

---

### Phase 4 — Ingress

Route external HTTP traffic into the cluster by hostname. Requires installing the nginx-ingress controller first.

```bash
# Install the ingress controller for kind
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for it to be ready
kubectl get pods -n ingress-nginx -w

# Apply your Ingress rule
kubectl apply -f ingress/ingress-hello.yaml

# Test
curl -H "Host: hello.local" http://localhost:8080
```

**AWS equivalent:** Ingress = ALB with listener rules. Ingress Controller = the ALB itself. Ingress resource = the listener rule set.

> Troubleshooting: if the controller pod is `Pending`, check `kubectl describe pod -n ingress-nginx <pod>` — it likely needs the `ingress-ready=true` node label. The kind-config.yaml in this repo handles this automatically.

---

### Phase 5 — Namespaces and RBAC

Isolate workloads and apply least-privilege permissions. Same instincts as IAM.

```bash
kubectl apply -f namespaces/

# Verify permissions
kubectl auth can-i get pods -n hello --as=system:serviceaccount:hello:hello-sa    # yes
kubectl auth can-i delete pods -n hello --as=system:serviceaccount:hello:hello-sa # no
```

**AWS equivalent:** Namespace = ECS cluster isolation or AWS account boundary. ServiceAccount = IAM Role. Role = IAM Policy (namespace-scoped). RoleBinding = attaching a policy to a role.

---

### Phase 6 — Real App End-to-End

Everything together: Deployment with resource limits and health probes, Service, ConfigMap, Secret, Ingress, Namespace, RBAC.

```bash
kubectl apply -f namespaces/namespace-app.yaml
kubectl apply -f namespaces/serviceaccount-app.yaml
kubectl apply -f configs/configmap-app.yaml
kubectl apply -f configs/secrets-app.yaml
kubectl apply -f deployments/deployment-app.yaml
kubectl apply -f services/  # if not already applied
kubectl apply -f ingress/ingress-app.yaml

kubectl get all -n app
```

---

### Phase 7 — Helm + Argo CD (GitOps)

Package the Phase 6 app as a Helm chart. Deploy and auto-sync it via Argo CD from GitHub.

#### Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w    # wait for all pods Running
```

#### Log in to the UI

```bash
# Get the admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open `https://localhost:8080`, login as `admin`. Bypass the browser cert warning — expected for local installs.

#### Connect your GitHub repo (SSH deploy key)

```bash
# Generate a keypair scoped to this repo
ssh-keygen -t ed25519 -C "argocd-deploy-key" -f ~/.ssh/argocd_deploy

# Add the PUBLIC key to GitHub
cat ~/.ssh/argocd_deploy.pub
# → GitHub repo → Settings → Deploy keys → Add deploy key → read-only
```

In Argo CD UI: **Settings → Repositories → Connect Repo → Via SSH**
- URL: `git@github.com:you/k8s.git`
- SSH private key: paste `cat ~/.ssh/argocd_deploy`

#### Apply the Application manifest

Update `argocd/application.yaml` with your repo URL, then:

```bash
kubectl apply -f argocd/application.yaml
```

Argo CD will sync automatically. To prove the GitOps loop: change `replicaCount` in `charts/app/values.yaml`, push to GitHub, watch Argo CD reconcile without running `kubectl`.

#### Helm commands (for reference)

```bash
helm lint charts/app              # Validate
helm template charts/app          # Render locally
helm install app charts/app       # Manual install (bypasses Argo CD)
```

---

### Phase 8 — Prometheus + Grafana (Observability)

Install the kube-prometheus-stack — bundles Prometheus, Grafana, Alertmanager, and node exporters with pre-built dashboards.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace

# Check pods are up (takes ~2 minutes)
kubectl get pods -n monitoring -w
```

#### Access Grafana

```bash
# Get password
kubectl --namespace monitoring get secrets kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo

# Port-forward
export POD_NAME=$(kubectl --namespace monitoring get pod \
  -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=kube-prometheus-stack" -oname)
kubectl --namespace monitoring port-forward $POD_NAME 3000
```

Open `http://localhost:3000`, login as `admin`.

Navigate to **Kubernetes / Compute Resources / Cluster** for cluster-wide metrics, or **Kubernetes / Compute Resources / Namespace (Pods)** to see per-pod CPU and memory.

> Resource note: the kube-prometheus-stack is heavy. If pods crash-loop, you likely need more Colima resources: `colima stop && colima start --cpu 4 --memory 6`.

---

## Reference

`docs/k8s-cheatsheet.md` — complete reference covering every object type, commands, AWS mappings, and a troubleshooting guide for the most common failure states.

---

## AWS Translation Layer

| AWS / ECS | Kubernetes |
|---|---|
| ECS Task | Pod |
| ECS Service | Deployment |
| ALB Target Group | Service (ClusterIP) |
| ALB with listener rules | Ingress |
| Parameter Store | ConfigMap |
| Secrets Manager | Secret |
| IAM Role (task) | ServiceAccount + RBAC |
| ECS Cluster isolation | Namespace |
| CloudFormation template | Helm chart |
| CodePipeline (GitOps) | Argo CD |
| CloudWatch metrics | Prometheus |
| CloudWatch dashboards | Grafana |

---

## Useful Commands

```bash
# Cluster
colima start --cpu 4 --memory 6
kind create cluster --config kind-config.yaml
kind delete cluster --name k8s-learning
kubectl get nodes

# General
kubectl get all -n <namespace>
kubectl describe <type> <name> -n <namespace>
kubectl logs <pod> -n <namespace>
kubectl exec -it <pod> -n <namespace> -- bash

# Argo CD UI
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Grafana UI
kubectl --namespace monitoring get secrets kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```
