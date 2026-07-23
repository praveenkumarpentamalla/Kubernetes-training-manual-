# Kubernetes Gap Topics — Hands-On Production Guide

**Companion to the 5-Volume Kubernetes Training Manual**

Every section below is a real, runnable walkthrough: setup, exact commands, exact YAML, and realistic expected output — the way you'd actually do this in production. Where a topic genuinely requires a real cloud account (AWS EKS, a multi-VM environment) and cannot run on your local Kind cluster from Volume 1, this is stated explicitly up front, and the commands/output shown are exactly what you'd run/see in that real environment.

**Environment assumed for the runnable sections:** Your `my-first-cluster` Kind cluster from Volume 1 (Kubernetes v1.31), with `kubectl` configured against it.

---

## 1. Helm

**Runs on:** Your local Kind cluster. Fully hands-on.

### 1.1 Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

**Expected output:**
```
version.BuildInfo{Version:"v3.15.4", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.22.6"}
```

### 1.2 Add a real, production-grade chart repository

We'll install the **ingress-nginx** controller — the same one referenced in Volume 3 Lab 5, but this time via Helm instead of a raw manifest, exactly how most real teams install it.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

**Expected output:**
```
"ingress-nginx" has been added to your repositories
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ingress-nginx" chart repository
Update Complete. ⎈ Happy Helming!⎈
```

### 1.3 Inspect the chart before installing (production habit — never install blind)

```bash
helm show values ingress-nginx/ingress-nginx | head -20
```

**Expected output (excerpt):**
```yaml
controller:
  image:
    registry: registry.k8s.io
    image: ingress-nginx/controller
    tag: "v1.11.2"
  replicaCount: 1
  service:
    type: LoadBalancer
```

This is exactly the `values.yaml` concept explained earlier — every one of these fields can be overridden at install time.

### 1.4 Install the chart, overriding one value

```bash
helm install my-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.replicaCount=1
```

**Command explanation:**
- `helm install my-ingress ingress-nginx/ingress-nginx` — `my-ingress` is the **release name** you're choosing; `ingress-nginx/ingress-nginx` is `<repo-name>/<chart-name>`.
- `--namespace ingress-nginx --create-namespace` — installs into a namespace that doesn't exist yet, creating it first.
- `--set controller.replicaCount=1` — overrides one specific value from `values.yaml` without needing a whole custom file, appropriate for a single-node Kind cluster.

**Expected output:**
```
NAME: my-ingress
LAST DEPLOYED: Wed Jul 23 10:02:14 2026
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
```

### 1.5 Verify like a real engineer would

```bash
helm list -n ingress-nginx
kubectl get pods -n ingress-nginx
```

**Expected output:**
```
NAME         NAMESPACE      REVISION  UPDATED                   STATUS    CHART                    APP VERSION
my-ingress   ingress-nginx  1         2026-07-23 10:02:14        deployed  ingress-nginx-4.11.2     1.11.2

NAME                                        READY   STATUS    RESTARTS   AGE
my-ingress-ingress-nginx-controller-7d9f8c  1/1     Running   0          45s
```

### 1.6 Upgrade with a values file (the real production pattern)

Create `prod-values.yaml`:
```yaml
controller:
  replicaCount: 2
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 250m
      memory: 256Mi
```

```bash
helm upgrade my-ingress ingress-nginx/ingress-nginx \
  -n ingress-nginx -f prod-values.yaml
```

**Expected output:**
```
Release "my-ingress" has been upgraded. Happy Helming!
NAME: my-ingress
REVISION: 2
```

### 1.7 Rollback (exactly like `kubectl rollout undo`, but for the whole chart)

```bash
helm history my-ingress -n ingress-nginx
```

**Expected output:**
```
REVISION  UPDATED                   STATUS      CHART                   APP VERSION  DESCRIPTION
1         Wed Jul 23 10:02:14 2026  superseded  ingress-nginx-4.11.2    1.11.2       Install complete
2         Wed Jul 23 10:05:40 2026  deployed    ingress-nginx-4.11.2    1.11.2       Upgrade complete
```

```bash
helm rollback my-ingress 1 -n ingress-nginx
```

**Expected output:**
```
Rollback was a success! Happy Helming!
```

### 1.8 Build your own chart (production scaffolding)

```bash
helm create myapp
tree myapp
```

**Expected output:**
```
myapp
├── Chart.yaml
├── values.yaml
├── charts/
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── _helpers.tpl
    ├── NOTES.txt
    └── tests/
        └── test-connection.yaml
```

```bash
helm install myapp-dev ./myapp --set replicaCount=2
kubectl get deployment myapp-dev-myapp
```

**Expected output:**
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
myapp-dev-myapp    2/2     2            2           10s
```

**Cleanup:**
```bash
helm uninstall myapp-dev
helm uninstall my-ingress -n ingress-nginx
```

---

## 2. GitOps — Argo CD

**Runs on:** Your local Kind cluster (using a public demo Git repo). Fully hands-on.

### 2.1 Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=180s
```

**Expected output:**
```
namespace/argocd created
customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io created
...
deployment.apps/argocd-server condition met
```

### 2.2 Access the Argo CD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
```

**Expected output:**
```
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

Get the auto-generated admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

**Expected output:**
```
aB3xK9pQz2Lm8wRt
```

Log in at `https://localhost:8080` with username `admin` and this password (accept the self-signed cert warning).

### 2.3 Define an Application (declaring "this Git repo is my source of truth")

```bash
cat > guestbook-app.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
kubectl apply -f guestbook-app.yaml
```

**YAML explanation:**
- `source.repoURL` / `path` — the Git repo and folder Argo CD should treat as the desired state.
- `destination` — where in the cluster to apply it.
- `syncPolicy.automated.selfHeal: true` — if someone manually changes a resource in the cluster, Argo CD will automatically revert it back to match Git.
- `prune: true` — if a manifest is removed from Git, Argo CD deletes the corresponding resource from the cluster too.

**Expected output:**
```
application.argoproj.io/guestbook created
```

### 2.4 Watch it sync

```bash
kubectl get application guestbook -n argocd -o jsonpath='{.status.sync.status}{"\n"}{.status.health.status}{"\n"}'
```

**Expected output (after ~30 seconds):**
```
Synced
Healthy
```

```bash
kubectl get pods -n default -l app.kubernetes.io/instance=guestbook
```

**Expected output:**
```
NAME                        READY   STATUS    RESTARTS   AGE
guestbook-ui-6d9f8c-abcde   1/1     Running   0          30s
```

### 2.5 Prove self-healing (the core GitOps guarantee)

```bash
kubectl scale deployment guestbook-ui --replicas=5 -n default
sleep 20
kubectl get deployment guestbook-ui -n default
```

**Expected output:**
```
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
guestbook-ui    1/1     1            1           2m
```

Notice `READY` is back to `1/1`, not `5/5` — Argo CD detected your manual change deviated from Git and reverted it automatically, exactly as `selfHeal: true` promised.

**Cleanup:**
```bash
kubectl delete -f guestbook-app.yaml
kubectl delete namespace argocd
```

---

## 3. Monitoring — Prometheus, Grafana, Metrics Server

**Runs on:** Your local Kind cluster. Fully hands-on.

### 3.1 Metrics Server (you may already have this from Volume 1 Lab 4)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Kind needs one patch (metrics-server can't verify Kind's self-signed kubelet certs by default):
```bash
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

**Verify:**
```bash
kubectl top nodes
```

**Expected output:**
```
NAME                              CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
my-first-cluster-control-plane    145m         3%     620Mi           15%
my-first-cluster-worker           98m          2%     410Mi           10%
```

### 3.2 Install the full Prometheus + Grafana stack via Helm

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.adminPassword=admin123
```

**Expected output:**
```
NAME: monitoring
LAST DEPLOYED: Wed Jul 23 10:20:01 2026
NAMESPACE: monitoring
STATUS: deployed
NOTES:
kube-prometheus-stack has been installed.
```

**Verify (this takes 1–2 minutes to become fully Ready):**
```bash
kubectl get pods -n monitoring
```

**Expected output:**
```
NAME                                                     READY   STATUS    RESTARTS   AGE
monitoring-grafana-6d4b8f-abcde                          3/3     Running   0          90s
monitoring-kube-prometheus-operator-7c9d8f-fghij         1/1     Running   0          90s
prometheus-monitoring-kube-prometheus-prometheus-0       2/2     Running   0          85s
alertmanager-monitoring-kube-prometheus-alertmanager-0   2/2     Running   0          85s
monitoring-kube-state-metrics-6f8b9d-klmno               1/1     Running   0          90s
```

Notice `prometheus-...-0` and `alertmanager-...-0` — the trailing `-0` gives away that these are **StatefulSets** (Volume 2 Lab 8), exactly as you'd expect for something needing stable storage.

### 3.3 Query Prometheus directly

```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090 &
curl -s 'http://localhost:9090/api/v1/query?query=up' | python3 -m json.tool | head -20
```

**Expected output (excerpt):**
```json
{
    "status": "success",
    "data": {
        "resultType": "vector",
        "result": [
            {
                "metric": {
                    "__name__": "up",
                    "job": "apiserver"
                },
                "value": [1721728820, "1"]
            }
        ]
    }
}
```

`"value": [..., "1"]` means that target is up (`0` would mean Prometheus can't reach it) — this is the exact same `up` metric real production dashboards alert on.

### 3.4 View Grafana dashboards

```bash
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80 &
```

Open `http://localhost:3000`, log in with `admin` / `admin123`. Navigate to **Dashboards** — you'll see pre-built dashboards like "Kubernetes / Compute Resources / Cluster" already populated with real data from your own cluster, no manual setup required — this is the standard `kube-prometheus-stack` chart's biggest selling point.

**Cleanup:**
```bash
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```

---

## 4. Logging — Loki

**Runs on:** Your local Kind cluster. Fully hands-on.

### 4.1 Install Loki + Promtail (the log-shipping agent, a DaemonSet — Volume 2 Lab 7)

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install loki grafana/loki-stack \
  --namespace logging --create-namespace \
  --set grafana.enabled=true
```

**Expected output:**
```
NAME: loki
STATUS: deployed
```

**Verify:**
```bash
kubectl get pods -n logging
```

**Expected output:**
```
NAME                          READY   STATUS    RESTARTS   AGE
loki-0                        1/1     Running   0          60s
loki-promtail-4vxsk           1/1     Running   0          60s
loki-promtail-p7wjq           1/1     Running   0          60s
loki-grafana-6c9f8d-abcde     2/2     Running   0          60s
```

Notice `loki-promtail-*` has **two** Pods — one per node, confirming it's a DaemonSet exactly as Volume 2 Lab 7 taught, collecting logs from every node.

### 4.2 Query logs via LogQL (Loki's query language)

```bash
kubectl port-forward svc/loki -n logging 3100:3100 &
curl -s -G "http://localhost:3100/loki/api/v1/query" \
  --data-urlencode 'query={namespace="kube-system"}' \
  --data-urlencode 'limit=5' | python3 -m json.tool | head -15
```

**Expected output (excerpt):**
```json
{
    "status": "success",
    "data": {
        "resultType": "streams",
        "result": [
            {
                "stream": {
                    "namespace": "kube-system",
                    "app": "kube-dns"
                },
                "values": [
                    ["1721728900000000000", "[INFO] plugin/ready: ready"]
                ]
            }
        ]
    }
}
```

This proves Loki is centrally aggregating logs from every namespace, without you running `kubectl logs` against individual Pods — exactly the scaling problem this section's introduction described.

**Cleanup:**
```bash
helm uninstall loki -n logging
kubectl delete namespace logging
```

---

## 5. Horizontal Pod Autoscaler (HPA)

**Runs on:** Your local Kind cluster (requires Metrics Server from Section 3.1). Fully hands-on.

### 5.1 Deploy a target application with resource requests set

HPA cannot function without `resources.requests` defined — this is what percentages are calculated against.

```bash
cat > hpa-demo-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
spec:
  replicas: 1
  selector:
    matchLabels: { app: hpa-demo }
  template:
    metadata:
      labels: { app: hpa-demo }
    spec:
      containers:
        - name: php-apache
          image: registry.k8s.io/hpa-example
          ports: [{ containerPort: 80 }]
          resources:
            requests: { cpu: "200m" }
            limits: { cpu: "500m" }
---
apiVersion: v1
kind: Service
metadata: { name: hpa-demo-svc }
spec:
  selector: { app: hpa-demo }
  ports: [{ port: 80 }]
EOF
kubectl apply -f hpa-demo-deployment.yaml
```

**Expected output:**
```
deployment.apps/hpa-demo created
service/hpa-demo-svc created
```

### 5.2 Create the HPA

```bash
kubectl autoscale deployment hpa-demo --cpu-percent=50 --min=1 --max=5
```

**Command explanation:**
- `--cpu-percent=50` — target: keep average CPU usage across all replicas at 50% of their `requests.cpu`.
- `--min=1 --max=5` — never scale below 1 replica or above 5.

**Expected output:**
```
horizontalpodautoscaler.autoscaling/hpa-demo autoscaled
```

**Verify:**
```bash
kubectl get hpa hpa-demo
```

**Expected output:**
```
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo   Deployment/hpa-demo   0%/50%    1         5         1          15s
```

**Column explanation:**
- `TARGETS` — current usage versus your target, live. `0%/50%` means currently idle.
- `REPLICAS` — current actual replica count, which HPA will adjust automatically.

### 5.3 Generate load and watch it scale

In one terminal, generate load:
```bash
kubectl run load-generator --image=busybox --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://hpa-demo-svc; done"
```

In another terminal, watch:
```bash
kubectl get hpa hpa-demo --watch
```

**Expected output (over the next 1–2 minutes):**
```
NAME       REFERENCE             TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo   Deployment/hpa-demo   0%/50%      1         5         1          40s
hpa-demo   Deployment/hpa-demo   210%/50%    1         5         1          55s
hpa-demo   Deployment/hpa-demo   210%/50%    1         5         4          70s
hpa-demo   Deployment/hpa-demo   65%/50%     1         5         4          100s
hpa-demo   Deployment/hpa-demo   48%/50%     1         5         5          130s
```

This is HPA driving `spec.replicas` automatically — the exact same field from Volume 2 Lab 4 that you previously changed by hand with `kubectl scale`.

**Cleanup:**
```bash
kubectl delete pod load-generator
kubectl delete hpa hpa-demo
kubectl delete -f hpa-demo-deployment.yaml
```

---

## 6. Cluster Autoscaler / Karpenter

**Runs on:** A real cloud cluster only (AWS EKS shown). **Cannot run on Kind** — Kind's "nodes" are Docker containers on your one laptop; there's no cloud API to provision a new physical/virtual machine from. The commands and output below are exactly what you would run/see on real EKS infrastructure.

### 6.1 Cluster Autoscaler — installing via Helm on EKS

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=my-eks-cluster \
  --set awsRegion=us-east-1
```

**Expected output:**
```
NAME: cluster-autoscaler
STATUS: deployed
```

### 6.2 Trigger a scale-up (deploy more than current nodes can hold)

```bash
kubectl create deployment scale-test --image=nginx:1.27 --replicas=20 \
  --requests=cpu=1
kubectl get pods -l app=scale-test
```

**Expected output (some Pods immediately Pending):**
```
NAME                          READY   STATUS    RESTARTS   AGE
scale-test-7d9f8c-a1b2c       0/1     Pending   0          10s
scale-test-7d9f8c-a1b2d       0/1     Pending   0          10s
scale-test-7d9f8c-a1b2e       1/1     Running   0          10s
```

**Watch Cluster Autoscaler react (its own logs):**
```bash
kubectl logs -n kube-system deployment/cluster-autoscaler --tail=20
```

**Expected output:**
```
I0723 10:45:02.1  scale_up.go:376] Scale-up: setting group eks-worker-nodes-asg size to 5
I0723 10:45:03.7  cloud_provider_aws.go:120] Setting asg size for eks-worker-nodes-asg to 5
```

**Expected outcome after ~2–3 minutes:** A new EC2 instance appears, joins the cluster, and the `Pending` Pods move to `Running`.

### 6.3 Karpenter — declarative node provisioning (no fixed ASG needed)

Karpenter uses a `NodePool` object instead of a traditional node group:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 100
```

```bash
kubectl apply -f nodepool.yaml
kubectl get nodepool
```

**Expected output:**
```
NAME      NODECLASS
default   default
```

When Pods are `Pending`, Karpenter directly launches a right-sized EC2 instance (choosing the exact instance type that fits, rather than scaling a pre-defined group):
```bash
kubectl logs -n karpenter deployment/karpenter --tail=10
```

**Expected output:**
```
{"level":"INFO","message":"launched new node","NodeClaim":"default-x7k2p","instance-type":"t3.large","zone":"us-east-1a"}
```

---

## 7. Pod Disruption Budgets (PDBs)

**Runs on:** Your local Kind cluster. Fully hands-on.

### 7.1 Deploy an application with 3 replicas

```bash
kubectl create deployment pdb-demo --image=nginx:1.27 --replicas=3
kubectl get pods -l app=pdb-demo -o wide
```

**Expected output:**
```
NAME                       READY   STATUS    NODE
pdb-demo-6d9f8c-abcde      1/1     Running   my-first-cluster-worker
pdb-demo-6d9f8c-fghij      1/1     Running   my-first-cluster-worker
pdb-demo-6d9f8c-klmno      1/1     Running   my-first-cluster-worker
```

### 7.2 Create a PodDisruptionBudget

```bash
cat > pdb.yaml << 'EOF'
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: pdb-demo-budget
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: pdb-demo
EOF
kubectl apply -f pdb.yaml
```

**YAML explanation:**
- `minAvailable: 2` — at least 2 of the 3 replicas must remain available during any *voluntary* disruption (drain, autoscaler scale-down); Kubernetes will block an eviction that would drop below this.
- `selector` — exactly like a Service, this identifies which Pods the budget applies to via labels.

**Expected output:**
```
poddisruptionbudget.policy/pdb-demo-budget created
```

**Verify:**
```bash
kubectl get pdb pdb-demo-budget
```

**Expected output:**
```
NAME               MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
pdb-demo-budget    2               N/A               1                     15s
```

`ALLOWED DISRUPTIONS: 1` means exactly one Pod could be safely evicted right now without violating the budget.

### 7.3 Prove it blocks an unsafe drain

Since all 3 Pods are on your single worker node, draining it would try to evict all 3 at once — violating `minAvailable: 2`.

```bash
kubectl drain my-first-cluster-worker --ignore-daemonsets --delete-emptydir-data
```

**Expected output:**
```
node/my-first-cluster-worker cordoned
evicting pod default/pdb-demo-6d9f8c-abcde
error when evicting pods/"pdb-demo-6d9f8c-fghij" -n "default" (will retry after 5s): Cannot evict pod as it would violate the pod's disruption budget.
```

Kubernetes evicted **one** Pod (bringing the count to 2, satisfying `minAvailable: 2`), then **refused** to evict any more — exactly the protection Volume 5 Lab 3's drain lab referenced but didn't fully demonstrate.

**Cleanup:**
```bash
kubectl uncordon my-first-cluster-worker
kubectl delete -f pdb.yaml
kubectl delete deployment pdb-demo
```

---

## 8. Security Contexts

**Runs on:** Your local Kind cluster. Fully hands-on.

### 8.1 Deploy a Pod WITHOUT a security context (the insecure default)

```bash
cat > insecure-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata: { name: insecure-pod }
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "id; sleep 3600"]
EOF
kubectl apply -f insecure-pod.yaml
kubectl logs insecure-pod
```

**Expected output:**
```
uid=0(root) gid=0(root) groups=0(root)
```

This container is running as `root` — a real security risk if this image or application were ever compromised.

### 8.2 Deploy the hardened version

```bash
cat > secure-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata: { name: secure-pod }
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "id; sleep 3600"]
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      volumeMounts:
        - { name: tmp, mountPath: /tmp }
  volumes:
    - name: tmp
      emptyDir: {}
EOF
kubectl apply -f secure-pod.yaml
```

**YAML line-by-line (new fields only):**
- `spec.securityContext.runAsNonRoot: true` — Pod-level: refuse to start if the image would default to root.
- `runAsUser: 1000` — force this specific, unprivileged user ID.
- `fsGroup: 2000` — any mounted volumes are group-owned by this GID, so the non-root user can still read/write them.
- `containers[].securityContext.allowPrivilegeEscalation: false` — container-level: block privilege escalation, even via a setuid binary inside the image.
- `readOnlyRootFilesystem: true` — the container's own filesystem is immutable; this is why we had to explicitly mount `/tmp` as an `emptyDir` (Volume 4 Lab 3) — the app needs *somewhere* writable.
- `capabilities.drop: ["ALL"]` — remove every Linux capability; add back individually with `add:` only if something specific is genuinely needed.

**Expected output:**
```
pod/secure-pod created
```

```bash
kubectl logs secure-pod
```

**Expected output:**
```
uid=1000 gid=0(root) groups=0(root),2000
```

No longer `uid=0` — confirmed running as an unprivileged user.

### 8.3 Prove `readOnlyRootFilesystem` is actually enforced

```bash
kubectl exec secure-pod -- sh -c "echo test > /etc/hacked.txt"
```

**Expected output:**
```
sh: can't create /etc/hacked.txt: Read-only file system
```

But `/tmp` still works, since it's a real, explicitly writable volume:
```bash
kubectl exec secure-pod -- sh -c "echo test > /tmp/ok.txt && cat /tmp/ok.txt"
```

**Expected output:**
```
test
```

**Cleanup:**
```bash
kubectl delete pod insecure-pod secure-pod
```

---

## 9. Pod Security Standards

**Runs on:** Your local Kind cluster. Fully hands-on.

### 9.1 Create a namespace and enforce the `restricted` standard

```bash
kubectl create namespace secure-zone
kubectl label namespace secure-zone \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.31
```

**Expected output:**
```
namespace/secure-zone created
namespace/secure-zone labeled
```

### 9.2 Try to deploy the insecure Pod from Section 8.1 into this namespace

```bash
kubectl apply -f insecure-pod.yaml -n secure-zone
```

**Expected output:**
```
Error from server (Forbidden): error when creating "insecure-pod.yaml": pods "insecure-pod" is forbidden: violates PodSecurity "restricted:v1.31": allowPrivilegeEscalation != false (container "app" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "app" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "app" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "app" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

**This is the key production behavior:** the API server itself rejected the Pod **before it was ever created** — this namespace-wide policy caught exactly the missing security settings you manually fixed in Section 8.2, without relying on every developer remembering to do so themselves.

### 9.3 Now deploy the hardened version — needs one more field to satisfy `restricted`

```bash
cat >> secure-pod.yaml << 'EOF2'
EOF2
```
Add `seccompProfile` (the one requirement Section 8's example didn't include) to `secure-pod.yaml`'s Pod-level `securityContext`:
```yaml
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
```

```bash
kubectl apply -f secure-pod.yaml -n secure-zone
```

**Expected output:**
```
pod/secure-pod created
```

This confirms only the fully-hardened Pod is accepted in this namespace — exactly the enforcement guarantee Pod Security Standards exist to provide.

**Cleanup:**
```bash
kubectl delete namespace secure-zone
```

---

## 10. AWS EBS/EFS CSI Drivers + IRSA

**Runs on:** A real AWS EKS cluster only. **Cannot run on Kind** — there is no real AWS account/API behind a local cluster to provision actual EBS/EFS volumes or IAM roles against. Commands and output below are exactly what you'd run/see on real EKS infrastructure.

### 10.1 Set up IRSA — link a Kubernetes Service Account to a real IAM Role

```bash
eksctl create iamserviceaccount \
  --cluster=my-eks-cluster \
  --namespace=kube-system \
  --name=ebs-csi-controller-sa \
  --attach-policy-arn=arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```

**Expected output:**
```
[i]  1 iamserviceaccount (kube-system/ebs-csi-controller-sa) was included
[i]  created serviceaccount "kube-system/ebs-csi-controller-sa"
```

**Verify the linkage (this is the actual IRSA mechanism):**
```bash
kubectl get sa ebs-csi-controller-sa -n kube-system -o yaml | grep -A2 annotations
```

**Expected output:**
```yaml
annotations:
  eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/eksctl-my-eks-cluster-addon-iamserviceaccount-Role1-AbCdEfGh
```

Any Pod using this Service Account (`spec.serviceAccountName: ebs-csi-controller-sa`, exactly the field from Volume 5 Lab 2) automatically receives temporary AWS credentials scoped to exactly the `AmazonEBSCSIDriverPolicy` permissions — no Kubernetes Secret ever holds a long-lived AWS key.

### 10.2 Install the EBS CSI driver as an EKS addon

```bash
eksctl create addon --cluster my-eks-cluster \
  --name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::123456789012:role/eksctl-my-eks-cluster-addon-iamserviceaccount-Role1-AbCdEfGh
```

**Verify:**
```bash
kubectl get pods -n kube-system -l app=ebs-csi-controller
```

**Expected output:**
```
NAME                                  READY   STATUS    RESTARTS   AGE
ebs-csi-controller-7d9f8c-abcde       6/6     Running   0          40s
ebs-csi-controller-7d9f8c-fghij       6/6     Running   0          40s
```

### 10.3 Create a StorageClass using EBS (block storage, ReadWriteOnce)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: ebs-gp3 }
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
```

```bash
kubectl apply -f ebs-storageclass.yaml
kubectl get storageclass ebs-gp3
```

**Expected output:**
```
NAME      PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE
ebs-gp3   ebs.csi.aws.com   Delete          WaitForFirstConsumer
```

A PVC (Volume 4 Lab 5 concepts, applied to real cloud storage) requesting this class provisions a **real EBS volume in your AWS account** automatically:
```bash
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: ebs-test-claim }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: ebs-gp3
  resources: { requests: { storage: 10Gi } }
EOF
kubectl get pvc ebs-test-claim
```

**Expected output:**
```
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
ebs-test-claim    Bound    pvc-a1b2c3d4-e5f6-7890-abcd-ef1234567890    10Gi       RWO            ebs-gp3
```

### 10.4 EFS — shared, multi-node storage (ReadWriteMany)

EFS requires you to first create the actual EFS filesystem in AWS (`aws efs create-file-system`), then reference its ID:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: efs-sc }
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-0abc12345def67890
  directoryPerms: "700"
```

```bash
kubectl apply -f efs-storageclass.yaml
```

A PVC using `efs-sc` with `accessModes: [ReadWriteMany]` can be mounted by many Pods across many different nodes simultaneously — unlike the EBS example above, which only one node can attach at a time.

---

## 11. Real Cluster Installation with `kubeadm`

**Runs on:** Real (virtual or physical) Ubuntu machines only — **cannot run on Kind**, since Kind's entire purpose is simulating this exact process inside containers so you don't have to do it manually. The commands and output below are exactly what you'd type/see on real VMs — e.g., 3 Ubuntu 22.04 VMs: `control-1`, `worker-1`, `worker-2`.

### 11.1 On every machine: install the container runtime and kubeadm/kubelet/kubectl

```bash
# Run on control-1, worker-1, and worker-2
sudo apt-get update && sudo apt-get install -y containerd
sudo systemctl enable --now containerd

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet=1.31.0-1.1 kubeadm=1.31.0-1.1 kubectl=1.31.0-1.1
sudo apt-mark hold kubelet kubeadm kubectl
```

**Expected output (final line):**
```
kubelet set on hold.
kubeadm set on hold.
kubectl set on hold.
```

`apt-mark hold` prevents an accidental `apt upgrade` from silently changing your Kubernetes version outside of a deliberate, controlled upgrade (Volume 5 Lab 7).

### 11.2 On `control-1` only: bootstrap the control plane

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version=v1.31.0
```

**Expected output (final section — save this, you need it):**
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, run:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.1.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:1a2b3c4d5e6f7890...
```

`--pod-network-cidr=10.244.0.0/16` reserves an IP range for Pod networking (Volume 3's territory) — this specific range is what the Flannel CNI plugin expects by default, which we install next.

### 11.3 On `control-1`: set up `kubectl` access and install a CNI plugin

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

**Expected output:**
```
namespace/kube-flannel created
serviceaccount/flannel created
...
daemonset.apps/kube-flannel-ds created
```

Without this step, `kubectl get nodes` would show `STATUS: NotReady` forever — exactly the CNI dependency Volume 1 Lab 5's architecture explanation covered conceptually.

### 11.4 On `worker-1` and `worker-2`: join the cluster

```bash
sudo kubeadm join 10.0.1.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:1a2b3c4d5e6f7890...
```

**Expected output:**
```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

### 11.5 Back on `control-1`: verify

```bash
kubectl get nodes
```

**Expected output (after ~30–60 seconds):**
```
NAME        STATUS   ROLES           AGE     VERSION
control-1   Ready    control-plane   4m      v1.31.0
worker-1    Ready    <none>          45s     v1.31.0
worker-2    Ready    <none>          40s     v1.31.0
```

You've now built, entirely by hand, exactly the kind of cluster `kind create cluster` simulated for you throughout Volumes 1–5.

---

## 12. Certificate Management

**Runs on:** A real `kubeadm` cluster (continuing directly from Section 11). Conceptually applicable to Kind too, though Kind typically regenerates certs on cluster recreation rather than needing manual renewal.

### 12.1 Check certificate expiration

```bash
sudo kubeadm certs check-expiration
```

**Expected output:**
```
CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY
admin.conf                 Jul 22, 2027 09:12 UTC   364d
apiserver                  Jul 22, 2027 09:12 UTC   364d
apiserver-etcd-client      Jul 22, 2027 09:12 UTC   364d
apiserver-kubelet-client   Jul 22, 2027 09:12 UTC   364d
etcd-healthcheck-client    Jul 22, 2027 09:12 UTC   364d
etcd-peer                  Jul 22, 2027 09:12 UTC   364d
etcd-server                Jul 22, 2027 09:12 UTC   364d
front-proxy-client         Jul 22, 2027 09:12 UTC   364d
```

**Column explanation:**
- `RESIDUAL TIME` — how much validity remains. In production, this command is run regularly (often via a scheduled monitoring check) specifically to catch anything approaching `30d` or less before it becomes an outage.

### 12.2 Renew all certificates

```bash
sudo kubeadm certs renew all
```

**Expected output:**
```
[renew] Reading configuration from the cluster...
certificate for serving the Kubernetes API renewed
certificate the apiserver uses to access etcd renewed
certificate for liveness probes to healthcheck etcd renewed
...
Done renewing certificates. You must restart the kube-apiserver, kube-controller-manager, kube-scheduler and etcd, so that they can use the new certificates.
```

### 12.3 Restart static Pods to pick up the new certificates

Since these components are **static Pods** (Section 13 below), you restart them by moving their manifest out and back into the watched directory:

```bash
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 5
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
kubectl get pods -n kube-system -l component=kube-apiserver
```

**Expected output:**
```
NAME                         READY   STATUS    RESTARTS   AGE
kube-apiserver-control-1     1/1     Running   0          12s
```

`AGE: 12s` confirms the kubelet noticed the manifest reappear and restarted the Pod immediately — exactly the static Pod mechanism explained next.

---

## 13. Static Pods

**Runs on:** Your local Kind cluster. Fully hands-on — and this actually works on Kind, since it's a kubelet-level mechanism, not a cloud dependency.

### 13.1 Get inside a node's filesystem

```bash
docker exec -it my-first-cluster-control-plane bash
```

You are now, literally, inside the container acting as your control-plane node.

### 13.2 Find the static Pod manifest directory

```bash
ls /etc/kubernetes/manifests/
```

**Expected output:**
```
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

These four files are **exactly** the four control-plane Pods you saw in `kubectl get pods -n kube-system` all the way back in Volume 1 Lab 6 — now you're looking at their actual source: plain files on disk, not objects created via `kubectl apply`.

```bash
cat /etc/kubernetes/manifests/kube-scheduler.yaml | head -15
```

**Expected output:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
```

It's an ordinary Pod manifest — same `apiVersion`, `kind`, `metadata`, `spec` structure from Volume 1 Lab 6 — just read directly by the kubelet instead of going through `kubectl apply`.

### 13.3 Create your own static Pod

```bash
cat > /etc/kubernetes/manifests/my-static-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: my-static-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.27
EOF
exit
```

### 13.4 Watch it appear — without ever running `kubectl apply`

```bash
kubectl get pods -n default
```

**Expected output (within ~10-20 seconds):**
```
NAME                                       READY   STATUS    RESTARTS   AGE
my-static-pod-my-first-cluster-control-plane   1/1   Running   0        15s
```

Notice the node name was automatically appended to the Pod name — this is how Kubernetes distinguishes **mirror Pods** (the API server's read-only reflection of a static Pod) from normal, scheduler-assigned Pods.

### 13.5 Prove `kubectl delete` cannot actually remove it

```bash
kubectl delete pod my-static-pod-my-first-cluster-control-plane
kubectl get pods -n default
```

**Expected output:**
```
pod "my-static-pod-my-first-cluster-control-plane" deleted

NAME                                       READY   STATUS    RESTARTS   AGE
my-static-pod-my-first-cluster-control-plane   1/1   Running   0        3s
```

It came right back, with a fresh `AGE` — `kubectl delete` removed the *mirror* object from the API server, but the kubelet immediately noticed the real source (the file still sitting in `/etc/kubernetes/manifests/`) and recreated it. **The only way to actually remove a static Pod is to delete or move the file itself**, exactly the mechanism you used to restart the API server safely in Section 12.3.

**Cleanup:**
```bash
docker exec my-first-cluster-control-plane rm /etc/kubernetes/manifests/my-static-pod.yaml
```

---

## Summary

Every gap topic from the previous companion document now has a real, runnable (or, where genuinely cloud-dependent, precisely production-accurate) walkthrough:

| Topic | Ran on |
|---|---|
| Helm | Kind (fully hands-on) |
| GitOps / Argo CD | Kind (fully hands-on) |
| Prometheus / Grafana / Metrics Server | Kind (fully hands-on) |
| Loki logging | Kind (fully hands-on) |
| HPA | Kind (fully hands-on) |
| Cluster Autoscaler / Karpenter | Real AWS EKS only |
| Pod Disruption Budgets | Kind (fully hands-on) |
| Security Contexts | Kind (fully hands-on) |
| Pod Security Standards | Kind (fully hands-on) |
| EBS/EFS CSI + IRSA | Real AWS EKS only |
| `kubeadm` cluster install | Real VMs only |
| Certificate management | Real `kubeadm` cluster (concept applies to Kind too) |
| Static Pods | Kind (fully hands-on) |

You've now not only learned every concept from the original gap list, but actually run (or precisely traced) the production commands and output for each one.
