# Kubernetes Cheatsheet
> Built hands-on through Phase 8. AWS equivalents included throughout.

---

## Core Concepts

| K8s Object | AWS Equivalent | What it does |
|---|---|---|
| Pod | ECS Task | Smallest deployable unit. One or more containers sharing a network + storage. |
| Deployment | ECS Service | Manages Pods. Maintains desired replica count. Restarts failed Pods. |
| ReplicaSet | (managed by ECS Service internally) | Created by Deployment to manage Pod copies. You rarely touch this directly. |
| Service | ALB / Target Group | Stable network endpoint for a set of Pods. Load balances across replicas. |
| ConfigMap | Parameter Store | Inject non-sensitive config into Pods. |
| Secret | Secrets Manager | Inject sensitive values into Pods. |
| Namespace | ECS Cluster / AWS account isolation | Logical isolation within a cluster. |
| Ingress | ALB with routing rules | Routes external traffic to Services by hostname/path. |
| ServiceAccount + RBAC | IAM Role + least-privilege | Identity and permissions for Pods. |

---

## Pod

Smallest unit. Almost never created directly in production — use a Deployment instead.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello
  labels:
    app: hello        # Key-value tags. Used by Services/Deployments to find this Pod.
spec:
  containers:
    - name: hello
      image: nginx:1.28.2
      ports:
        - containerPort: 80   # Documents the port. Doesn't expose anything — that's a Service's job.
```

### Pod commands
```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod <name>       # Full detail + events — use when something's wrong
kubectl port-forward pod/<name> 8080:80
kubectl delete pod <name>
```

---

## Deployment

Manages Pods. Equivalent to ECS Service — maintains desired count, restarts on failure.

```yaml
apiVersion: apps/v1             # Note: apps/v1, not v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 2                   # Desired Pod count — like ECS desired count
  selector:
    matchLabels:
      app: hello                # How the Deployment finds its Pods
  template:                     # Pod template — everything below is the Pod spec
    metadata:
      labels:
        app: hello              # Must match selector.matchLabels
    spec:
      containers:
        - name: hello
          image: nginx:1.28.2
          ports:
            - containerPort: 80
```

**Label wiring:** `selector.matchLabels` → `template.metadata.labels` must match. This is how the Deployment knows which Pods it owns.

### Rolling updates

When you change the image tag and apply, K8s replaces Pods one at a time — no downtime.

```bash
kubectl set image deployment/<name> <container>=nginx:1.29.0   # Trigger a rollout
kubectl rollout status deployment/<name>                        # Watch progress
kubectl rollout history deployment/<name>                       # See previous versions
kubectl rollout undo deployment/<name>                          # Roll back one version
```

### Deployment commands
```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get replicasets           # See the layer Deployment creates between itself and Pods
kubectl describe deployment <name>
kubectl delete deployment <name>
```

---

## Service

Stable network endpoint. Routes traffic to Pods matching its selector. Load balances across replicas.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello          # Finds Pods with this label
  ports:
    - port: 80          # Port the Service listens on (inside the cluster)
      targetPort: 80    # Port on the Pod to forward to
  type: ClusterIP       # Only reachable inside the cluster
```

### Service types
| Type | Equivalent | Use |
|---|---|---|
| ClusterIP | Internal ALB | Default. Only reachable within the cluster. |
| NodePort | Opens a port on the node | Dev/testing. Not for production. |
| LoadBalancer | Public ALB | Provisions a cloud load balancer (AWS ELB when on EKS). |

### Service commands
```bash
kubectl apply -f service.yaml
kubectl get services
kubectl port-forward service/<name> 8080:80   # Port-forward to the Service, not a specific Pod
kubectl describe service <name>
```

---

## General Commands

```bash
# Cluster health
kubectl get nodes
kubectl get all                        # Pods, Deployments, Services, ReplicaSets in one shot

# Watching live
kubectl get pods -w                    # Watch mode — updates in place

# Debugging
kubectl describe <type> <name>         # Full detail + event log
kubectl logs <pod-name>                # Container stdout
kubectl logs <pod-name> -f             # Follow logs
kubectl exec -it <pod-name> -- bash    # Shell into a running container

# Cleanup
kubectl delete -f <file.yaml>          # Delete everything defined in the file
kubectl delete pod <name>
kubectl delete deployment <name>
kubectl delete service <name>
```

---

## Common Flags

| Flag | Command | What it does |
|---|---|---|
| `-f <file>` | `apply`, `delete` | Target a file (or directory) instead of a named resource |
| `-w` | `get` | Watch mode — streams live updates until Ctrl+C |
| `-f` | `logs` | Follow mode — streams logs live until Ctrl+C |
| `-it` | `exec` | Interactive terminal — `-i` keeps stdin open, `-t` allocates a TTY |
| `-n <namespace>` | most commands | Target a specific namespace (default is `default`) |
| `--all-namespaces` | `get` | Show resources across all namespaces |
| `-o wide` | `get` | Extra columns (node name, IP, etc.) |
| `-o yaml` | `get` | Output the full resource definition as YAML — useful for inspecting live state |

---

## ConfigMap

Inject non-sensitive config into Pods. Equivalent to Parameter Store. Two consumption methods:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-config
data:
  APP_ENV: production    # Values must be strings — wrap numbers in quotes
  APP_PORT: "80"
```

### Method 1: Env vars (envFrom)
Injects all keys as environment variables.
```yaml
containers:
  - name: hello
    envFrom:
      - configMapRef:
          name: hello-config    # Injects ALL keys as env vars
```
Verify: `kubectl exec -it <pod> -- env | grep APP`

### Method 2: Volume mount
Each key becomes a file inside the container. Good for config files.
```yaml
containers:
  - name: hello
    volumeMounts:
      - name: config-volume
        mountPath: /etc/config    # Directory created inside the container
volumes:
  - name: config-volume           # Must match volumeMounts.name
    configMap:
      name: hello-config          # Each key = file, value = file contents
```
Verify: `kubectl exec -it <pod> -- ls /etc/config`

### ConfigMap commands
```bash
kubectl apply -f configmap.yaml
kubectl get configmaps
kubectl describe configmap <name>
```

---

## Secret

Same as ConfigMap but base64-encoded at rest. Equivalent to Secrets Manager. Not encrypted by default in kind — treat as plaintext for local learning.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: hello-secret
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQ=    # base64-encoded — echo -n 'password' | base64
```

Encode/decode:
```bash
echo -n 'myvalue' | base64        # encode
echo 'bXl2YWx1ZQ==' | base64 -d  # decode
```

Consume identically to ConfigMap — use `secretRef` instead of `configMapRef`:
```yaml
envFrom:
  - secretRef:
      name: hello-secret
```

**Key difference from ConfigMap:** K8s will refuse to show Secret values in plain text via `kubectl get` — use `kubectl get secret <name> -o jsonpath="{.data.KEY}" | base64 -d` to read a value.

---

## Namespaces and RBAC

### Namespace
Logical isolation within a cluster. Same manifest can run in multiple namespaces as separate instances.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

```bash
kubectl get namespaces
kubectl apply -f manifest.yaml -n dev      # Deploy into a specific namespace
kubectl get pods -n dev                    # View resources in a namespace
```

Built-in namespaces — don't deploy into these:

| Namespace | Purpose |
|---|---|
| `kube-system` | Control plane components (API server, scheduler, CoreDNS) |
| `kube-public` | Readable by all users, including unauthenticated |
| `kube-node-lease` | Node heartbeats — how the control plane knows nodes are alive |

Every namespace gets a `default` ServiceAccount automatically. Pods use it if none is specified.

### RBAC

Three resources — same mental model as IAM:

| K8s | AWS |
|---|---|
| ServiceAccount | IAM Role |
| Role | IAM Policy (namespace-scoped) |
| RoleBinding | Attaching a policy to a role |
| ClusterRole | IAM Policy (cluster-wide) |

```yaml
# ServiceAccount — the identity
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
  namespace: dev

---
# Role — the permissions (namespace-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]        # "" = core API group (Pods, Services, ConfigMaps, etc.)
  resources: ["pods"]
  verbs: ["get", "list", "watch"]   # read-only

---
# RoleBinding — glues them together
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: dev
subjects:
- kind: ServiceAccount
  name: my-sa
  namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Verify permissions
```bash
kubectl auth can-i get pods -n dev --as=system:serviceaccount:dev:my-sa    # yes
kubectl auth can-i delete pods -n dev --as=system:serviceaccount:dev:my-sa # no
```

---

## Ingress Controller (nginx-ingress)

Routes external traffic into the cluster by hostname/path rules. AWS equivalent: ALB with listener rules.

Two layers:
- **Ingress resource** — your routing rules (what you write, `kind: Ingress`)
- **Ingress Controller** — the thing that reads those rules and programs nginx (runs as a Deployment in `ingress-nginx` namespace)

### Admission webhooks (the Completed pods)

When you install ingress-nginx you'll see two one-shot pods:

| Pod | What it does |
|---|---|
| `admission-create` | Registers a `ValidatingWebhookConfiguration` — tells the API server to call the controller before accepting any Ingress object (validation gate) |
| `admission-patch` | Patches the webhook config with a TLS cert so the API server trusts the callback |

Both run once and exit `Completed`. That's expected — not a failure.

### Why the controller pod gets stuck Pending (kind-specific)

The kind ingress-nginx manifest ships with a `nodeSelector` that requires the node to have the label `ingress-ready=true`. The scheduler won't place the pod until a matching node exists.

Fix:
```bash
kubectl label node <node-name> ingress-ready=true
```

After labelling: `Pending` → `ContainerCreating` → `Running`.

**AWS equivalent:** ECS placement constraint — a task stays `PENDING` until an instance matching the constraint (e.g. `attribute:ecs.instance-type == m5.large`) is available in the cluster.

### Ingress Controller commands
```bash
kubectl get pods -n ingress-nginx                        # Check controller status
kubectl describe pod -n ingress-nginx <controller-pod>  # Debug Pending/CrashLoop
kubectl get ingressclass                                 # Verify nginx IngressClass registered
kubectl get ingress                                      # List your Ingress rules
kubectl describe ingress <name>                          # See which rules are active + events
```

---

## Resource Limits

Equivalent to ECS task CPU/memory settings. Two concepts:
- **requests** — minimum guaranteed. Used by the scheduler to decide which node to place the Pod on.
- **limits** — hard ceiling. Container is killed (OOMKilled) or throttled if it exceeds this.

```yaml
containers:
  - name: app
    resources:
      requests:
        memory: "64Mi"   # Scheduler needs this much free on the node
        cpu: "200m"      # 200 millicores = 0.2 vCPU
      limits:
        memory: "128Mi"  # Pod is OOMKilled if it exceeds this
        cpu: "200m"      # CPU is throttled (not killed) if it exceeds this
```

**CPU units:** `1000m` = 1 vCPU. `200m` = 20% of one core.

**Rule of thumb:** set `requests` to what the app needs at idle, `limits` to what it needs at peak. Never set limits lower than requests.

### Resource commands
```bash
kubectl describe node <name>          # See total allocatable CPU/memory and what's claimed
kubectl top pods                      # Live CPU + memory usage (requires metrics-server)
kubectl top nodes
```

---

## Health Probes

Tell K8s whether your container is alive and ready to receive traffic. AWS equivalent: ALB target health checks.

Two probes:
| Probe | Fails → | AWS equivalent |
|---|---|---|
| `livenessProbe` | Pod is killed and restarted | Nothing — ECS doesn't have this natively |
| `readinessProbe` | Pod removed from Service endpoints (traffic stops) | ALB target health check |

```yaml
containers:
  - name: app
    livenessProbe:
      httpGet:
        path: /        # K8s hits this endpoint
        port: 8080
      initialDelaySeconds: 5    # Wait before first check (give the app time to start)
      periodSeconds: 10         # Check every 10s
    readinessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
```

**Why both?** Liveness restarts a stuck/deadlocked container. Readiness prevents traffic from hitting a container that's still starting up or temporarily overloaded. A Pod can be live but not ready.

**Other probe types:**
- `tcpSocket` — checks if the port accepts connections (good for non-HTTP services)
- `exec` — runs a command inside the container; exit 0 = healthy

### Probe debugging
```bash
kubectl describe pod <name>     # Events section shows probe failures
kubectl get events              # Cluster-wide events — CrashLoopBackOff, probe failures, etc.
```

---

## Helm

Package manager for Kubernetes. Equivalent to CloudFormation templates + a parameters file — the chart is the template, `values.yaml` is the parameters.

### Structure
```
charts/app/
├── Chart.yaml          # Chart metadata (name, version)
├── values.yaml         # Default values — override at deploy time
└── templates/          # Manifests with {{ .Values.x }} substitution
    ├── deployment.yaml
    ├── service.yaml
    └── ...
```

### values.yaml
```yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.28.2"
```

### Template substitution
```yaml
# templates/deployment.yaml
replicas: {{ .Values.replicaCount }}
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

### Helm commands
```bash
helm lint charts/app                        # Validate chart structure and syntax
helm template charts/app                    # Render templates locally — see the output YAML
helm install my-release charts/app          # Install into the cluster
helm upgrade my-release charts/app          # Apply changes
helm uninstall my-release                   # Remove everything the chart installed
helm list                                   # List installed releases
```

---

## Argo CD (GitOps)

Argo CD watches a Git repo and keeps cluster state in sync with it. AWS equivalent: CodePipeline watching a repo, but stricter — the repo is the source of truth and drift is auto-corrected.

**GitOps loop:**
1. You push a change to Git
2. Argo CD detects the diff between Git state and cluster state
3. Argo CD applies the change — no `kubectl` needed

### Install
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd     # Wait for all 6 deployments to be Running
```

### Log in to the UI
```bash
# Get the auto-generated admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port-forward the UI (runs in foreground — keep terminal open)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Open `https://localhost:8080`, login as `admin`. Browser cert warning is expected — bypass it.

### Application manifest
The `Application` resource tells Argo CD what to watch and where to deploy it.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd           # Always deployed into the argocd namespace
spec:
  project: default
  source:
    repoURL: https://github.com/you/your-repo
    targetRevision: HEAD       # Branch or tag to watch
    path: charts/app           # Path to the Helm chart (or plain manifests)
  destination:
    server: https://kubernetes.default.svc   # The local cluster
    namespace: default
  syncPolicy:
    automated:
      selfHeal: true           # Revert manual kubectl changes — drift correction
      prune: true              # Delete resources removed from Git
```

### Key concepts
| Concept | AWS equivalent |
|---|---|
| Helm chart | CloudFormation template |
| `values.yaml` | CloudFormation parameters / Terraform `.tfvars` |
| GitOps | CodePipeline watching a repo, but repo is sole source of truth |
| Argo CD Application | Pipeline definition |
| `selfHeal: true` | Drift detection + auto-remediation (like AWS Config rules that auto-remediate) |
| `prune: true` | Resources deleted from Git are deleted from the cluster |

### GitHub SSH deploy key setup

Argo CD needs read access to your repo. Use a deploy key — scoped to one repo, read-only.

```bash
# 1. Generate a keypair (don't use your personal SSH key)
ssh-keygen -t ed25519 -C "argocd-deploy-key"

# 2. Add the PUBLIC key to GitHub
cat ~/.ssh/id_ed25519.pub
# GitHub repo → Settings → Deploy keys → Add deploy key → paste → read-only
```

In Argo CD UI: **Settings → Repositories → Connect Repo → Via SSH**
- Repository URL: `git@github.com:you/repo.git` (SSH format, not HTTPS)
- SSH private key: paste output of `cat ~/.ssh/id_ed25519`

### Argo CD commands
```bash
kubectl get applications -n argocd                  # List all Applications
kubectl describe application <name> -n argocd       # Full sync status + events
kubectl get pods -n argocd                          # Check Argo CD health
```

---

## Prometheus + Grafana (Observability)

AWS equivalent: CloudWatch metrics + CloudWatch dashboards — except open source and running inside your cluster.

**Split of responsibilities:**
- **Prometheus** — scrapes and stores metrics (pull-based)
- **Grafana** — visualizes metrics from Prometheus

**Pull model:** Prometheus polls `/metrics` endpoints on a schedule. Your app exposes the endpoint; Prometheus finds it via labels and scrapes it. Nothing is pushed.

### Install (kube-prometheus-stack)
Bundles Prometheus, Grafana, Alertmanager, node exporters, and pre-built dashboards.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

### Access Grafana
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

### Key concepts
| Concept | AWS equivalent |
|---|---|
| Prometheus | CloudWatch metrics |
| Grafana | CloudWatch dashboards |
| Node exporter | CloudWatch agent on EC2 |
| ServiceMonitor | Tells Prometheus which Services to scrape |
| `/metrics` endpoint | Custom CloudWatch metrics endpoint |

### Adding custom metrics
To scrape your own app, it needs to expose a `/metrics` endpoint, plus a `ServiceMonitor` resource:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: my-app       # Finds Services with this label
  endpoints:
    - port: http
      path: /metrics
```

### Commands
```bash
kubectl get pods -n monitoring                    # Check stack health
kubectl get servicemonitors -n monitoring         # List scrape targets
```

---

## Cluster Setup (kind + Colima)

Local stack: **Colima** (lightweight VM) → **Docker** → **kind** (K8s in Docker).

```bash
# Start Colima with enough resources for a full stack
colima start --cpu 4 --memory 6

# Create the cluster (with ingress support)
kind create cluster --name k8s-learning --config kind-config.yaml

# Check it's up
kubectl get nodes
```

**kind-config.yaml** (saves the ingress-ready label and port mapping):
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

### Cluster commands
```bash
kind get clusters                          # List clusters
kind delete cluster --name k8s-learning   # Delete cluster
colima stop                                # Stop the VM
colima start --cpu 4 --memory 6           # Restart with more resources
kubectl config get-contexts               # See available clusters
kubectl config use-context <name>         # Switch cluster
```

---

## Troubleshooting

### Pod stuck in Pending
Scheduler can't place the Pod. Common causes:

| Cause | How to confirm | Fix |
|---|---|---|
| Not enough CPU/memory | `kubectl describe node` → Allocated resources | Scale down other pods or increase Colima resources |
| Missing node label | `kubectl describe pod` → Events: "didn't match node selector" | `kubectl label node <name> ingress-ready=true` |
| No nodes available | `kubectl get nodes` shows NotReady | Restart Colima/kind |

```bash
kubectl describe pod <name>     # Events section tells you exactly why it's Pending
```

### Pod in CrashLoopBackOff
Container starts and immediately exits. K8s keeps restarting it with exponential backoff.

```bash
kubectl logs <pod>              # Check what the app printed before dying
kubectl logs <pod> --previous  # Logs from the previous (crashed) container
kubectl describe pod <pod>     # Exit code in Last State — 137 = OOMKilled, 1 = app error
```

Common causes: bad config/env vars, app crash on startup, liveness probe firing too early (`initialDelaySeconds` too low).

### Pod in ImagePullBackOff
K8s can't pull the container image.

```bash
kubectl describe pod <name>    # Events: "Failed to pull image"
```

Common causes: typo in image name, wrong tag, private registry not authenticated, no internet from inside kind.

### Port-forward fails: "pod is not running"
The pod isn't ready yet. Wait for `READY` to show `n/n`:
```bash
kubectl get pods -n <namespace> -w    # Watch until fully ready, then Ctrl+C and retry
```

### Helm install fails after laptop sleep
TLS handshake timeout — the cluster connection dropped. Clean up and retry:
```bash
helm list -n <namespace>                    # Check if release exists in a broken state
helm uninstall <release> -n <namespace>     # Clean up if it does
helm install ...                            # Retry
```

### Argo CD: "ssh: no key found"
You connected the repo with an HTTPS URL but Argo CD tried SSH. The repo URL must match the auth method:
- SSH auth → `git@github.com:you/repo.git`
- HTTPS auth → `https://github.com/you/repo.git`

### Argo CD: app shows OutOfSync but won't sync
Check the app events:
```bash
kubectl describe application <name> -n argocd
```
Common cause: `prune: false` and a resource was removed from Git — Argo CD won't delete it without prune enabled.

### Nodes show NotReady after Colima restart
Colima restarted but the kind cluster lost connectivity. Check:
```bash
colima status
kubectl get nodes
```
If nodes are NotReady, restart Colima: `colima stop && colima start --cpu 4 --memory 6`.

---

## Key Behaviours to Remember

- **Pods are ephemeral.** Never rely on a specific Pod being there. That's what Services are for.
- **Applying a changed image to a Pod doesn't update it** — K8s terminates and recreates it. That's the `RESTARTS` counter.
- **`containerPort` is documentation only.** It doesn't open a firewall rule. Traffic routing is the Service's job.
- **Pin image versions.** `latest` is non-deterministic. Use `nginx:1.28.2` not `nginx:latest`.
- **Labels are the glue.** Deployments find Pods via labels. Services find Pods via labels. Get them wrong and nothing connects.
- **`ingressClassName` is a controller selector.** A cluster can run multiple Ingress Controllers. This field tells K8s which one owns your Ingress resource. Omit it and no controller claims it — rules are never enforced.
- **Ingress ≠ Ingress Controller.** The Ingress resource is just routing rules. The controller is the process that reads and enforces them. Both must exist.
- **Requests ≠ limits.** Requests are what the scheduler uses to place the Pod. Limits are the hard ceiling. Set both — omitting either is a footgun.
- **Liveness ≠ readiness.** Liveness failure → restart. Readiness failure → pulled from Service endpoints. A stuck app needs liveness. A slow-starting app needs readiness. Most apps need both.
- **`initialDelaySeconds` matters.** Without it, probes fire before the app is up and trigger a restart loop. Set it to longer than your app's startup time.
