# Kubernetes Cheatsheet
> Built hands-on during Phase 2 local learning. AWS equivalents included throughout.

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

## Key Behaviours to Remember

- **Pods are ephemeral.** Never rely on a specific Pod being there. That's what Services are for.
- **Applying a changed image to a Pod doesn't update it** — K8s terminates and recreates it. That's the `RESTARTS` counter.
- **`containerPort` is documentation only.** It doesn't open a firewall rule. Traffic routing is the Service's job.
- **Pin image versions.** `latest` is non-deterministic. Use `nginx:1.28.2` not `nginx:latest`.
- **Labels are the glue.** Deployments find Pods via labels. Services find Pods via labels. Get them wrong and nothing connects.
- **`ingressClassName` is a controller selector.** A cluster can run multiple Ingress Controllers. This field tells K8s which one owns your Ingress resource. Omit it and no controller claims it — rules are never enforced.
- **Ingress ≠ Ingress Controller.** The Ingress resource is just routing rules. The controller is the process that reads and enforces them. Both must exist.
