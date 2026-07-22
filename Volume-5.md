# Kubernetes Training Manual

## Volume 5 — Administration & Troubleshooting

**A Self-Study Handbook for Beginners to Intermediate Learners**
---

### About This Volume

Volumes 1–4 covered compute, networking, and storage — everything an application needs to run. Volume 5 covers everything an **administrator** needs to keep a cluster secure, well-scheduled, upgraded, and recoverable: RBAC and Service Accounts for access control, node management and scheduling controls (taints, tolerations, affinity), cluster upgrades, etcd backup and restore, and systematic production troubleshooting. This is also the final volume of the core course — Lab 10 combines everything from all five volumes into a comprehensive final mini project.

**Prerequisites for this volume:** Completion of Volumes 1–4.

---

## Table of Contents — Volume 5

- Lab 1 — RBAC
- Lab 2 — Service Accounts
- Lab 3 — Node Management
- Lab 4 — Scheduling
- Lab 5 — Taints & Tolerations
- Lab 6 — Affinity & Anti-Affinity
- Lab 7 — Cluster Upgrades
- Lab 8 — etcd Backup & Restore
- Lab 9 — Troubleshooting Production Clusters
- Lab 10 — Final Mini Project
- Volume 5 Review, Exam, and Reference

---

# Lab 1 — RBAC

**Estimated Duration:** 60 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain the RBAC model: subjects, verbs, resources, and bindings.
2. Write Roles and RoleBindings for namespace-scoped permissions.
3. Write ClusterRoles and ClusterRoleBindings for cluster-wide permissions.
4. Test and verify permissions using `kubectl auth can-i`.

### Prerequisites
- Completion of Volume 4.

### Theory

**Role-Based Access Control (RBAC)** governs *who* can do *what* to *which resources* in a Kubernetes cluster. Every API request — whether from a human via `kubectl`, or from a Pod's own Service Account (Lab 2) — is checked against RBAC rules before being allowed. Understanding RBAC properly finally explains something referenced but deferred throughout this manual: exactly what "protects" a Secret (Volume 4 Lab 2) beyond base64 encoding.

Four objects make up the RBAC model. A **Role** defines a set of permissions (which `verbs` — get, list, create, update, delete, watch — are allowed on which `resources`) scoped to a single namespace. A **ClusterRole** is the same idea but cluster-scoped, either for cluster-wide resources (like Nodes, from Volume 1) or to define a reusable permission set that can be bound at the namespace level too. A **RoleBinding** grants a Role's (or a ClusterRole's) permissions to specific subjects — users, groups, or Service Accounts — within one namespace. A **ClusterRoleBinding** grants a ClusterRole's permissions cluster-wide, across every namespace.

A critical distinction worth internalizing: a ClusterRole can be bound *either* cluster-wide (via a ClusterRoleBinding) *or* to just one namespace (via a RoleBinding referencing that ClusterRole) — this lets you define a reusable permission set once (e.g., "read-only access to Pods and Services") and grant it either broadly or narrowly as needed, without duplicating the underlying Role definition.

### Architecture Diagram

```
   Role: pod-reader (namespace: dev)          ClusterRole: node-viewer (cluster-wide resource)
   { verbs: [get,list], resources: [pods] }   { verbs: [get,list], resources: [nodes] }
        |                                            |
   RoleBinding: bind pod-reader              ClusterRoleBinding: bind node-viewer
   to User "alice" in namespace "dev"        to Group "sre-team" cluster-wide
        |                                            |
   alice can read Pods only in "dev"         sre-team can read Nodes (cluster-scoped) everywhere
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl apply -f pod-reader-role.yaml
kubectl apply -f pod-reader-binding.yaml
kubectl auth can-i get pods --as=alice -n dev
kubectl auth can-i delete pods --as=alice -n dev
```
**Purpose:** Creates a namespace-scoped Role and RoleBinding, then verifies exactly what the bound subject can and cannot do.
**Expected Output:** `yes` for the first command, `no` for the second — `alice` can read but not delete Pods in `dev`.
**Common mistakes:** Forgetting `-n <namespace>` when testing a namespace-scoped Role's permissions, since `can-i` defaults to your own current context's namespace.
**Real-world usage:** `kubectl auth can-i` is the standard, fast way to verify RBAC configuration without waiting for a real user to hit a permission error.

```bash
kubectl apply -f node-viewer-clusterrole.yaml
kubectl apply -f node-viewer-binding.yaml
kubectl auth can-i get nodes --as=bob
```
**Purpose:** Creates a ClusterRole and a ClusterRoleBinding, granting cluster-wide access to a cluster-scoped resource (revisiting Volume 1's namespaced-vs-cluster-scoped distinction).
**Expected Output:** `yes`
**Real-world usage:** Granting SRE/platform teams broad, cluster-wide read access without granting write/delete permissions everywhere.

```bash
kubectl get rolebindings,clusterrolebindings -A | grep alice
```
**Purpose:** Audits every binding granting permissions to a specific subject, across the entire cluster.
**Real-world usage:** Security review — confirming exactly what access a given user or Service Account actually has, cluster-wide.

### YAML Files

`pod-reader-role.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev                # Roles are always scoped to one namespace
rules:
  - apiGroups: [""]              # "" means the core API group (Pods, Services, etc.)
    resources: ["pods"]
    verbs: ["get", "list", "watch"]   # Read-only; no create/update/delete
```

`pod-reader-binding.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-pod-reader
  namespace: dev
subjects:
  - kind: User
    name: alice                  # Could also be "Group" or "ServiceAccount" (Lab 2)
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role                     # References the Role above (could also reference a ClusterRole)
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

`node-viewer-clusterrole.yaml` and `node-viewer-binding.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-viewer
rules:
  - apiGroups: [""]
    resources: ["nodes"]          # Nodes are cluster-scoped (Volume 1) - only a ClusterRole can grant access
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bob-node-viewer
subjects:
  - kind: User
    name: bob
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-viewer
  apiGroup: rbac.authorization.k8s.io
```

### Expected Output
```
$ kubectl auth can-i get pods --as=alice -n dev
yes
$ kubectl auth can-i delete pods --as=alice -n dev
no
```

### Validation Steps
1. `kubectl auth can-i get pods --as=alice -n dev` returns `yes`.
2. `kubectl auth can-i delete pods --as=alice -n dev` returns `no`.
3. `kubectl auth can-i get pods --as=alice -n prod` (a different namespace) also returns `no`, confirming the RoleBinding's namespace scope.

### Common Errors
- **"Error from server (Forbidden)"** — the exact, expected behavior when a subject lacks a needed permission; this is RBAC working correctly, not a bug.
- **RoleBinding referencing a nonexistent Role/ClusterRole** — silently grants no effective permissions; always double check `roleRef.name` matches an actual, existing Role/ClusterRole.
- **Confusing apiGroups** — forgetting that core resources (Pods, Services, ConfigMaps) use `""` (empty string) as their apiGroup, while others (Deployments) use `apps`, and others (Ingress) use `networking.k8s.io`.

### Troubleshooting
"Forbidden" errors are RBAC functioning as intended — the fix is to grant the specific, needed permission, not to treat it as a bug. For a RoleBinding with no effect, verify `roleRef` points to an actually-existing Role/ClusterRole with the exact correct name and kind. For apiGroup confusion, use `kubectl api-resources` (from Volume 1) to look up the correct `apiGroup` for any resource type you're writing rules for.

### Practice Tasks
1. Create a Role granting only `get`/`list` on ConfigMaps in one namespace, and verify with `can-i`.
2. Create a ClusterRole granting `get`/`list`/`watch` on Deployments across all namespaces (via a ClusterRoleBinding), and verify.
3. Bind the same ClusterRole to just one namespace using a RoleBinding instead, and confirm the access is now namespace-scoped rather than cluster-wide.
4. Audit all RoleBindings and ClusterRoleBindings for one specific subject using `kubectl get rolebindings,clusterrolebindings -A`.
5. Attempt an action you deliberately haven't granted permission for, and read the exact "Forbidden" error message closely.

### Challenge Lab
Design a complete RBAC scheme for a fictional three-person team: an "admin" with full access to their team's namespace, a "developer" with read/write on Deployments/Services/ConfigMaps but no Secret access, and a "viewer" with read-only access to everything in that namespace — write and apply all necessary Roles and RoleBindings, then verify every permission boundary with `can-i`.

### Best Practices
- Always follow the principle of least privilege: grant only the specific verbs/resources actually needed, never broad wildcard access as a shortcut.
- Prefer Roles/RoleBindings (namespace-scoped) over ClusterRoles/ClusterRoleBindings (cluster-wide) whenever access genuinely only needs to be namespace-scoped.
- Regularly audit bindings, especially ClusterRoleBindings, since they're easy to over-grant and easy to forget about.

### Interview Questions
1. **Q: What are the four core RBAC objects?** A: Role, ClusterRole, RoleBinding, ClusterRoleBinding.
2. **Q: What's the difference between a Role and a ClusterRole?** A: A Role is namespace-scoped; a ClusterRole is cluster-scoped (usable for cluster-wide resources or as a reusable permission set).
3. **Q: What's the difference between a RoleBinding and a ClusterRoleBinding?** A: A RoleBinding grants permissions within one namespace; a ClusterRoleBinding grants them cluster-wide.
4. **Q: Can a ClusterRole be bound to just one namespace?** A: Yes, via a RoleBinding that references it, rather than a ClusterRoleBinding.
5. **Q: How do you test what permissions a specific user has?** A: `kubectl auth can-i <verb> <resource> --as=<user> [-n <namespace>]`.
6. **Q: What apiGroup do core resources like Pods use?** A: `""` (empty string).
7. **Q: Why must Node permissions be granted via a ClusterRole, not a Role?** A: Nodes are cluster-scoped resources; namespace-scoped Roles cannot grant access to them.
8. **Q: What's the principle of least privilege in the context of RBAC?** A: Granting only the specific permissions actually needed, nothing broader.
9. **Q: What happens if a RoleBinding references a nonexistent Role?** A: It silently grants no effective permissions.
10. **Q: What kinds of subjects can a RoleBinding/ClusterRoleBinding target?** A: User, Group, or ServiceAccount.

### Quiz
1. Four core RBAC objects are: (a) Role, ClusterRole, RoleBinding, ClusterRoleBinding (b) Pod, Service, Deployment, Namespace (c) PV, PVC, StorageClass, CSI (d) User, Group, Team, Org — **a**
2. A Role is scoped to: (a) The whole cluster (b) One namespace (c) One node (d) One Pod — **b**
3. A ClusterRoleBinding grants permissions: (a) In one namespace only (b) Cluster-wide (c) Nowhere (d) Only to Nodes — **b**
4. Command to test permissions: (a) kubectl test-rbac (b) kubectl auth can-i (c) kubectl check-access (d) kubectl rbac test — **b**
5. Core resources (Pods, Services) use apiGroup: (a) apps (b) "" (empty string) (c) core.k8s.io (d) v1beta1 — **b**
6. Can a ClusterRole be bound to just one namespace? (a) No (b) Yes, via a RoleBinding — **b**
7. Node permissions require: (a) A Role (b) A ClusterRole (c) Neither (d) A ConfigMap — **b**
8. Principle of least privilege means: (a) Grant broad access for convenience (b) Grant only specifically needed permissions (c) Grant no access ever (d) Grant admin to everyone — **b**
9. A RoleBinding referencing a nonexistent Role: (a) Errors immediately (b) Silently grants no effective permissions (c) Grants full access (d) Is auto-corrected — **b**
10. Valid RoleBinding subject kinds: (a) User, Group, ServiceAccount (b) Only User (c) Only Pod (d) Only Node — **a**

### Homework
Design and implement a complete RBAC scheme for a fictional five-person platform team with three distinct roles (admin, developer, read-only auditor), write every Role/ClusterRole/RoleBinding/ClusterRoleBinding needed, and verify every intended permission boundary using `kubectl auth can-i`, documenting your full test matrix.

### Summary
RBAC governs every API request in a Kubernetes cluster via Roles/ClusterRoles (defining permissions) and RoleBindings/ClusterRoleBindings (granting them to subjects), finally explaining what truly protects sensitive resources like Secrets. Lab 2 covers Service Accounts — the specific subject type Pods themselves use to authenticate to the API server.

---

# Lab 2 — Service Accounts

**Estimated Duration:** 45 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain what a Service Account is and how it differs from a human User.
2. Understand the default Service Account every Pod automatically receives.
3. Create a custom Service Account and bind RBAC permissions to it.
4. Configure a Pod to use a specific, non-default Service Account.

### Prerequisites
- Completion of Lab 1.

### Theory

Every request to the Kubernetes API must be authenticated as some **subject** — Lab 1 covered Users and Groups (human identities, typically managed outside Kubernetes itself via a cloud provider's IAM or a corporate identity system). A **Service Account** is the equivalent identity for **non-human** callers — most importantly, for **Pods themselves** when they need to call the Kubernetes API (for example, a controller Pod that watches other resources, or an application that needs to read its own Pod's labels via the API rather than the Downward API).

Every namespace automatically has a `default` Service Account, and every Pod that doesn't explicitly specify one gets it automatically mounted (as a token, via a projected volume) at a well-known path inside the container. Critically, this default Service Account starts with **no RBAC permissions bound to it** — it can authenticate, but by default it's authorized to do essentially nothing, following the least-privilege principle from Lab 1.

For any Pod that genuinely needs to call the Kubernetes API, best practice is to create a **dedicated, purpose-specific Service Account**, bind only the exact RBAC permissions it needs (via a RoleBinding/ClusterRoleBinding referencing it as the subject, exactly as Lab 1 demonstrated with Users), and explicitly reference that Service Account in the Pod spec via `spec.serviceAccountName`. For Pods that don't need API access at all — the vast majority of ordinary application workloads — it's a further best practice to set `automountServiceAccountToken: false`, preventing even the unprivileged default token from being mounted unnecessarily, reducing the attack surface if that Pod is ever compromised.

### Architecture Diagram

```
   Namespace: dev
   +------------------------------------------------------+
   |  ServiceAccount: default        (auto-created, no RBAC bound) |
   |  ServiceAccount: pod-watcher     (custom, purpose-specific)    |
   +------------------------------------------------------+
                              |
              RoleBinding grants "pod-watcher" get/list/watch on Pods
                              |
   Pod (spec.serviceAccountName: pod-watcher)
        |
   Automatically gets a mounted token identifying it as "pod-watcher"
   when it calls the Kubernetes API from inside the container
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl get pod <any-existing-pod> -o jsonpath='{.spec.serviceAccountName}'
kubectl exec -it <any-existing-pod> -- ls /var/run/secrets/kubernetes.io/serviceaccount/
```
**Purpose:** Confirms every Pod, even without explicit configuration, has a Service Account and an automatically mounted token.
**Expected Output:** `default` for the first command; `ca.crt  namespace  token` for the second.
**Common mistakes:** Assuming a Pod with no explicitly-set `serviceAccountName` has no identity at all — it always defaults to `default`.
**Real-world usage:** Understanding this default mounting behavior is essential context for the `automountServiceAccountToken: false` best practice.

```bash
kubectl create serviceaccount pod-watcher
kubectl apply -f pod-watcher-binding.yaml
kubectl apply -f pod-watcher-deployment.yaml
kubectl exec -it <pod-watcher-pod> -- sh -c \
  'TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token); \
   curl -s --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
   -H "Authorization: Bearer $TOKEN" \
   https://kubernetes.default.svc/api/v1/namespaces/dev/pods'
```
**Purpose:** Demonstrates a Pod actually authenticating to the Kubernetes API using its own mounted Service Account token, exercising the specific RBAC permissions granted to `pod-watcher`.
**Real-world usage:** Exactly how in-cluster controllers, operators, and monitoring tools (which are themselves just Pods) interact with the Kubernetes API.

### YAML Files

`pod-watcher-binding.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-watcher-binding
  namespace: dev
subjects:
  - kind: ServiceAccount        # A Service Account subject, exactly like Lab 1's User subject
    name: pod-watcher
    namespace: dev                # Service Account subjects must specify their own namespace
roleRef:
  kind: Role
  name: pod-reader                # Reusing the Role defined in Lab 1
  apiGroup: rbac.authorization.k8s.io
```

`pod-watcher-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: pod-watcher-app, namespace: dev }
spec:
  replicas: 1
  selector: { matchLabels: { app: pod-watcher-app } }
  template:
    metadata: { labels: { app: pod-watcher-app } }
    spec:
      serviceAccountName: pod-watcher       # Explicitly use this custom Service Account, not "default"
      containers:
        - name: app
          image: curlimages/curl
          command: ["sh", "-c", "sleep 3600"]
```

Disabling token auto-mount for a Pod that doesn't need API access:
```yaml
spec:
  serviceAccountName: default
  automountServiceAccountToken: false   # No token mounted at all, reducing attack surface if compromised
```

### Expected Output
```
$ kubectl exec -it <pod-watcher-pod> -- sh -c '... curl ...'
{"kind":"PodList","apiVersion":"v1","items":[...]}    # Successful, authorized response
```

### Validation Steps
1. `kubectl get pod <pod> -o jsonpath='{.spec.serviceAccountName}'` shows `pod-watcher` for the custom Deployment.
2. The in-Pod `curl` call against the Kubernetes API using the mounted token succeeds and returns actual Pod data, proving both authentication (the token is valid) and authorization (RBAC permits the specific `get`/`list` action).

### Common Errors
- **"Forbidden" when calling the API from inside a Pod** — the Pod's Service Account lacks the necessary RBAC binding; revisit Lab 1's troubleshooting approach.
- **Assuming a Pod has no identity because `serviceAccountName` wasn't set** — it always defaults to `default`, which itself defaults to no permissions.
- **Accidentally granting excessive permissions to the `default` Service Account** — since every Pod in a namespace without explicit configuration shares it, over-granting `default` inadvertently affects every such Pod.

### Troubleshooting
For "Forbidden" errors from within a Pod, the fix is identical to Lab 1's approach — create or adjust a RoleBinding granting the Pod's specific Service Account the needed permissions. Never assume "no explicit Service Account" means "no identity" — always check the actual default. Avoid binding permissions directly to the `default` Service Account in any namespace with more than one workload; create purpose-specific Service Accounts instead.

### Practice Tasks
1. Confirm the `default` Service Account has no meaningful RBAC permissions bound to it in a fresh namespace.
2. Create a custom Service Account, bind it a narrow permission set, and use it in a Deployment.
3. Successfully call the Kubernetes API from inside a Pod using its own mounted token.
4. Set `automountServiceAccountToken: false` on a Pod that doesn't need API access, and confirm no token is mounted.
5. Create a second Service Account with a different, non-overlapping set of permissions, and compare their `can-i` results side by side.

### Challenge Lab
Build a minimal "Pod counter" application (using `curl` and the Kubernetes API directly, as demonstrated in this lab) that periodically reports the number of Pods in its own namespace, running as a Deployment using a dedicated, correctly-scoped Service Account — a simplified but realistic pattern mirroring how real Kubernetes controllers/operators work.

### Best Practices
- Never bind meaningful permissions to the `default` Service Account in a shared namespace; always create dedicated Service Accounts for anything needing API access.
- Set `automountServiceAccountToken: false` for the majority of ordinary application Pods that don't call the Kubernetes API at all.
- Apply the same least-privilege principle from Lab 1 to every Service Account — grant only the exact verbs/resources actually required.

### Interview Questions
1. **Q: What is a Service Account?** A: An identity for non-human callers, most importantly Pods, when they need to authenticate to the Kubernetes API.
2. **Q: What Service Account does a Pod get if none is explicitly specified?** A: The namespace's `default` Service Account.
3. **Q: Does the default Service Account have any RBAC permissions by default?** A: No, it can authenticate but is authorized to do nothing by default.
4. **Q: How does a Pod use a specific, custom Service Account?** A: Via `spec.serviceAccountName` in the Pod spec.
5. **Q: How do you grant RBAC permissions to a Service Account?** A: The same way as a User — via a RoleBinding/ClusterRoleBinding with `kind: ServiceAccount` as the subject.
6. **Q: Where is a Pod's Service Account token mounted inside the container?** A: `/var/run/secrets/kubernetes.io/serviceaccount/`.
7. **Q: What does `automountServiceAccountToken: false` do?** A: Prevents any Service Account token from being mounted into the Pod at all.
8. **Q: Why is it best practice to avoid binding permissions directly to `default`?** A: Every Pod in that namespace without explicit configuration shares it, so over-granting affects every such Pod.
9. **Q: Give a real-world example of something that needs a dedicated Service Account.** A: An in-cluster controller or operator that watches/manages other Kubernetes resources via the API.
10. **Q: What's mounted alongside the token that lets a Pod verify the API server's identity?** A: The cluster's CA certificate (`ca.crt`).

### Quiz
1. What identity does a Pod use to call the Kubernetes API? (a) A User (b) A Service Account (c) A Group (d) None needed — **b**
2. Default Service Account name: (a) system (b) default (c) admin (d) root — **b**
3. Does the default Service Account have RBAC permissions out of the box? (a) Yes, full access (b) No, none by default — **b**
4. Field to specify a custom Service Account on a Pod: (a) spec.serviceAccount (deprecated alias) / spec.serviceAccountName (b) metadata.serviceAccount (c) spec.identity (d) metadata.account — **a**
5. How to grant a Service Account RBAC permissions: (a) Cannot be done (b) RoleBinding/ClusterRoleBinding with kind: ServiceAccount (c) Only via ClusterRoleBinding (d) Automatic — **b**
6. Token mount path inside a container: (a) /etc/k8s/token (b) /var/run/secrets/kubernetes.io/serviceaccount/ (c) /root/.kube (d) /tmp/token — **b**
7. automountServiceAccountToken: false does: (a) Nothing (b) Prevents any token from being mounted (c) Grants more permissions (d) Deletes the Service Account — **b**
8. Best practice for the default Service Account: (a) Bind broad permissions to it (b) Avoid binding permissions directly to it (c) Delete it always (d) Use it for everything — **b**
9. A controller Pod watching other resources needs: (a) No identity (b) A dedicated Service Account with appropriate RBAC (c) Only a ConfigMap (d) A Secret only — **b**
10. What verifies the API server's identity to an in-Pod caller? (a) The token alone (b) The mounted ca.crt (c) DNS (d) Nothing needed — **b**

### Homework
Build a small in-cluster tool (using curl and a mounted Service Account token) that lists Services in its own namespace, create a dedicated Service Account and exactly the RBAC permissions it needs, and verify the entire flow works end-to-end, documenting each step.

### Summary
Service Accounts are the Pod-facing equivalent of the human User subjects from Lab 1, always present (defaulting to `default` with no permissions) and best used via dedicated, narrowly-scoped custom accounts for anything genuinely needing API access. Lab 3 shifts from access control to node management — inspecting, cordoning, and draining nodes.

---

# Lab 3 — Node Management

**Estimated Duration:** 45 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Inspect detailed node status, capacity, and conditions.
2. Cordon a node to prevent new Pod scheduling.
3. Drain a node safely for maintenance.
4. Understand node capacity versus allocatable resources.

### Prerequisites
- Completion of Lab 2.

### Theory

Nodes were introduced architecturally in Volume 1, but real cluster administration requires actively managing them — particularly during maintenance (OS patching, hardware replacement, planned decommissioning). Kubernetes provides two key operations for this: **cordoning** and **draining**.

**Cordoning** a node (`kubectl cordon`) marks it `SchedulingDisabled` — the scheduler will no longer place any *new* Pods on it, but existing Pods already running there are left completely untouched and continue running normally. This is the first, safe step before any maintenance — it stops the situation from getting "more full" while you prepare.

**Draining** a node (`kubectl drain`) goes further: it cordons the node (if not already) **and** evicts every existing Pod from it, respecting each Pod's PodDisruptionBudget if one exists (a safety mechanism, not covered in depth in this manual, that limits how many Pods of a given application can be down simultaneously) and each Pod's graceful termination period (Volume 1 Lab 7). Pods managed by controllers (Deployments, ReplicaSets, DaemonSets) are automatically recreated elsewhere by their controller; truly standalone Pods (rare, and generally discouraged per Volume 1's guidance) are simply deleted with no automatic replacement, which is why `drain` requires an explicit `--force` flag to proceed past them.

Every node reports both **capacity** (the node's total physical resources) and **allocatable** (capacity minus resources reserved for the OS and Kubernetes system components themselves, like kubelet and the container runtime) — the scheduler makes all its decisions based on **allocatable**, not raw capacity, which is why a node can appear to have "room" by capacity numbers alone but still be correctly treated as full by the scheduler.

### Architecture Diagram

```
   Node: worker-1 (before maintenance)
        |
   kubectl cordon worker-1  -->  SchedulingDisabled (existing Pods untouched, no NEW Pods placed)
        |
   kubectl drain worker-1   -->  Cordons (if not already) + evicts existing Pods
        |
   Deployment/DaemonSet-managed Pods --> automatically recreated on OTHER nodes
   Standalone Pods --> deleted, NOT automatically replaced (drain requires --force for these)
        |
   Node is now safe for maintenance (patching, reboot, decommission)
        |
   kubectl uncordon worker-1  -->  SchedulingDisabled removed, node accepts new Pods again
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl describe node <node-name>
```
**Purpose:** Revisits Volume 1's node inspection, now focused on the `Capacity`, `Allocatable`, and `Conditions` sections specifically.
**Expected Output:** Sections showing e.g. `Capacity: cpu: 4, memory: 8144600Ki` versus a slightly lower `Allocatable` value, plus `Conditions` like `Ready: True`, `MemoryPressure: False`.
**Real-world usage:** First command run when investigating why a Pod won't schedule onto an apparently-available node.

```bash
kubectl cordon <node-name>
kubectl get nodes
```
**Purpose:** Marks a node unschedulable for new Pods while leaving existing workloads untouched.
**Expected Output:** The node's `STATUS` column now shows `Ready,SchedulingDisabled`.
**Common mistakes:** Assuming `cordon` evicts existing Pods — it does not; that's specifically what `drain` does.
**Real-world usage:** First, safe step before any planned node maintenance.

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```
**Purpose:** Safely evicts all evictable Pods from a node in preparation for maintenance or decommissioning.
**Syntax:** `--ignore-daemonsets` is required since DaemonSet Pods (Volume 2 Lab 7) are, by design, meant to run on every node and would otherwise block the drain; `--delete-emptydir-data` acknowledges that any `emptyDir`-based data (Volume 4 Lab 3) on this node's Pods will be lost.
**Common mistakes:** Omitting `--ignore-daemonsets` on a cluster with DaemonSets present, causing the drain to fail/hang unexpectedly.
**Real-world usage:** Standard pre-maintenance step in any real production cluster.

```bash
kubectl uncordon <node-name>
```
**Purpose:** Re-enables scheduling on a node after maintenance is complete.

### YAML Files
Node management in this lab is performed via `kubectl` commands rather than YAML manifests, since Nodes themselves are typically registered automatically by the kubelet rather than hand-authored. No new manifests are introduced in this lab.

### Expected Output
```
$ kubectl get nodes
NAME       STATUS                     ROLES    AGE   VERSION
worker-1   Ready,SchedulingDisabled   <none>   2d    v1.29.0
worker-2   Ready                      <none>   2d    v1.29.0
```

### Validation Steps
1. After `cordon`, the node shows `SchedulingDisabled`, and existing Pods on it remain `Running` unaffected.
2. After `drain`, `kubectl get pods -o wide --field-selector spec.nodeName=<node-name>` shows no controller-managed Pods remaining (only DaemonSet Pods, if `--ignore-daemonsets` was used).
3. After `uncordon`, new Pods can be scheduled onto the node again.

### Common Errors
- **`drain` hangs or fails due to DaemonSet Pods** — missing `--ignore-daemonsets`.
- **`drain` refuses to proceed due to standalone (non-controller-managed) Pods** — requires explicit `--force`, since those Pods have no automatic replacement.
- **`drain` blocked by a PodDisruptionBudget** — the operation respects PDBs by design, preventing an unsafe number of an application's Pods from being down simultaneously; wait, or address the underlying capacity/PDB configuration rather than forcing through it carelessly.

### Troubleshooting
For DaemonSet-related drain failures, add `--ignore-daemonsets`. For standalone Pod blocks, use `--force` only after confirming their loss is genuinely acceptable (per Volume 1's general guidance against relying on standalone Pods in the first place). For PodDisruptionBudget blocks, investigate whether the underlying application has enough healthy replicas elsewhere before proceeding — this is a genuine safety mechanism working as intended, not an obstacle to bypass reflexively.

### Practice Tasks
1. Inspect a node's Capacity versus Allocatable values and calculate the difference, reasoning about what's likely reserved for system overhead.
2. Cordon a node and confirm existing Pods remain untouched while new Pods avoid it.
3. Drain a node with DaemonSets present, correctly using `--ignore-daemonsets`.
4. Uncordon a node and confirm it becomes schedulable again.
5. Deploy a standalone Pod (no controller) and observe drain's behavior/requirement of `--force` when attempting to remove it.

### Challenge Lab
Simulate a full, realistic maintenance cycle on a multi-node Kind cluster: cordon a node, drain it fully, verify (via `kubectl get pods -o wide`) that all evictable workloads relocated successfully to remaining nodes, then uncordon and confirm the cluster returns to its original, fully-schedulable state.

### Best Practices
- Always cordon before draining (though `drain` does this automatically) as a mental model — never remove a node's workloads without first stopping new ones from landing there.
- Avoid deploying genuinely standalone Pods in any environment where node maintenance is a routine operational concern — use Deployments/DaemonSets/StatefulSets instead, exactly as Volume 1 and 2 recommended.
- Always uncordon a node promptly after maintenance completes — a forgotten, permanently-cordoned node silently wastes cluster capacity.

### Interview Questions
1. **Q: What does cordoning a node do?** A: Marks it unschedulable for new Pods, while leaving existing Pods on it completely untouched.
2. **Q: What does draining a node do?** A: Cordons it (if needed) and evicts all existing evictable Pods from it.
3. **Q: Why does draining require `--ignore-daemonsets` on most real clusters?** A: DaemonSet Pods are designed to run on every node and would otherwise block the drain operation.
4. **Q: What happens to Deployment-managed Pods when their node is drained?** A: They're automatically recreated on other, schedulable nodes by their controller.
5. **Q: What happens to standalone Pods (no controller) when their node is drained?** A: They're simply deleted with no automatic replacement, requiring `--force` to proceed.
6. **Q: What's the difference between a node's Capacity and Allocatable resources?** A: Capacity is total physical resources; Allocatable is capacity minus resources reserved for the OS and Kubernetes system components.
7. **Q: Which value does the scheduler actually use for placement decisions?** A: Allocatable, not raw Capacity.
8. **Q: What safety mechanism can block a drain operation from proceeding too aggressively?** A: A PodDisruptionBudget.
9. **Q: How do you make a cordoned node schedulable again?** A: `kubectl uncordon <node-name>`.
10. **Q: Why should genuinely standalone Pods be avoided in production?** A: They have no automatic replacement mechanism, including during routine node maintenance/draining.

### Quiz
1. Cordoning a node: (a) Evicts all Pods (b) Prevents new Pod scheduling only (c) Deletes the node (d) Reboots the node — **b**
2. Draining a node: (a) Only cordons (b) Cordons and evicts existing Pods (c) Does nothing (d) Deletes the node object — **b**
3. Flag needed to drain a node with DaemonSets: (a) --force (b) --ignore-daemonsets (c) --skip-ds (d) --no-daemon — **b**
4. Deployment-managed Pods on a drained node are: (a) Lost forever (b) Automatically recreated elsewhere (c) Manually recreated only (d) Left running on the drained node — **b**
5. Standalone Pods on a drained node: (a) Auto-replaced (b) Deleted with no automatic replacement (c) Never affected (d) Cause the drain to always fail silently — **b**
6. Scheduler decisions are based on: (a) Capacity (b) Allocatable (c) Neither (d) Node age — **b**
7. What can block a drain from proceeding? (a) Nothing (b) A PodDisruptionBudget (c) Node age (d) CPU architecture — **b**
8. Command to make a node schedulable again: (a) kubectl schedule (b) kubectl uncordon (c) kubectl enable (d) kubectl resume — **b**
9. Why avoid standalone Pods in production? (a) No reason (b) No automatic replacement during maintenance/failures (c) They're faster (d) They're required — **b**
10. Allocatable is: (a) Total physical resources (b) Capacity minus system-reserved resources (c) Always equal to Capacity (d) Irrelevant to scheduling — **b**

### Homework
On a multi-node local cluster, deploy a mix of Deployment-managed and one standalone Pod, then perform a full cordon-drain-uncordon cycle on one node, documenting exactly what happened to each type of Pod and why, referencing the concepts from this lab.

### Summary
Cordoning and draining provide the safe, standard workflow for taking a node out of service for maintenance without disrupting controller-managed workloads, while node Capacity versus Allocatable explains what the scheduler actually sees. Lab 4 goes deeper into scheduling itself — how Kubernetes decides which node a Pod lands on in the first place.

---

# Lab 4 — Scheduling

**Estimated Duration:** 45 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain the two-phase scheduling process: filtering and scoring.
2. Use `nodeSelector` to constrain scheduling to labeled nodes.
3. Manually schedule a Pod by directly specifying `nodeName`.
4. Diagnose common Pod scheduling failures.

### Prerequisites
- Completion of Lab 3.

### Theory

Volume 1 introduced `kube-scheduler` as the component that assigns unscheduled Pods to nodes, but didn't explain exactly how it decides. The scheduler works in two phases for every Pod needing placement. **Filtering** eliminates every node that cannot possibly run the Pod — insufficient allocatable resources (Lab 3), a `nodeSelector`/affinity rule (this lab and Lab 6) that doesn't match, a taint the Pod doesn't tolerate (Lab 5), or a port conflict. **Scoring** then ranks the remaining, viable nodes using a set of built-in priority functions (e.g., preferring nodes with more free resources, spreading Pods across nodes for the same Deployment to avoid clustering) and picks the highest-scoring one.

The simplest way to constrain which nodes a Pod can land on is `nodeSelector` — a straightforward, exact-match label requirement (you already used this in Volume 2 Lab 7 for DaemonSets) added directly to the Pod spec: the Pod will only be scheduled onto nodes carrying every specified label. For more expressive matching (OR logic, "in"/"not in" sets, soft preferences rather than hard requirements), node affinity (Lab 6) extends well beyond what `nodeSelector` alone can express.

For debugging and rare manual-override scenarios, `spec.nodeName` bypasses the scheduler **entirely** — you directly name the exact node a Pod must run on. This skips filtering and scoring completely; if that node lacks capacity or the Pod doesn't actually fit, it simply fails rather than being retried elsewhere. This is a specialized, low-level mechanism, not a general scheduling tool — real workload placement control should almost always use `nodeSelector` or affinity instead.

### Architecture Diagram

```
   New, unscheduled Pod
        |
   PHASE 1: Filtering
        |
   Eliminate nodes lacking: allocatable resources, matching nodeSelector/affinity,
   tolerated taints, available ports
        |
   Remaining nodes: [node-2, node-4, node-5]
        |
   PHASE 2: Scoring
        |
   Rank by: free resources, Pod spreading, affinity preferences, etc.
        |
   Highest-scoring node selected: node-4
        |
   Pod's spec.nodeName is set to "node-4" (visible after scheduling, even though you didn't set it yourself)
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl label node <node-name> disktype=ssd
kubectl apply -f ssd-pod.yaml
kubectl get pod ssd-pod -o wide
```
**Purpose:** Labels a node, then constrains a Pod to only schedule onto nodes carrying that exact label.
**Expected Output:** The Pod lands specifically on the labeled node; `kubectl describe pod ssd-pod` Events show a successful `Scheduled` event referencing that node.
**Common mistakes:** Typos in the label key/value causing the Pod to stay `Pending` forever, with no node satisfying the (misspelled) requirement.
**Real-world usage:** Ensuring specific workloads (e.g., requiring fast local SSD storage) land only on appropriately-equipped nodes.

```bash
kubectl apply -f manual-schedule-pod.yaml
kubectl get pod manual-schedule-pod -o wide
```
**Purpose:** Demonstrates bypassing the scheduler entirely via `spec.nodeName`.
**Common mistakes:** Using this as a general scheduling mechanism rather than the narrow, specialized tool it is — it provides none of the scheduler's safety checks.
**Real-world usage:** Rare; mostly useful for very specific debugging scenarios or highly specialized infrastructure tooling.

```bash
kubectl describe pod <pending-pod-name>
```
**Purpose:** Revisits Volume 1's Pod troubleshooting, now specifically focused on reading the exact scheduling-failure reason in Events (e.g., `0/3 nodes are available: 3 Insufficient cpu`).
**Real-world usage:** The essential first diagnostic step for any Pod stuck `Pending`.

### YAML Files

`ssd-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata: { name: ssd-pod }
spec:
  nodeSelector:
    disktype: ssd            # Exact-match requirement; Pod only schedules onto nodes with this label
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
```

`manual-schedule-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata: { name: manual-schedule-pod }
spec:
  nodeName: my-cluster-worker     # Bypasses the scheduler entirely; Pod goes straight to this exact node
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
```

### Expected Output
```
$ kubectl get pod ssd-pod -o wide
NAME       READY   STATUS    NODE
ssd-pod    1/1     Running   my-cluster-worker    (the specific labeled node)
```

### Validation Steps
1. `ssd-pod` lands specifically on the node you labeled, confirmed via the `NODE` column of `kubectl get pod -o wide`.
2. `manual-schedule-pod` lands on the exact node named in `spec.nodeName`, with `kubectl describe pod` showing no normal `Scheduled` Event from `default-scheduler` (since the scheduler was bypassed).
3. A Pod requesting a nonexistent label combination stays `Pending`, with `describe` clearly explaining why.

### Common Errors
- **Pod stuck `Pending` with a `nodeSelector`** — no node carries the exact required label; check for typos or verify the label was actually applied.
- **`spec.nodeName` Pod stuck `Pending` or failing** — the named node doesn't exist, or genuinely lacks capacity; remember this bypasses the scheduler's normal safety filtering entirely.
- **Assuming scoring behaves identically across clusters** — built-in scoring priorities can be influenced by cluster-specific scheduler configuration; don't assume identical behavior everywhere.

### Troubleshooting
For `nodeSelector`-related `Pending` Pods, run `kubectl get nodes --show-labels` to confirm the exact label exists on at least one node, matching character-for-character. For `nodeName`-related failures, confirm the exact node name via `kubectl get nodes` and remember this mechanism intentionally skips all of the scheduler's normal resource/constraint checking. Always start troubleshooting scheduling failures with `kubectl describe pod`, exactly as established back in Volume 1.

### Practice Tasks
1. Label two nodes differently and create Pods with `nodeSelector` targeting each specifically.
2. Create a Pod with a `nodeSelector` value that matches no node, and read the exact `Pending` reason in Events.
3. Manually schedule a Pod via `spec.nodeName` and compare its Events output (no `Scheduled` event) against a normally-scheduled Pod's.
4. Explain, in writing, the two-phase filtering/scoring process in your own words, using a concrete example.
5. Research (via `kubectl explain pod.spec` or documentation) at least one built-in scheduler priority function not covered in this lab's theory section.

### Challenge Lab
Set up a 3-node Kind cluster, label one node `workload=batch` and another `workload=web`, then deploy two different Deployments each using `nodeSelector` to land exclusively on their intended node type, confirming correct placement for every resulting Pod.

### Best Practices
- Prefer `nodeSelector` (or affinity, Lab 6) over `spec.nodeName` for essentially all real scheduling constraints — it participates safely in the scheduler's normal resource-aware placement process.
- Reserve `spec.nodeName` for narrow debugging or highly specialized infrastructure scenarios, understanding it bypasses all safety checks.
- Always verify node labels are applied correctly (`--show-labels`) before assuming a `nodeSelector`-related scheduling failure is a Kubernetes bug rather than a labeling typo.

### Interview Questions
1. **Q: What are the two phases of Kubernetes scheduling?** A: Filtering (eliminating unsuitable nodes) and scoring (ranking remaining nodes to pick the best one).
2. **Q: What does `nodeSelector` do?** A: Constrains a Pod to only schedule onto nodes matching every specified label exactly.
3. **Q: What does `spec.nodeName` do?** A: Bypasses the scheduler entirely, directly assigning the Pod to a specific, named node.
4. **Q: What happens if a Pod's `nodeSelector` matches no node?** A: The Pod stays `Pending` indefinitely.
5. **Q: What's a key risk of using `spec.nodeName`?** A: It skips all of the scheduler's normal resource/constraint safety checks.
6. **Q: Name two things the filtering phase checks.** A: Allocatable resources and matching nodeSelector/affinity/taint-toleration requirements (any two of several valid answers).
7. **Q: What does the scoring phase do?** A: Ranks the remaining, filter-passing nodes to select the best one, based on priority functions like free resources and Pod spreading.
8. **Q: Where would you look to diagnose why a Pod is stuck `Pending` due to scheduling?** A: `kubectl describe pod`, checking the Events section.
9. **Q: Why is `nodeSelector` generally preferred over `nodeName` for real placement requirements?** A: It participates in normal, safe scheduler filtering/scoring rather than bypassing it entirely.
10. **Q: What's a real-world use case for `nodeSelector`?** A: Ensuring a workload requiring specific hardware (e.g., SSD storage, GPU) only lands on appropriately-equipped nodes.

### Quiz
1. Two scheduling phases are: (a) Filtering, scoring (b) Creating, deleting (c) Binding, unbinding (d) Starting, stopping — **a**
2. nodeSelector requires: (a) Fuzzy matching (b) Exact label matches (c) No matching needed (d) Only numeric labels — **b**
3. spec.nodeName: (a) Uses normal scheduling (b) Bypasses the scheduler entirely (c) Is the same as nodeSelector (d) Requires no node to exist — **b**
4. A Pod with an unmatched nodeSelector: (a) Schedules anyway (b) Stays Pending (c) Errors immediately (d) Auto-creates a matching node — **b**
5. Key risk of spec.nodeName: (a) None (b) Skips scheduler safety checks (c) Too slow (d) Requires more YAML — **b**
6. Filtering phase checks include: (a) Allocatable resources and label/taint matching (b) Only CPU (c) Only image name (d) Nothing — **a**
7. Scoring phase: (a) Eliminates nodes (b) Ranks remaining nodes to pick the best (c) Deletes Pods (d) Creates new nodes — **b**
8. Where to diagnose Pending scheduling issues: (a) kubectl logs (b) kubectl describe pod, Events section (c) kubectl top (d) kubectl config — **b**
9. Preferred mechanism for real placement requirements: (a) spec.nodeName (b) nodeSelector or affinity (c) Neither (d) Random assignment — **b**
10. Real-world nodeSelector use case: (a) None (b) Constraining hardware-specific workloads to appropriate nodes (c) Only for testing (d) Required on every Pod — **b**

### Homework
Set up a 3-node cluster with distinct labels representing different hardware profiles (e.g., `gpu=true`, `disktype=ssd`, no special label), deploy three different workloads each with an appropriate `nodeSelector`, and document the exact placement outcome and reasoning for each.

### Summary
Kubernetes scheduling works through filtering (eliminating unsuitable nodes) and scoring (ranking the rest), with `nodeSelector` providing simple exact-match placement constraints and `spec.nodeName` offering a narrow, scheduler-bypassing escape hatch. Lab 5 covers taints and tolerations — the mechanism that lets nodes actively repel Pods unless explicitly permitted.

---

# Lab 5 — Taints & Tolerations

**Estimated Duration:** 50 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain how taints and tolerations differ conceptually from `nodeSelector`.
2. Apply taints to nodes and observe Pods being repelled.
3. Add tolerations to Pods to permit scheduling onto tainted nodes.
4. Understand the three taint effects: NoSchedule, PreferNoSchedule, NoExecute.

### Prerequisites
- Completion of Lab 4.

### Theory

`nodeSelector` (Lab 4) works by **Pods opting in** to specific nodes. **Taints and tolerations** work the opposite way: a **taint** is applied to a **node**, actively repelling any Pod that doesn't explicitly declare it can tolerate that taint — this is a **node opting out** of accepting arbitrary Pods, unless those Pods specifically say otherwise. This is precisely the mechanism referenced back in Volume 2 Lab 7 explaining why ordinary Pods don't land on control-plane nodes by default.

A taint has three parts: a `key`, an optional `value`, and an `effect`. Three effects exist: `NoSchedule` (the scheduler will not place new Pods here unless they tolerate it; existing Pods already running are unaffected), `PreferNoSchedule` (a soft version — the scheduler tries to avoid this node but will use it if no better option exists), and `NoExecute` (the strongest — it not only prevents new scheduling but actively **evicts** any already-running Pods that don't tolerate it, optionally after a grace period via `tolerationSeconds`).

A **toleration** on a Pod doesn't force it *onto* a tainted node — it merely permits the Pod to land there if the scheduler's normal filtering/scoring (Lab 4) would otherwise choose to. To force placement onto specific nodes, you'd combine a toleration (permission) with a `nodeSelector` or affinity rule (Lab 6) targeting those specific nodes — taints/tolerations and node-targeting mechanisms solve two different, complementary problems: *repelling* unwanted Pods versus *attracting* specific ones.

### Architecture Diagram

```
   Node: gpu-node-1
   Taint: { key: "gpu", value: "true", effect: "NoSchedule" }
        |
   Pod A (no toleration)  --X-->  REPELLED, will not schedule here
   Pod B (tolerates "gpu"="true":NoSchedule)  ----->  ALLOWED to schedule here (not forced, just permitted)

   Control-plane node (typical default cluster setup):
   Taint: { key: "node-role.kubernetes.io/control-plane", effect: "NoSchedule" }
        |
   Ordinary Pods --X--> repelled by default
   System DaemonSet Pods (with matching toleration) --> allowed to run there
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl taint node <node-name> gpu=true:NoSchedule
kubectl apply -f no-toleration-pod.yaml
kubectl get pod no-toleration-pod -o wide
```
**Purpose:** Taints a node, then attempts to schedule an ordinary Pod without any matching toleration.
**Expected Output:** The Pod stays `Pending` (assuming no other, untainted node exists to absorb it), with Events showing the specific taint as the blocking reason.
**Common mistakes:** Confusing taint syntax — it's `key=value:effect`, easy to mistype the colon/equals placement.
**Real-world usage:** Reserving specific, specialized nodes (e.g., GPU-equipped) exclusively for workloads that explicitly request them.

```bash
kubectl apply -f gpu-workload-pod.yaml
kubectl get pod gpu-workload-pod -o wide
```
**Purpose:** Deploys a Pod with a matching toleration, confirming it's now permitted onto the tainted node.
**Expected Output:** The Pod successfully schedules, potentially landing on the tainted node (though not guaranteed unless also combined with a `nodeSelector`, per this lab's theory).

```bash
kubectl taint node <node-name> gpu=true:NoSchedule-
```
**Purpose:** Removes a taint (note the trailing dash, exactly like Volume 1 Lab 8's label-removal syntax).

### YAML Files

`no-toleration-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata: { name: no-toleration-pod }
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
  # No tolerations - will be repelled by the "gpu=true:NoSchedule" taint
```

`gpu-workload-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata: { name: gpu-workload-pod }
spec:
  tolerations:
    - key: "gpu"
      operator: "Equal"          # Or "Exists", to match any value for this key
      value: "true"
      effect: "NoSchedule"        # Must match the taint's effect exactly
  nodeSelector:
    gpu: "true"                    # Combined with the toleration to actually GUARANTEE landing on a GPU node
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
```

A `NoExecute` taint with a grace period (for a Pod that should stay briefly even after a taint is applied):
```yaml
tolerations:
  - key: "maintenance"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300         # Pod is evicted 5 minutes after the NoExecute taint is applied, not instantly
```

### Expected Output
```
$ kubectl get pod no-toleration-pod -o wide
NAME                  READY   STATUS    NODE
no-toleration-pod     0/1     Pending   <none>

$ kubectl get pod gpu-workload-pod -o wide
NAME                 READY   STATUS    NODE
gpu-workload-pod     1/1     Running   gpu-node-1
```

### Validation Steps
1. A Pod without a matching toleration stays `Pending` while a tainted node is the only schedulable candidate.
2. A Pod with a matching toleration (and, if combined, a matching `nodeSelector`) successfully schedules onto the tainted node.
3. Removing the taint (`taint-`) allows previously-blocked Pods to schedule normally.

### Common Errors
- **Taint syntax errors** — incorrect `key=value:effect` formatting; double-check each part carefully.
- **Toleration present but Pod still not landing on the intended tainted node** — a toleration only *permits*, it doesn't *force*; add a `nodeSelector`/affinity if guaranteed placement is required.
- **Forgetting `NoExecute` evicts already-running Pods, not just blocking new ones** — a meaningful difference from `NoSchedule`/`PreferNoSchedule`, worth confirming before applying in a live cluster.

### Troubleshooting
For syntax errors, re-verify the exact `key=value:effect` format matches Kubernetes' expected syntax precisely. For a toleration that doesn't achieve the intended guaranteed placement, remember to combine it with `nodeSelector`/affinity — tolerations alone only remove the *block*, they don't create an *attraction*. Before applying any `NoExecute` taint to a live, in-use node, understand it will actively evict non-tolerating Pods, not just prevent new ones.

### Practice Tasks
1. Taint a node with `NoSchedule` and confirm ordinary Pods are repelled while a tolerating Pod succeeds.
2. Taint a node with `PreferNoSchedule` and observe the softer, best-effort avoidance behavior (Pods land there only if no untainted alternative exists).
3. Apply a `NoExecute` taint to a node with existing Pods and observe eviction behavior, with and without `tolerationSeconds`.
4. Remove a taint and confirm previously-blocked Pods can now schedule normally.
5. Combine a toleration with a `nodeSelector` to achieve genuinely guaranteed placement onto a specific tainted node, contrasting this with a toleration used alone.

### Challenge Lab
Design a complete node-reservation scheme for a fictional cluster with three node pools — general-purpose (untainted), GPU-only (tainted `gpu=true:NoSchedule`), and a temporary maintenance pool (tainted `maintenance=true:NoExecute` with a 60-second `tolerationSeconds` grace period for critical Pods) — and verify the correct behavior for each pool with test Pods.

### Best Practices
- Use taints/tolerations to *reserve* nodes for specific workloads (repelling everything else), and combine with `nodeSelector`/affinity to *guarantee* the intended workload actually lands there.
- Be deliberate and cautious with `NoExecute` taints on nodes with live traffic — understand exactly which Pods will be evicted before applying.
- Document your cluster's taint scheme clearly, since it's easy for new team members to be confused by Pods mysteriously refusing to schedule onto specific nodes.

### Interview Questions
1. **Q: How do taints and tolerations differ conceptually from nodeSelector?** A: nodeSelector is a Pod opting in to specific nodes; taints/tolerations are a node opting out of accepting arbitrary Pods unless explicitly tolerated.
2. **Q: What are the three taint effects?** A: NoSchedule, PreferNoSchedule, NoExecute.
3. **Q: What does NoSchedule do?** A: Prevents new Pods from scheduling unless they tolerate it; existing Pods are unaffected.
4. **Q: What does NoExecute do differently from NoSchedule?** A: It also actively evicts already-running Pods that don't tolerate it, not just blocking new scheduling.
5. **Q: Does a toleration force a Pod onto a tainted node?** A: No, it only permits it; guaranteed placement requires combining it with nodeSelector/affinity.
6. **Q: What does `tolerationSeconds` control?** A: A grace period before a NoExecute-tainted node evicts a tolerating-but-time-limited Pod.
7. **Q: What real-world default taint explains why ordinary Pods don't land on control-plane nodes?** A: A control-plane taint (typically `node-role.kubernetes.io/control-plane:NoSchedule`).
8. **Q: What does PreferNoSchedule do?** A: A soft version of NoSchedule — the scheduler tries to avoid the node but will use it if no better option exists.
9. **Q: How do you remove a taint from a node?** A: `kubectl taint node <name> key=value:effect-` (trailing dash).
10. **Q: What's a real-world use case for taints?** A: Reserving specialized nodes (e.g., GPU-equipped) exclusively for workloads that explicitly tolerate and request them.

### Quiz
1. Taints are applied to: (a) Pods (b) Nodes (c) Namespaces (d) Services — **b**
2. Tolerations are applied to: (a) Nodes (b) Pods (c) Namespaces (d) Services — **b**
3. Three taint effects are: (a) NoSchedule, PreferNoSchedule, NoExecute (b) Allow, Deny, Maybe (c) Hard, Soft, Medium (d) Block, Permit, Force — **a**
4. NoExecute additionally: (a) Does nothing extra (b) Evicts already-running non-tolerating Pods (c) Only blocks new Pods (d) Deletes the node — **b**
5. Does a toleration force placement? (a) Yes (b) No, only permits it — **b**
6. tolerationSeconds controls: (a) Scheduling delay (b) Eviction grace period under NoExecute (c) Node capacity (d) Pod restart count — **b**
7. PreferNoSchedule is: (a) A hard block (b) A soft, best-effort avoidance (c) Identical to NoExecute (d) Ignored entirely — **b**
8. Control-plane nodes typically have: (a) No taints (b) A NoSchedule taint by default (c) Only PreferNoSchedule (d) A NoExecute taint always — **b**
9. To remove a taint: (a) kubectl untaint (b) kubectl taint node <name> key=value:effect- (c) kubectl delete taint (d) Not possible — **b**
10. Guaranteed placement onto a tainted node requires: (a) Toleration alone (b) Toleration plus nodeSelector/affinity (c) Nothing special (d) NoExecute only — **b**

### Homework
Design and implement a taint-based node reservation scheme on a multi-node local cluster with at least two distinct tainted node pools, deploy workloads with and without matching tolerations, and document the exact scheduling outcome for each combination tested.

### Summary
Taints let nodes actively repel Pods, with tolerations granting specific permission (not force) to land there anyway, across three effects of increasing strength (NoSchedule, PreferNoSchedule, NoExecute). Lab 6 covers affinity and anti-affinity — a more expressive mechanism for both attracting and spreading Pods based on rich matching rules.

---

# Lab 6 — Affinity & Anti-Affinity

**Estimated Duration:** 55 minutes
**Difficulty Level:** Intermediate–Advanced

### Learning Objectives
1. Explain node affinity as a more expressive alternative to nodeSelector.
2. Distinguish "required" versus "preferred" affinity rules.
3. Use Pod affinity and anti-affinity to control Pod placement relative to other Pods.
4. Design a realistic high-availability spreading strategy using anti-affinity.

### Prerequisites
- Completion of Lab 5.

### Theory

**Node affinity** extends Lab 4's simple `nodeSelector` with much richer expressiveness: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt` operators against node labels, plus a crucial distinction `nodeSelector` lacks entirely — **required** rules (`requiredDuringSchedulingIgnoredDuringExecution`, a hard constraint identical in strictness to `nodeSelector`) versus **preferred** rules (`preferredDuringSchedulingIgnoredDuringExecution`, a soft, weighted preference the scheduler tries to honor but won't block scheduling over if unsatisfiable). The verbose field names directly describe their behavior: rules apply *during scheduling*, but are *ignored during execution* — meaning if a node's labels change after a Pod is already running there, that Pod isn't retroactively evicted (a similar, deliberate philosophy to Lab 5's `NoSchedule` leaving existing Pods alone).

**Pod affinity** and **Pod anti-affinity** extend this same concept to reason about placement *relative to other Pods*, not just node labels — "schedule this Pod near Pods matching this selector" (affinity) or "schedule this Pod away from Pods matching this selector" (anti-affinity), using a `topologyKey` (commonly a node label like `kubernetes.io/hostname` for per-node granularity, or `topology.kubernetes.io/zone` for per-availability-zone granularity) to define what "near" or "away" actually means.

The single most common real-world use of Pod anti-affinity is **high-availability spreading**: ensuring a Deployment's replicas don't all land on the same node (or same availability zone), so that a single node/zone failure doesn't take down every replica simultaneously — directly strengthening the reliability guarantees a Deployment alone (Volume 2 Lab 4) doesn't automatically provide.

### Architecture Diagram

```
   Node Affinity (like nodeSelector, but richer):
   "Require: disktype In [ssd, nvme]"    "Prefer: zone = us-east-1a (weight: 80)"

   Pod Anti-Affinity for HA spreading:
   Deployment: web (3 replicas)
   Rule: "Don't co-locate with other app=web Pods" (topologyKey: kubernetes.io/hostname)
        |
   Node A: [web-pod-1]     Node B: [web-pod-2]     Node C: [web-pod-3]
   (scheduler actively spreads replicas across distinct nodes, not clustering them together)
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl label node <node-name> zone=us-east-1a
kubectl apply -f node-affinity-pod.yaml
kubectl get pod affinity-demo -o wide
```
**Purpose:** Demonstrates a required node affinity rule, functionally similar to `nodeSelector` but using the richer affinity syntax.
**Real-world usage:** Useful when you need `In`/`NotIn` set-based logic that plain `nodeSelector`'s exact-match-only model can't express.

```bash
kubectl apply -f web-deployment-antiaffinity.yaml
kubectl get pods -l app=web -o wide
```
**Purpose:** Deploys a 3-replica Deployment with Pod anti-affinity, confirming each replica lands on a distinct node.
**Expected Output:** Each of the three Pods shows a different value in the `NODE` column.
**Common mistakes:** Using `required` anti-affinity on a cluster with fewer nodes than replicas — the excess replicas will stay `Pending` forever, since the hard constraint can never be satisfied; `preferred` avoids this risk at the cost of a weaker guarantee.
**Real-world usage:** Standard high-availability configuration for any Deployment where losing an entire node shouldn't be able to take down every replica.

```bash
kubectl describe pod <pending-pod-if-any>
```
**Purpose:** Revisits Lab 4's scheduling diagnostics, now specifically watching for affinity-related "didn't match Pod's node affinity/selector" or "didn't satisfy existing pods anti-affinity rules" messages.

### YAML Files

`node-affinity-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata: { name: affinity-demo }
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:    # Hard requirement, like nodeSelector
        nodeSelectorTerms:
          - matchExpressions:
              - key: zone
                operator: In
                values: ["us-east-1a", "us-east-1b"]      # OR logic - either value satisfies this
      preferredDuringSchedulingIgnoredDuringExecution:    # Soft preference, doesn't block scheduling
        - weight: 80                                        # Higher weight = stronger preference among soft rules
          preference:
            matchExpressions:
              - { key: disktype, operator: In, values: ["ssd"] }
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
```

`web-deployment-antiaffinity.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: web }
spec:
  replicas: 3
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:   # Hard: NEVER co-locate two "app=web" Pods on one node
            - labelSelector:
                matchExpressions:
                  - { key: app, operator: In, values: ["web"] }
              topologyKey: kubernetes.io/hostname             # "Node" granularity; use a zone label for zone-level spreading
      containers:
        - name: nginx
          image: nginx:1.25
          resources:
            requests: { cpu: "100m", memory: "64Mi" }
            limits: { cpu: "250m", memory: "128Mi" }
```

### Expected Output
```
$ kubectl get pods -l app=web -o wide
NAME                  READY   STATUS    NODE
web-6d9f8c-abcde      1/1     Running   worker-1
web-6d9f8c-fghij      1/1     Running   worker-2
web-6d9f8c-klmno      1/1     Running   worker-3
```

### Validation Steps
1. `affinity-demo` lands only on a node matching the required zone rule.
2. Every Pod in the anti-affinity Deployment shows a distinct `NODE` value — no two `app=web` Pods share a node.
3. If you scale the anti-affinity Deployment beyond the available node count, excess replicas stay `Pending`, confirming the "required" constraint is genuinely hard.

### Common Errors
- **`required` anti-affinity causing Pods to stay `Pending`** — not enough distinct nodes exist to satisfy the hard spreading constraint; either add nodes, reduce replicas, or switch to `preferred`.
- **Confusing required versus preferred** — accidentally using `preferred` when a hard guarantee was actually needed, or vice versa, unnecessarily blocking scheduling.
- **Wrong `topologyKey`** — using `kubernetes.io/hostname` when zone-level (not just node-level) spreading was actually intended, or vice versa.

### Troubleshooting
For `Pending` Pods due to `required` anti-affinity, run `kubectl describe pod` to confirm this is indeed the blocking reason, then decide whether to add capacity, reduce replica count, or relax to `preferred`. Always deliberately choose `required` versus `preferred` based on whether the constraint is a genuine hard requirement or just a nice-to-have, and pick `topologyKey` based on the actual failure domain you're protecting against (single node versus entire availability zone).

### Practice Tasks
1. Create a Pod with a required node affinity rule using the `In` operator across two acceptable label values.
2. Add a preferred (soft) node affinity rule alongside a required one, and observe how it influences the scoring phase without blocking scheduling.
3. Deploy a 3-replica Deployment with required Pod anti-affinity on a 3-node cluster, confirming perfect spreading.
4. Scale that same Deployment to 5 replicas on the same 3-node cluster and observe the 2 excess replicas stay `Pending`.
5. Switch the anti-affinity rule to `preferred` and re-scale to 5 replicas, observing it now succeeds (with some inevitable co-location) rather than blocking.

### Challenge Lab
Design a complete high-availability scheme for a fictional 6-node, 3-availability-zone cluster: a Deployment with required Pod anti-affinity at the zone level (using an appropriate `topologyKey`) ensuring replicas spread across all three zones, tested by scaling and observing actual placement via `kubectl get pods -o wide` cross-referenced with each node's zone label.

### Best Practices
- Use `required` anti-affinity only when you have (or will always have) enough distinct nodes/zones to satisfy it; otherwise use `preferred` to avoid unexpectedly blocking scheduling.
- Choose `topologyKey` deliberately based on the actual failure domain you're protecting against — node-level spreading protects against single-node failures; zone-level spreading protects against entire zone outages.
- Combine node affinity's `preferred` rules with resource-aware scheduling to gently guide placement without creating hard failure points.

### Interview Questions
1. **Q: How does node affinity differ from nodeSelector?** A: It supports richer operators (In, NotIn, Exists, etc.) and distinguishes required (hard) from preferred (soft) rules.
2. **Q: What does `requiredDuringSchedulingIgnoredDuringExecution` mean?** A: A hard constraint enforced during scheduling, but not retroactively enforced if node labels change after the Pod is already running.
3. **Q: What's the difference between Pod affinity and Pod anti-affinity?** A: Affinity schedules Pods near matching Pods; anti-affinity schedules them away from matching Pods.
4. **Q: What does `topologyKey` define?** A: What "near" or "away" actually means — e.g., same node (`kubernetes.io/hostname`) or same zone.
5. **Q: What's the most common real-world use of Pod anti-affinity?** A: High-availability spreading of a Deployment's replicas across distinct nodes or zones.
6. **Q: What happens if required anti-affinity can't be satisfied?** A: Excess Pods stay Pending indefinitely.
7. **Q: What happens if preferred anti-affinity can't be satisfied?** A: The Pod still schedules, just without honoring the soft preference — no blocking occurs.
8. **Q: What weight field is used for among preferred rules?** A: Assigning relative importance/strength among multiple soft preferences.
9. **Q: Why might you choose zone-level over node-level anti-affinity?** A: To protect against an entire availability zone outage, not just a single node failure.
10. **Q: Does affinity alone guarantee a Deployment survives a node failure?** A: It significantly improves the odds by preventing co-location, but true resilience also depends on sufficient replica count and healthy remaining capacity.

### Quiz
1. Node affinity compared to nodeSelector offers: (a) Less expressiveness (b) Richer operators and required/preferred distinction (c) No difference (d) Only preferred rules — **b**
2. requiredDuringSchedulingIgnoredDuringExecution is: (a) A soft preference (b) A hard constraint (c) Ignored entirely (d) Only for Pods, not nodes — **b**
3. Pod anti-affinity schedules Pods: (a) Near matching Pods (b) Away from matching Pods (c) Randomly (d) Only on tainted nodes — **b**
4. topologyKey defines: (a) Node CPU (b) What "near/away" means (e.g., node or zone) (c) Pod name (d) Namespace — **b**
5. Most common real-world anti-affinity use: (a) Reducing costs (b) HA spreading of Deployment replicas (c) DNS resolution (d) Secret management — **b**
6. If required anti-affinity can't be satisfied: (a) Pods schedule anyway (b) Excess Pods stay Pending (c) Error immediately (d) Nodes are auto-added — **b**
7. If preferred anti-affinity can't be satisfied: (a) Pod stays Pending (b) Pod still schedules without the preference honored (c) Error (d) Nothing happens — **b**
8. weight is used for: (a) Node capacity (b) Relative strength among preferred rules (c) Required rules only (d) Namespace priority — **b**
9. Zone-level anti-affinity protects against: (a) Single node failure only (b) Entire availability zone outages (c) Nothing (d) DNS failures — **b**
10. Does affinity alone guarantee full HA? (a) Yes, completely (b) No, also depends on replica count and remaining capacity (c) Not related to HA (d) Only for StatefulSets — **b**

### Homework
Design a complete affinity/anti-affinity strategy for a fictional production application requiring: a database that must run on SSD-labeled nodes (required node affinity), a preference for a specific zone (preferred node affinity), and application replicas that must never co-locate on the same node (required Pod anti-affinity) — write and test the complete YAML on a local multi-node cluster.

### Summary
Node affinity extends nodeSelector with richer matching and required/preferred distinctions, while Pod affinity/anti-affinity control placement relative to other Pods — most commonly used for high-availability spreading. Lab 7 shifts to cluster upgrades, a critical operational skill for keeping a cluster's Kubernetes version current.

---

# Lab 7 — Cluster Upgrades

**Estimated Duration:** 55 minutes
**Difficulty Level:** Intermediate–Advanced

### Learning Objectives
1. Explain the Kubernetes version skew policy and why upgrade order matters.
2. Walk through a control-plane and node upgrade using `kubeadm`.
3. Understand the role of draining (Lab 3) during a real upgrade.
4. Plan a safe, minimal-downtime upgrade strategy.

### Prerequisites
- Completion of Lab 6.

### Theory

Kubernetes follows a strict, published **version skew policy**: the control plane (`kube-apiserver` specifically) must always be upgraded **first**, and worker node components (`kubelet`, `kube-proxy`) may lag behind the control plane by at most a few minor versions (the exact number is defined in official Kubernetes documentation and has varied slightly across releases) — never the other way around. This ordering exists because newer control planes are built to understand older node components, but not vice versa; an old control plane talking to newer, unfamiliar node components would be far more likely to break in unpredictable ways.

For clusters built with `kubeadm` (a common, standard cluster-bootstrapping tool referenced but not covered in depth in this manual, as this course focuses on Kind/Minikube for hands-on labs), the standard upgrade sequence is: **first**, upgrade the `kubeadm` tool itself on the primary control-plane node, then run `kubeadm upgrade plan` (a dry-run showing what would change) followed by `kubeadm upgrade apply <version>`; **second**, upgrade `kubelet` and `kubectl` on that same control-plane node and restart the kubelet service; **third**, repeat a similar node-level process for every worker node, one at a time, **draining each node first** (directly applying Lab 3's cordon/drain workflow) so its workloads relocate safely before that specific node's components are upgraded, then uncordoning it afterward.

Managed cloud Kubernetes offerings (EKS, GKE, AKS) significantly simplify this entire process — the cloud provider handles control-plane upgrades entirely, and often provides automated, one-click (or one-API-call) node pool upgrades that handle the drain/upgrade/uncordon cycle automatically per node. Understanding the manual `kubeadm` process, though, is exactly what makes it possible to reason correctly about what a managed upgrade button is actually doing underneath, and remains essential for genuinely self-managed, on-premises clusters.

### Architecture Diagram

```
   Upgrade order (STRICT):

   1. Control plane FIRST (kube-apiserver, controller-manager, scheduler, kubeadm upgrade apply)
        |
   2. Control-plane node's own kubelet/kubectl
        |
   3. Worker nodes, ONE AT A TIME:
        cordon --> drain --> upgrade kubelet/kubeadm on that node --> uncordon --> next node
        |
   Never: upgrading worker kubelet versions AHEAD of the control plane
```

### Installation
This lab describes the real-world `kubeadm` upgrade workflow conceptually and via command reference, since Kind and Minikube manage their own Kubernetes version internally (typically via recreating the cluster with a different node image) rather than exposing a `kubeadm upgrade` workflow directly. For genuine hands-on practice with real version skew and `kubeadm`, a from-scratch VM-based cluster (outside this manual's Kind/Minikube scope) would be required.

### Hands-on Practice

```bash
kubectl version --short 2>/dev/null || kubectl version
kubectl get nodes -o custom-columns='NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion'
```
**Purpose:** Confirms the current control-plane (client/server) version and each individual node's kubelet version, revealing any existing version skew directly.
**Expected Output:** All nodes typically showing the same version on a freshly-created Kind/Minikube cluster; on a real, partially-upgraded cluster, you might see a legitimate mix within the supported skew range.
**Real-world usage:** First check before planning or during any upgrade, confirming exactly what versions are currently in play.

```bash
# Illustrative kubeadm commands (run on a real kubeadm-based cluster's control-plane node):
kubeadm upgrade plan
kubeadm upgrade apply v1.30.0
```
**Purpose:** `plan` shows a dry-run of exactly what would change without applying anything; `apply` performs the actual control-plane upgrade to the specified version.
**Common mistakes:** Skipping `upgrade plan` and jumping straight to `apply`, missing an opportunity to review exactly what's about to change.
**Real-world usage:** The standard, official `kubeadm` control-plane upgrade sequence.

```bash
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data
# (on that specific worker node, as illustrative commands): apt-get upgrade kubelet kubectl; systemctl restart kubelet
kubectl uncordon <worker-node>
```
**Purpose:** Demonstrates the drain-upgrade-uncordon cycle for a single worker node, directly reusing Lab 3's exact workflow as the safety mechanism during an upgrade.
**Real-world usage:** Repeated one node at a time across an entire worker fleet during a real cluster upgrade.

### YAML Files
Cluster upgrades are performed via `kubeadm` and package manager commands rather than Kubernetes YAML manifests — no new manifests are introduced in this lab.

### Expected Output
```
$ kubectl get nodes -o custom-columns='NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion'
NAME             VERSION
control-plane    v1.30.0
worker-1         v1.29.0    (mid-upgrade; still within supported skew)
worker-2         v1.30.0    (already upgraded)
```

### Validation Steps
1. `kubectl get nodes` shows `STATUS: Ready` for every node throughout the process (aside from the brief, expected `SchedulingDisabled` during each node's own drain).
2. Every node's reported kubelet version stays within the cluster's supported version skew relative to the (already-upgraded) control plane at every point during the process.
3. No workload downtime beyond what Lab 3's normal drain/reschedule cycle already causes for standalone (non-controller-managed) Pods.

### Common Errors
- **Attempting to upgrade worker kubelets ahead of the control plane** — violates the version skew policy and can cause unpredictable failures.
- **Skipping the drain step during a worker node upgrade** — risks disrupting running workloads unnecessarily, when Lab 3's safe drain workflow was specifically designed to prevent this.
- **Upgrading multiple worker nodes simultaneously without sufficient remaining capacity** — can leave too few healthy nodes to absorb evicted workloads, causing scheduling failures during the upgrade window.

### Troubleshooting
For version skew violations, always confirm the control plane's version first, then only proceed with worker upgrades to versions within its supported skew range. For skipped-drain risks, always integrate Lab 3's cordon/drain/uncordon cycle directly into every worker node upgrade step, never upgrading a node's kubelet while it still has active, non-drained workloads. For capacity issues during multi-node upgrades, upgrade nodes serially (one at a time) rather than in parallel, unless you have significant spare capacity to safely absorb several simultaneous drains.

### Practice Tasks
1. Check your own Kind/Minikube cluster's current Kubernetes version and research what the next minor version's release notes highlight.
2. Write out, step by step, the full manual `kubeadm`-based upgrade sequence for a 1-control-plane, 3-worker cluster, in the correct order.
3. Explain, in writing, exactly why the control plane must always be upgraded before worker nodes, referencing the version skew policy.
4. Research how one major managed cloud Kubernetes offering (EKS, GKE, or AKS) automates this process, and compare it to the manual `kubeadm` sequence from this lab.
5. Simulate (via Lab 3's cordon/drain commands, even without an actual version change) the worker-node portion of an upgrade cycle on your local cluster.

### Challenge Lab
Write a complete, step-by-step runbook (as if for an on-call engineer) for upgrading a fictional 1-control-plane, 5-worker production `kubeadm` cluster by exactly one minor version, including pre-upgrade checks, the exact command sequence for the control plane and each worker node, and rollback considerations if something goes wrong mid-upgrade.

### Best Practices
- Always upgrade the control plane first, and never let worker node versions get ahead of it, per Kubernetes' official version skew policy.
- Always drain a worker node (Lab 3) before upgrading its kubelet, never upgrading components under a node with active, non-relocated workloads.
- Upgrade worker nodes serially, one at a time, to maintain sufficient cluster capacity throughout the process; never take down more capacity simultaneously than the remaining cluster can safely absorb.
- Read release notes for every version jump — breaking changes and deprecations are common between Kubernetes minor versions.

### Interview Questions
1. **Q: What is the Kubernetes version skew policy?** A: The control plane must be upgraded first; worker node components may lag behind it by at most a limited, defined number of minor versions.
2. **Q: What is the correct upgrade order?** A: Control plane first, then worker nodes, one at a time.
3. **Q: What kubeadm command shows a dry-run of an upgrade before applying it?** A: `kubeadm upgrade plan`.
4. **Q: What kubeadm command actually performs the control-plane upgrade?** A: `kubeadm upgrade apply <version>`.
5. **Q: What Lab 3 concept is directly reused during every worker node upgrade?** A: Cordon and drain, before upgrading that node's kubelet, followed by uncordon.
6. **Q: Why should worker nodes be upgraded one at a time rather than all simultaneously?** A: To maintain sufficient remaining cluster capacity to safely absorb each node's evicted workloads.
7. **Q: What happens if worker kubelet versions get ahead of the control plane?** A: This violates the version skew policy and risks unpredictable failures, since newer node components aren't guaranteed compatible with an older control plane.
8. **Q: How do managed cloud Kubernetes offerings simplify this process?** A: They handle control-plane upgrades entirely and often automate the node pool drain/upgrade/uncordon cycle.
9. **Q: Why is it important to read release notes before a version upgrade?** A: To identify breaking changes and deprecations that could affect existing workloads.
10. **Q: Why does Kind/Minikube not expose a typical `kubeadm upgrade` workflow directly?** A: They manage their own Kubernetes version internally, often via recreating the cluster with a different node image, rather than an in-place kubeadm upgrade.

### Quiz
1. Correct upgrade order is: (a) Workers first, then control plane (b) Control plane first, then workers (c) Simultaneously always (d) Random order — **b**
2. kubeadm dry-run command: (a) kubeadm upgrade apply (b) kubeadm upgrade plan (c) kubeadm check (d) kubeadm dry-run — **b**
3. kubeadm actual upgrade command: (a) kubeadm upgrade plan (b) kubeadm upgrade apply <version> (c) kubeadm install (d) kubeadm update — **b**
4. Before upgrading a worker node's kubelet, you should: (a) Do nothing extra (b) Cordon and drain it first (c) Delete the node (d) Upgrade the control plane again — **b**
5. Workers should be upgraded: (a) All simultaneously (b) One at a time (c) Never (d) Before the control plane — **b**
6. Version skew violation risk: (a) None (b) Unpredictable failures from incompatible newer node components (c) Faster performance (d) Automatic rollback — **b**
7. Managed cloud offerings typically: (a) Require full manual kubeadm work (b) Automate control-plane and often node pool upgrades (c) Don't support upgrades (d) Require downtime always — **b**
8. Why read release notes before upgrading? (a) Not necessary (b) To identify breaking changes/deprecations (c) Only for major versions (d) They don't exist — **b**
9. Kind/Minikube typically handle version changes via: (a) kubeadm upgrade apply (b) Recreating the cluster with a different node image (c) Not supported at all (d) Automatic nightly upgrades — **b**
10. Why upgrade nodes serially, not all at once? (a) No reason (b) Maintain sufficient remaining capacity during the process (c) Required by kubeadm syntax (d) Faster overall — **b**

### Homework
Write a complete upgrade runbook and rollback plan for a fictional production kubeadm cluster, including pre-upgrade validation steps (checking version skew, reading release notes, confirming backup status — foreshadowing Lab 8), the full command sequence, and specific criteria for deciding whether to proceed, pause, or roll back at each stage.

### Summary
Cluster upgrades must strictly follow the control-plane-first version skew policy, with worker nodes upgraded one at a time using the cordon/drain/uncordon cycle from Lab 3 as the core safety mechanism. Lab 8 covers etcd backup and restore — the critical disaster-recovery skill that should always precede any risky cluster operation, including upgrades.

---

# Lab 8 — etcd Backup & Restore

**Estimated Duration:** 55 minutes
**Difficulty Level:** Advanced

### Learning Objectives
1. Explain why etcd backup is the single most critical disaster-recovery skill in Kubernetes administration.
2. Perform a snapshot backup of etcd using `etcdctl`.
3. Restore a cluster from an etcd snapshot.
4. Design a backup schedule and retention strategy.

### Prerequisites
- Completion of Lab 7.

### Theory

Volume 1's architecture lab first introduced etcd as the cluster's single source of truth — every Pod, Service, Secret, ConfigMap, and RBAC rule from every volume of this manual ultimately lives there. Losing etcd without a backup means losing your **entire** cluster's configuration simultaneously — not just one workload, but everything. This makes etcd backup unambiguously the single most important disaster-recovery practice in Kubernetes administration, and one that should always be performed **before** any risky operation, very much including the cluster upgrades from Lab 7.

Backups are taken using `etcdctl snapshot save`, producing a single point-in-time snapshot file capturing etcd's entire key-value store at that moment. This requires connecting directly to etcd with the correct TLS certificates (etcd, like the rest of the control plane, uses mutual TLS for security) and, critically, must be run **on** (or with direct network access to) a control-plane node where etcd is actually running — this is fundamentally a control-plane-level operation, distinct from every namespace/Pod-level backup concern you might otherwise consider.

Restoring uses `etcdctl snapshot restore`, which reconstructs a **new** etcd data directory from a snapshot file — critically, this does **not** modify a running etcd cluster in place; the typical disaster-recovery procedure is to stop the (broken/corrupted) etcd process, restore the snapshot into a fresh data directory, reconfigure etcd to use that restored directory, and restart it, effectively rolling the entire cluster's state back to exactly the moment the snapshot was taken — meaning any changes made after that snapshot are permanently lost, which is precisely why backup **frequency** and retention policy matter enormously in a real production strategy.

### Architecture Diagram

```
   Control-plane node
   +----------------------------------------------+
   |  etcd (holds ALL cluster state - every         |
   |  volume/lab of this manual lives here)          |
   +----------------------------------------------+
                |
     etcdctl snapshot save backup.db
                |
     backup.db  (a single point-in-time file;
                  store this OFF the node too - a local-disk-only
                  backup doesn't survive that node's own failure)
                |
     [DISASTER: etcd corrupted/lost]
                |
     etcdctl snapshot restore backup.db --data-dir=/new/etcd/data
                |
     Reconfigure etcd to use /new/etcd/data, restart
                |
     Cluster state rolled back to exactly the snapshot's moment in time
```

### Installation
`etcdctl` is typically already present on `kubeadm`-based control-plane nodes (often as a container, requiring `crictl`/`docker exec` into the etcd Pod/container, or as a standalone binary depending on the specific cluster setup). For a Kind cluster, you can access it via:
```bash
docker exec -it <kind-control-plane-container-name> sh
# etcdctl is typically available inside this container
```

### Hands-on Practice

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-backup.db
```
**Purpose:** Creates a full, point-in-time snapshot of etcd's entire key-value store.
**Syntax:** The three certificate/key flags are required, since etcd requires mutual TLS authentication; exact paths vary by cluster setup (these are `kubeadm`'s typical default paths).
**Expected Output:** `Snapshot saved at /tmp/etcd-backup.db`
**Common mistakes:** Forgetting to copy the resulting snapshot file **off** that specific node — a backup that only exists on the same node that might fail provides no real disaster-recovery protection.
**Real-world usage:** Should run on a regular, automated schedule (e.g., via a CronJob-adjacent external scheduling mechanism, since this operates below the Kubernetes API layer itself) in any real production cluster.

```bash
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db --write-out=table
```
**Purpose:** Verifies a snapshot file's integrity and shows basic metadata (revision, key count, size) without actually restoring it.
**Real-world usage:** Regularly testing that your backups are actually valid and restorable, not just silently corrupted or incomplete — an all-too-common real-world failure mode of untested backup processes.

```bash
ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db --data-dir=/var/lib/etcd-restored
```
**Purpose:** Reconstructs a new etcd data directory from the snapshot (illustrative; a full restore also requires stopping etcd, pointing its configuration at this new directory, and restarting it — full sequence varies by cluster setup).
**Common mistakes:** Expecting this command alone to restore a live, running cluster — it only creates the new data directory; integrating that into a running etcd instance requires additional, cluster-specific steps.
**Real-world usage:** The core recovery step during a genuine disaster-recovery incident, or when practicing/validating your recovery procedure (which should be done regularly, not just when needed).

### YAML Files
etcd backup/restore is performed via `etcdctl` command-line operations directly against the control plane, entirely below the Kubernetes API layer — no Kubernetes YAML manifests are involved in this lab.

### Expected Output
```
$ ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db --write-out=table
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| a1b2c3d4 |    12345 |        847 |     2.1 MB |
+----------+----------+------------+------------+
```

### Validation Steps
1. `snapshot save` completes successfully and produces a non-empty file.
2. `snapshot status` on that file succeeds and reports a plausible key count and revision, confirming the backup is structurally valid.
3. (In a genuine disaster-recovery drill) A `snapshot restore` followed by full etcd/control-plane reconfiguration results in a cluster reflecting exactly the state captured at snapshot time.

### Common Errors
- **TLS certificate errors during `snapshot save`** — incorrect cert/key/CA paths for your specific cluster's etcd configuration.
- **Backup file only stored locally on the same node** — provides no protection against that specific node's own failure; always copy backups to separate, durable storage.
- **Never having actually tested a restore** — a backup that's never been validated by an actual (practice) restore may turn out to be unusable exactly when you need it most.

### Troubleshooting
For TLS errors, verify the exact certificate paths for your specific cluster (they vary between `kubeadm`, managed offerings, and custom setups) using `kubectl describe pod -n kube-system etcd-<node-name>` to inspect the actual mounted certificate paths if uncertain. Always copy backup files to storage independent of the node they were taken from — local-disk-only backups are a common, serious real-world mistake. Schedule regular restore drills (in a non-production, disposable test cluster) specifically to validate your backup process actually works, not just that it runs without error.

### Practice Tasks
1. Take an etcd snapshot on your own cluster (via Kind's control-plane container, if using Kind) and verify it with `snapshot status`.
2. Copy that snapshot file to a location outside the node/container it was taken on.
3. Research and write out, step by step, the full restore procedure for your specific cluster type, including stopping etcd, restoring into a new directory, and reconfiguring/restarting it.
4. Design a backup schedule (e.g., hourly snapshots, daily off-site copies, 30-day retention) for a fictional production cluster, and justify your retention choices.
5. Explain, in writing, exactly what data would be lost if you restored from a backup taken 6 hours before a disaster occurred.

### Challenge Lab
On a disposable Kind cluster (never production), perform a full, real disaster-recovery drill: take a snapshot, deliberately create some new resources afterward (to simulate "lost" post-backup changes), then attempt a full restore procedure, and document precisely what state was recovered versus what was correctly lost (the post-snapshot resources), confirming your understanding of exactly what a restore does and doesn't preserve.

### Best Practices
- Take etcd backups on a frequent, automated, regular schedule — never rely on remembering to do it manually before "risky" operations alone.
- Always store backups in a location independent of the node/cluster they were taken from — off-site, durable storage is essential for genuine disaster recovery.
- Regularly test actual restores in a disposable, non-production environment — an untested backup process is not a reliable one.
- Always take a fresh etcd backup immediately before any major, risky cluster operation, including the upgrades covered in Lab 7.

### Interview Questions
1. **Q: Why is etcd backup considered the single most critical Kubernetes disaster-recovery skill?** A: etcd holds the entire cluster's state; losing it without a backup means losing everything, not just one workload.
2. **Q: What command creates an etcd snapshot?** A: `etcdctl snapshot save <path>`.
3. **Q: What does `etcdctl snapshot status` do?** A: Verifies a snapshot file's integrity and shows metadata (revision, key count, size) without restoring it.
4. **Q: Does `etcdctl snapshot restore` modify a running etcd cluster in place?** A: No, it creates a new data directory that must then be integrated into a (typically stopped and reconfigured) etcd instance.
5. **Q: What happens to changes made after a snapshot was taken, once you restore from it?** A: They are permanently lost, since the restore rolls state back to exactly the snapshot's moment in time.
6. **Q: Why must etcd backups be stored off the node they were taken from?** A: A local-disk-only backup provides no protection if that specific node fails.
7. **Q: What authentication mechanism does connecting to etcd for backup/restore require?** A: Mutual TLS, using the correct CA certificate, client certificate, and key.
8. **Q: Why should backup restores be regularly tested, not just performed when a disaster occurs?** A: An untested backup process may turn out to be unusable exactly when it's actually needed.
9. **Q: When, relative to a cluster upgrade (Lab 7), should you take an etcd backup?** A: Immediately before, as a critical safety net in case the upgrade goes wrong.
10. **Q: What granularity of "undo" does an etcd restore provide?** A: A full, cluster-wide rollback to a single point in time — not selective, per-resource undo.

### Quiz
1. What does etcd hold? (a) Only Secrets (b) The entire cluster's state (c) Only Pod logs (d) Nothing important — **b**
2. Command to create an etcd snapshot: (a) etcdctl backup (b) etcdctl snapshot save <path> (c) kubectl backup etcd (d) kubectl snapshot — **b**
3. snapshot status does: (a) Restores the cluster (b) Verifies snapshot integrity/metadata (c) Deletes old snapshots (d) Upgrades etcd — **b**
4. Does restore modify a running etcd cluster in place? (a) Yes (b) No, creates a new data directory requiring further integration — **b**
5. Post-snapshot changes after a restore are: (a) Preserved (b) Permanently lost (c) Merged automatically (d) Backed up separately — **b**
6. Why store backups off-node? (a) No reason (b) Local-only backups don't survive that node's failure (c) Required by etcdctl (d) Faster access — **b**
7. etcd backup/restore requires: (a) No authentication (b) Mutual TLS with correct certs (c) Only a username/password (d) SSH keys only — **b**
8. Why regularly test restores? (a) Not necessary (b) Untested backups may be unusable when actually needed (c) Only for compliance (d) To save disk space — **b**
9. When should you back up etcd relative to an upgrade? (a) After only (b) Immediately before (c) Never needed (d) Only annually — **b**
10. etcd restore provides what granularity of rollback? (a) Per-resource selective undo (b) Full, cluster-wide point-in-time rollback (c) No rollback at all (d) Only for Secrets — **b**

### Homework
Design a complete, written backup and disaster-recovery policy document for a fictional production Kubernetes cluster: backup frequency, retention period, off-site storage location, restore-testing schedule, and the exact step-by-step procedure an on-call engineer would follow during an actual etcd-loss incident.

### Summary
etcd backup and restore is the ultimate safety net for a Kubernetes cluster's entire configuration state, requiring regular, automated, off-site-stored snapshots and — critically — regularly tested restore procedures. Lab 9 closes out the administrative skills with systematic production troubleshooting, tying together diagnostic techniques from every volume of this manual.

---

# Lab 9 — Troubleshooting Production Clusters

**Estimated Duration:** 60 minutes
**Difficulty Level:** Advanced

### Learning Objectives
1. Apply a systematic, layered troubleshooting methodology to any cluster issue.
2. Diagnose issues spanning multiple volumes' concepts simultaneously.
3. Use advanced diagnostic commands beyond the basics from earlier volumes.
4. Practice incident response under a realistic, multi-symptom scenario.

### Prerequisites
- Completion of Lab 8.

### Theory

This lab is a deliberate **synthesis** lab rather than one introducing brand-new objects — its purpose is to consolidate the diagnostic techniques scattered across all five volumes into one coherent, systematic methodology, since real production incidents rarely announce which volume's concept is actually at fault.

The recommended systematic approach, working from the outside in: **(1) Symptom** — what is actually failing, from the user's or monitoring system's perspective (a request times out, an error rate spikes)? **(2) Ingress/Networking layer** (Volume 3) — is traffic even reaching the cluster and being routed correctly (Ingress, Service Endpoints, DNS)? **(3) Workload layer** (Volumes 1–2) — are the relevant Pods actually `Running` and `Ready`, and has anything changed recently (a rollout, a scaling event)? **(4) Configuration/Storage layer** (Volume 4) — could a ConfigMap/Secret change or a storage issue (a full PVC, a failed mount) be the actual root cause? **(5) Cluster/Node layer** (this volume) — is a node under resource pressure, cordoned unexpectedly, or exhibiting a scheduling/RBAC/taint-related issue?

Critically, this ordering isn't rigid — an experienced engineer often jumps directly to the most likely layer based on the specific symptom and recent changes — but having this full mental checklist prevents the common trap of tunnel-visioning on one layer (e.g., endlessly re-checking application logs) while the actual root cause sits in a completely different layer (e.g., a NetworkPolicy silently blocking traffic, or a node running out of allocatable memory).

Beyond the individual commands already covered throughout this manual, a few additional diagnostic tools round out a production troubleshooter's toolkit: `kubectl get events --sort-by='.lastTimestamp' -A` (a cluster-wide, chronological event feed, invaluable for correlating *when* something changed against *when* symptoms began), `kubectl top nodes` / `kubectl top pods` (live resource usage, requiring the metrics-server addon from Volume 1 Lab 4), and `kubectl logs --previous` (Volume 1 Lab 7's crash-loop debugging technique, worth re-emphasizing as a frequently-forgotten essential).

### Architecture Diagram

```
   Systematic troubleshooting funnel (outside-in):

   1. SYMPTOM: "Users report intermittent 502 errors on checkout.example.com"
        |
   2. INGRESS/NETWORKING (Vol 3): Ingress routing OK? Service Endpoints populated? DNS resolving?
        |
   3. WORKLOAD (Vol 1-2): Pods Running/Ready? Recent rollout? CrashLoopBackOff anywhere?
        |
   4. CONFIG/STORAGE (Vol 4): Recent ConfigMap/Secret change? PVC full or unhealthy?
        |
   5. CLUSTER/NODE (Vol 5): Node resource pressure? Unexpected cordon? RBAC/taint issue?
        |
   ROOT CAUSE IDENTIFIED --> targeted fix --> confirm symptom resolved --> postmortem
```

### Installation
No new installation — this lab exercises diagnostic commands against whatever workloads/state already exist on your cluster from prior labs.

### Hands-on Practice

```bash
kubectl get events --sort-by='.lastTimestamp' -A | tail -30
```
**Purpose:** Shows the most recent cluster-wide events chronologically, immediately surfacing anything unusual that happened recently (failed schedules, image pull failures, evictions, node condition changes) across every namespace at once.
**Common mistakes:** Forgetting `-A` and only seeing events from the current namespace, missing a root cause that occurred elsewhere (e.g., in `kube-system`).
**Real-world usage:** Often the single fastest way to spot what actually changed right before symptoms began, cutting straight past manual guessing.

```bash
kubectl top nodes
kubectl top pods -A --sort-by=memory
```
**Purpose:** Shows live CPU/memory usage across nodes and Pods, immediately surfacing resource-pressure-related root causes.
**Common mistakes:** Forgetting this requires the metrics-server addon (Volume 1 Lab 4) to be installed and healthy — the command fails outright without it.
**Real-world usage:** Quickly confirming or ruling out "is something just resource-starved" as a hypothesis.

```bash
kubectl logs <pod-name> --previous --tail=100
kubectl describe pod <pod-name> | tail -20
```
**Purpose:** Revisits Volume 1 Lab 7's crash-loop debugging, retrieving logs from a container's last (crashed) run and the most recent Events, together forming the core of most application-level root cause investigations.

### YAML Files
This lab is diagnostic in nature; no new manifests are introduced. All commands are read-only investigative tools applied against existing cluster state.

### Expected Output
```
$ kubectl get events --sort-by='.lastTimestamp' -A | tail -10
NAMESPACE   LAST SEEN   TYPE      REASON             OBJECT                MESSAGE
prod        45s         Warning   FailedScheduling   pod/web-7d9f-abcde    0/3 nodes are available: 3 Insufficient memory
prod        30s         Normal    Killing            pod/web-6c8e-fghij    Stopping container nginx
kube-system 10s         Warning   NodeMemoryPressure  node/worker-2         Node worker-2 status is now: MemoryPressure
```

### Validation Steps
Given a deliberately-introduced problem (see Practice Tasks below), correctly identify the root cause using the layered methodology, and confirm the fix by observing the original symptom actually resolves — not just that "some error went away."

### Common Errors
- **Fixing a symptom instead of the root cause** — e.g., repeatedly restarting a Pod that keeps crashing due to a genuinely missing ConfigMap key, without ever fixing the underlying missing key itself.
- **Tunnel-visioning on one layer** — spending excessive time re-reading application logs when the actual root cause is a NetworkPolicy or node resource issue at a completely different layer.
- **Not correlating timing** — missing an obviously-relevant recent change (a deployment, a config edit) visible in `kubectl get events` because it wasn't specifically checked.

### Troubleshooting
This entire lab *is* the troubleshooting methodology — apply the five-layer funnel deliberately and explicitly for any nontrivial issue, rather than jumping randomly between commands. Always check `kubectl get events --sort-by='.lastTimestamp' -A` early in any investigation, since it frequently reveals the actual root cause (or a strong clue toward it) in seconds rather than minutes of manual layer-by-layer checking.

### Practice Tasks
1. Deliberately introduce a NetworkPolicy (Volume 3 Lab 8) blocking legitimate traffic, then practice diagnosing it purely from the "users report errors" symptom using the layered methodology.
2. Deliberately misconfigure a ConfigMap reference (Volume 4 Lab 1), causing `CreateContainerConfigError`, and diagnose it using `kubectl get events` as your first step.
3. Deliberately cordon a node (Lab 3) unexpectedly and observe how new Pod scheduling failures manifest and how you'd trace them back to the cordon.
4. Practice `kubectl top nodes`/`kubectl top pods` on your cluster (installing metrics-server if not already present) to build familiarity with baseline, healthy resource usage.
5. Write your own five-layer troubleshooting checklist from memory, without referring back to this lab's theory section, then compare it against the original.

### Challenge Lab
Design and execute a "chaos" exercise on a disposable test cluster: have someone else (or, working alone, script something to run later without you watching) introduce one deliberate, unannounced problem from anywhere across Volumes 1–5's concepts, then time yourself finding and fixing the actual root cause using strictly the systematic methodology from this lab, documenting exactly which layer the issue was in and how you confirmed it.

### Best Practices
- Always start any nontrivial investigation with `kubectl get events --sort-by='.lastTimestamp' -A` — it's fast and frequently points directly at the root cause.
- Resist the urge to fix symptoms (restarting Pods, scaling up) without first understanding and addressing the actual root cause — symptom-fixes without root-cause fixes just delay recurrence.
- Write a brief postmortem after any real incident, even a minor one — documenting root cause and resolution builds institutional knowledge and speeds up future diagnosis of similar issues.
- Maintain and practice this five-layer mental model regularly, even outside of actual incidents, so it's second nature during real, higher-pressure situations.

### Interview Questions
1. **Q: What is the recommended five-layer troubleshooting methodology from this lab?** A: Symptom, Ingress/Networking, Workload, Configuration/Storage, Cluster/Node — working from the outside in.
2. **Q: Why isn't this ordering meant to be followed rigidly?** A: An experienced engineer often jumps directly to the most likely layer based on the specific symptom and recent changes.
3. **Q: What command provides a fast, cluster-wide, chronological view of recent events?** A: `kubectl get events --sort-by='.lastTimestamp' -A`.
4. **Q: What's required for `kubectl top nodes`/`kubectl top pods` to work?** A: The metrics-server addon must be installed and healthy.
5. **Q: What Volume 1 command retrieves logs from a crashed container's previous run?** A: `kubectl logs --previous`.
6. **Q: What's the risk of "fixing" a symptom without addressing the root cause?** A: The underlying problem persists and will likely recur.
7. **Q: Why is tunnel-visioning on one layer a common troubleshooting mistake?** A: It can waste significant time while the actual root cause sits in a completely different layer.
8. **Q: Why is correlating timing (via events) important during an investigation?** A: A recent change (deployment, config edit) is often directly responsible for a newly-appearing symptom.
9. **Q: What's the value of writing a postmortem after an incident?** A: It builds institutional knowledge and speeds up diagnosis of similar future issues.
10. **Q: Give an example of a root cause that would only be found by checking the Cluster/Node layer.** A: A node under memory pressure, an unexpected cordon, or an RBAC/taint-related scheduling issue.

### Quiz
1. The five troubleshooting layers, outside-in, are: (a) Symptom, Ingress, Workload, Config/Storage, Cluster/Node (b) Random order (c) Only Pod logs (d) Only node checks — **a**
2. Is the layer order meant to be followed rigidly? (a) Yes, always (b) No, adapt based on symptom/context — **b**
3. Fast, cluster-wide chronological event command: (a) kubectl get pods -A (b) kubectl get events --sort-by='.lastTimestamp' -A (c) kubectl logs -A (d) kubectl top -A — **b**
4. kubectl top requires: (a) Nothing extra (b) The metrics-server addon (c) RBAC admin (d) A Secret — **b**
5. Command for a crashed container's previous logs: (a) kubectl logs (b) kubectl logs --previous (c) kubectl describe --previous (d) kubectl events --previous — **b**
6. Risk of fixing symptoms without root cause: (a) None (b) The problem persists and likely recurs (c) Faster resolution (d) No risk — **b**
7. Tunnel-visioning is risky because: (a) It's always correct (b) Root cause may be in a different layer entirely (c) It's the fastest method (d) No risk — **b**
8. Why check event timing during investigation? (a) Not useful (b) Recent changes often directly cause new symptoms (c) Only for compliance (d) Irrelevant to root cause — **b**
9. Value of a postmortem: (a) None (b) Builds institutional knowledge, speeds future diagnosis (c) Only for blame (d) Not recommended — **b**
10. A Cluster/Node-layer root cause example: (a) A missing ConfigMap key (b) Node memory pressure or unexpected cordon (c) A DNS typo (d) A wrong image tag — **b**

### Homework
Pick any three prior labs from Volumes 1–5, deliberately reintroduce (on a disposable test cluster) the specific failure each one covered, and practice diagnosing all three purely from their user-facing symptoms using this lab's five-layer methodology, writing a brief postmortem for each.

### Summary
Systematic, layered troubleshooting — moving from symptom through networking, workload, configuration/storage, and cluster/node layers — ties together every diagnostic technique from this entire manual into one coherent, repeatable methodology. Lab 10 closes the course with a comprehensive final mini project combining concepts from all five volumes into one complete, realistic system.

---

# Lab 10 — Final Mini Project

**Estimated Duration:** 120 minutes
**Difficulty Level:** Advanced (capstone for Volume 5 and the entire course)

### Learning Objectives
1. Design and deploy a complete, realistic, production-shaped application using concepts from all five volumes.
2. Apply security, reliability, and operational best practices learned throughout the course.
3. Practice systematic troubleshooting (Lab 9) against your own deliberately-introduced issue.
4. Consolidate the entire course into one working system you built yourself.

### Prerequisites
- Completion of every prior lab across all five volumes.

### Theory

This final project asks you to build a realistic three-tier e-commerce-style application — `frontend`, `api`, and `db` — using nearly every concept from this entire manual: Deployments with rolling updates and readiness probes (Volume 2), Services and an Ingress with TLS (Volume 3), ConfigMaps/Secrets and dynamically-provisioned storage with tuned resource limits (Volume 4), and RBAC, a dedicated Service Account, Pod anti-affinity for HA spreading, and appropriate node scheduling controls (Volume 5) — plus a NetworkPolicy enforcing least-privilege traffic (Volume 3 Lab 8), directly mirroring Volume 3's own mini project but now with the full depth of storage, configuration, and administrative controls layered on top.

Rather than providing every line of YAML pre-written (as earlier labs did), this capstone is intentionally **specification-driven**: you are given the requirements and must design and write the complete manifest set yourself, exercising genuine synthesis rather than transcription — exactly the skill a real production deployment demands.

### Architecture Diagram

```
                         Internet
                             |
                Ingress (TLS, host: shop.example.com)
                             |
                    Service: frontend-svc
                             |
        Deployment: frontend (2 replicas, anti-affinity spread)
                             |
                    Service: api-svc
                             |
   Deployment: api (2 replicas, anti-affinity, dedicated ServiceAccount + RBAC)
        - Consumes: api-config (ConfigMap), api-secret (Secret)
                             |
                    Service: db-svc (headless, if using StatefulSet)
                             |
        StatefulSet: db (1 replica, dynamically-provisioned PVC)

   NetworkPolicy: frontend->api allowed, api->db allowed, frontend->db BLOCKED
   RBAC: api's ServiceAccount can read its own namespace's ConfigMaps only
```

### Installation
No new installation beyond what earlier volumes already required (Ingress Controller from Volume 3 Lab 5, a policy-enforcing CNI from Volume 3 Lab 8, and a working default StorageClass from Volume 4 Lab 6 — all standard on Kind/Minikube or easily added).

### Hands-on Practice / Project Requirements

Design and implement a complete manifest set satisfying every requirement below. This lab intentionally does not provide the full YAML — apply everything you've learned to write it yourself, then validate against the Expected Output and Validation Steps sections.

**Requirements:**
1. **`frontend`**: A Deployment (2 replicas) serving simple content (e.g., `hashicorp/http-echo`), with a readiness probe, required Pod anti-affinity (never co-locate replicas on the same node), and a ClusterIP Service.
2. **`api`**: A Deployment (2 replicas) with a readiness probe and required Pod anti-affinity, consuming a ConfigMap (`api-config`, at least 2 keys) and a Secret (`api-secret`, at least 1 key) as environment variables, running under a **dedicated Service Account** (`api-sa`) bound to a Role granting only `get`/`list` on ConfigMaps in its own namespace (nothing more), with a ClusterIP Service.
3. **`db`**: A StatefulSet (1 replica) with a headless Service, using `volumeClaimTemplates` for dynamically-provisioned storage (at least 1Gi), with resource requests/limits set to achieve **Guaranteed** QoS.
4. **Ingress**: Routes `shop.example.com` to `frontend-svc`, with a `tls` block referencing a Secret (a self-signed certificate is fine for this exercise).
5. **NetworkPolicy**: `frontend` can reach `api`; `api` can reach `db`; `frontend` cannot reach `db` directly.
6. **Resource governance**: A ResourceQuota and LimitRange applied to the project's namespace.

```bash
kubectl apply -f final-project/    # applying your own complete manifest directory
kubectl get all,ingress,networkpolicy,pvc,sa,role,rolebinding -n final-project
```

### YAML Files
Intentionally not provided in full for this capstone — design and write your own manifests satisfying every requirement above, drawing on the specific YAML patterns demonstrated throughout every lab of all five volumes. As a structural checkpoint, your final directory should contain at minimum: a Namespace, two ConfigMaps/Secrets, three Deployments/StatefulSets, three Services (one headless), one Ingress plus its TLS Secret, two NetworkPolicies, one ServiceAccount with its Role/RoleBinding, one PersistentVolumeClaim (via `volumeClaimTemplates`), and one ResourceQuota/LimitRange pair.

### Expected Output
```
$ kubectl get deployments,statefulsets -n final-project
NAME                    READY   UP-TO-DATE   AVAILABLE
deployment.apps/api       2/2     2            2
deployment.apps/frontend  2/2     2            2
NAME                READY
statefulset.apps/db   1/1

$ curl -k -H "Host: shop.example.com" https://<ingress-address>/
hello from frontend
```

### Validation Steps
1. All Deployments/StatefulSet show full readiness.
2. `frontend` and `api` Pods are spread across distinct nodes (anti-affinity working).
3. External HTTPS request through the Ingress succeeds.
4. `frontend -> api` succeeds; `api -> db` succeeds; `frontend -> db` fails (NetworkPolicy working).
5. `kubectl auth can-i list configmaps --as=system:serviceaccount:final-project:api-sa -n final-project` returns `yes`; `kubectl auth can-i delete pods --as=system:serviceaccount:final-project:api-sa -n final-project` returns `no`.
6. `db`'s Pod reports `Guaranteed` QoS via `jsonpath`.
7. `db`'s PVC shows `Bound` with no manually-created PV.

### Common Errors
This project deliberately doesn't enumerate specific errors — use Lab 9's systematic, five-layer troubleshooting methodology to diagnose whatever issues arise during your own build, exactly as a real engineer would.

### Troubleshooting
Apply Lab 9's full methodology: confirm the symptom, then work through Ingress/Networking (Volume 3), Workload (Volumes 1–2), Configuration/Storage (Volume 4), and Cluster/Node (this volume) layers in turn. Every technique needed to resolve any issue in this project has been covered somewhere in this manual — use this as an opportunity to practice locating the right prior lab's concept independently.

### Practice Tasks
1. Build the complete project from the requirements list without referring back to earlier labs' exact YAML, writing every manifest from your own understanding.
2. Deliberately break one requirement (e.g., remove the NetworkPolicy) and confirm, via testing, exactly what security guarantee is lost.
3. Scale `api` to 5 replicas on a 2-node cluster and observe the required-anti-affinity behavior directly (extra replicas staying Pending).
4. Perform a rolling update on `api` (Volume 2 Lab 5) and confirm zero-downtime behavior via continuous `curl` requests during the rollout.
5. Take an etcd backup (Lab 8) of your test cluster before and after deploying this project, and describe what would be lost if you restored from the "before" snapshot.

### Challenge Lab
Extend the final project with a CronJob (Volume 2 Lab 10) performing a nightly "backup" simulation of the `db` StatefulSet's data (e.g., copying files to a separate PVC), running under its own dedicated, minimally-scoped Service Account — synthesizing yet another cross-volume combination (Jobs/CronJobs, storage, and RBAC together).

### Best Practices
This entire project **is** the accumulated best practices of all five volumes, applied together: least-privilege RBAC, dedicated Service Accounts, anti-affinity for HA, NetworkPolicy least-privilege traffic control, Guaranteed QoS for the stateful component, ResourceQuota/LimitRange namespace governance, readiness probes everywhere, dynamically-provisioned storage, and TLS termination at the Ingress layer — treat this project as your personal reference architecture for real-world Kubernetes deployments going forward.

### Interview Questions
1. **Q: Why does this capstone project intentionally withhold full YAML, unlike earlier labs?** A: To exercise genuine design and synthesis skill, mirroring real-world production deployment work rather than transcription.
2. **Q: Why does `api` need its own dedicated Service Account rather than using `default`?** A: To follow least-privilege RBAC practices from Lab 1–2, granting only the exact permissions `api` actually needs.
3. **Q: Why is `db` implemented as a StatefulSet rather than a Deployment?** A: It needs stable identity and stable, dynamically-provisioned per-Pod storage, exactly the problem StatefulSets solve (Volume 2 Lab 8).
4. **Q: Why does `frontend` need Pod anti-affinity?** A: To ensure its replicas spread across distinct nodes, so a single node failure can't take down every replica simultaneously.
5. **Q: Why is `frontend -> db` blocked even though `frontend -> api -> db` transitively reaches it?** A: Least-privilege network access control — each component should only reach exactly what it needs directly, limiting blast radius if any one component is compromised.
6. **Q: Why does `db` specifically target Guaranteed QoS?** A: As a genuinely critical, stateful workload, it should be evicted last under any node resource pressure.
7. **Q: What would you check first if the external HTTPS request through the Ingress failed?** A: Following Lab 9's methodology — the Ingress/Networking layer first, confirming controller health and certificate configuration.
8. **Q: Why apply a ResourceQuota/LimitRange to this project's namespace?** A: To prevent this application from consuming unbounded cluster resources in a genuinely shared, multi-tenant cluster.
9. **Q: What would an etcd backup taken before this project's deployment be missing if restored?** A: The entire project's resources — every Deployment, Service, Secret, ConfigMap, and PVC created during the build.
10. **Q: How does this project synthesize concepts across all five volumes?** A: Compute/workloads (1-2), networking (3), storage/configuration (4), and administration/security (5) are all combined into one coherent, realistic system.

### Quiz
1. Why is db a StatefulSet, not a Deployment? (a) No reason (b) Needs stable identity and per-Pod storage (c) StatefulSets are simpler (d) Deployments can't run containers — **b**
2. Why does frontend need anti-affinity? (a) No reason (b) Spread replicas so one node failure doesn't take all down (c) Required by Kubernetes always (d) Improves image pulls — **b**
3. Why does api need a dedicated Service Account? (a) Not necessary (b) Least-privilege RBAC (c) Required for all Pods (d) Faster startup — **b**
4. Why block frontend->db directly? (a) No reason (b) Least-privilege network access, limiting blast radius (c) DNS doesn't support it (d) Required by StatefulSets — **b**
5. Why target Guaranteed QoS for db? (a) No reason (b) Evicted last under resource pressure, appropriate for critical stateful workloads (c) Required for StatefulSets always (d) Faster performance — **b**
6. First troubleshooting layer for a failed external HTTPS request: (a) Node layer (b) Ingress/Networking layer (c) RBAC layer (d) Storage layer — **b**
7. Purpose of ResourceQuota/LimitRange here: (a) None (b) Prevent unbounded resource consumption in a shared cluster (c) Required for Ingress (d) Only for Secrets — **b**
8. What's lost if restoring from a pre-deployment etcd backup? (a) Nothing (b) This entire project's resources (c) Only Secrets (d) Only the Ingress — **b**
9. This project combines concepts from: (a) One volume only (b) All five volumes (c) Only Volume 5 (d) Only networking — **b**
10. Why is this capstone specification-driven rather than fully pre-written? (a) No reason (b) To exercise genuine design/synthesis skill (c) To save space (d) Because YAML isn't needed — **b**

### Homework
Build the complete final project from the requirements list, deliberately introduce three different issues (one per layer: networking, workload/config, and cluster/administration), and write a full incident report for each — symptom, diagnosis process using Lab 9's methodology, root cause, and fix — as your final deliverable for this course.

### Summary
This capstone combined Deployments, StatefulSets, Services, Ingress with TLS, ConfigMaps, Secrets, dynamically-provisioned storage, RBAC, dedicated Service Accounts, Pod anti-affinity, NetworkPolicies, and namespace resource governance into one complete, realistic, production-shaped application — synthesizing the entire five-volume course into a single working system you designed and built yourself. This concludes the core curriculum; the manual's final reference section (cheat sheets, command references, and exam preparation guides) follows this volume's own review materials.

---

# Volume 5 — End of Volume Materials

## Volume Review

Volume 5 covered the administrative and operational skills needed to run a Kubernetes cluster responsibly in production:

- **Lab 1–2:** RBAC and Service Accounts — controlling who (human or Pod) can do what, finally explaining what truly protects sensitive resources.
- **Lab 3–4:** Node management (cordon/drain) and scheduling internals (filtering/scoring, nodeSelector).
- **Lab 5–6:** Taints/tolerations (nodes repelling Pods) and affinity/anti-affinity (rich placement and HA spreading).
- **Lab 7–8:** Cluster upgrades (strict version-skew-ordered process) and etcd backup/restore (the ultimate disaster-recovery safety net).
- **Lab 9:** Systematic, five-layer production troubleshooting methodology.
- **Lab 10:** A comprehensive final mini project synthesizing concepts from all five volumes into one complete, realistic system.

You have now completed the full Kubernetes Training Manual — from first principles (Volume 1) through workload controllers (Volume 2), networking (Volume 3), storage and configuration (Volume 4), to cluster administration and production troubleshooting (Volume 5). The final reference section that follows consolidates cheat sheets, command references, and exam preparation materials from across the entire course.

## Practice Exam (25 Questions)

1. What are the four core RBAC objects? — Role, ClusterRole, RoleBinding, ClusterRoleBinding.
2. What's the difference between a Role and a ClusterRole? — Role is namespace-scoped; ClusterRole is cluster-scoped.
3. What Service Account does a Pod get if none is specified? — The namespace's `default` Service Account.
4. Does the default Service Account have RBAC permissions by default? — No.
5. What does cordoning a node do? — Marks it unschedulable for new Pods; existing Pods untouched.
6. What does draining a node do? — Cordons and evicts existing evictable Pods.
7. What are the two phases of scheduling? — Filtering and scoring.
8. What does nodeSelector do? — Constrains a Pod to nodes matching specific labels exactly.
9. How do taints and tolerations differ from nodeSelector conceptually? — Taints are a node opting out; nodeSelector is a Pod opting in.
10. What are the three taint effects? — NoSchedule, PreferNoSchedule, NoExecute.
11. What does NoExecute do differently from NoSchedule? — It also evicts already-running non-tolerating Pods.
12. What's the most common real-world use of Pod anti-affinity? — High-availability spreading of Deployment replicas.
13. What must always be upgraded first in a cluster? — The control plane.
14. What command shows a dry-run of a kubeadm upgrade? — `kubeadm upgrade plan`.
15. Why is etcd backup the most critical disaster-recovery skill? — etcd holds the entire cluster's state.
16. What command creates an etcd snapshot? — `etcdctl snapshot save <path>`.
17. Does etcd restore modify a running cluster in place? — No, it creates a new data directory requiring further integration.
18. What are the five troubleshooting layers, outside-in? — Symptom, Ingress/Networking, Workload, Configuration/Storage, Cluster/Node.
19. What command shows a fast, cluster-wide chronological event feed? — `kubectl get events --sort-by='.lastTimestamp' -A`.
20. What's required for `kubectl top` to work? — The metrics-server addon.
21. What QoS class results from requests exactly equaling limits? — Guaranteed.
22. What's the eviction order under node pressure? — BestEffort first, then Burstable, then Guaranteed last.
23. Why does the final mini project block frontend from reaching db directly? — Least-privilege network access, limiting blast radius.
24. Why give the api component its own Service Account? — Least-privilege RBAC.
25. What single command tests a specific subject's permissions? — `kubectl auth can-i <verb> <resource> --as=<subject>`.

## Practical Assessment

Without referring back to the labs:
1. Create a Role and RoleBinding granting read-only Pod access to a specific user, and verify with `can-i`.
2. Create a dedicated Service Account, bind it a narrow permission set, and use it in a Deployment.
3. Cordon and drain a node, confirming workloads relocate safely, then uncordon it.
4. Create a Pod using nodeSelector and a separate Pod using required node affinity, confirming both land correctly.
5. Taint a node and confirm a non-tolerating Pod is repelled while a tolerating one succeeds.
6. Deploy a 3-replica Deployment with required Pod anti-affinity on a 3-node cluster, confirming perfect spreading.
7. Take an etcd snapshot and verify it with `snapshot status`.
8. Diagnose a deliberately-broken multi-layer issue using the five-layer methodology.
9. Clean up every resource created in this assessment.

**Self-grading:** You should complete every step from memory in under 40 minutes.

## Mini Project Reference
See Lab 10 for the full final capstone project specification (frontend/api/db with RBAC, anti-affinity, NetworkPolicy, and resource governance). Use it as your reference architecture for the Practical Assessment above and for real-world deployments going forward.

## Troubleshooting Exercises

1. **Scenario:** A user reports "Forbidden" errors. **Diagnosis:** Check RBAC bindings for that user/Service Account (Lab 1–2).
2. **Scenario:** New Pods won't schedule onto an apparently-available node. **Diagnosis:** Check for taints, node affinity mismatches, or resource pressure (Labs 4–5).
3. **Scenario:** A Deployment's replicas are all on one node. **Diagnosis:** Check whether Pod anti-affinity is configured, and whether it's required or preferred (Lab 6).
4. **Scenario:** A cluster upgrade causes unexpected node failures. **Diagnosis:** Verify version skew policy was followed and nodes were properly drained first (Lab 7).
5. **Scenario:** A multi-layer production issue with unclear root cause. **Diagnosis:** Apply the five-layer systematic methodology from Lab 9.

## Volume 5 Cheat Sheet

| Task | Command |
|---|---|
| Test permissions | `kubectl auth can-i <verb> <resource> --as=<user> [-n <ns>]` |
| Audit bindings for a subject | `kubectl get rolebindings,clusterrolebindings -A \| grep <subject>` |
| Cordon a node | `kubectl cordon <node>` |
| Drain a node | `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` |
| Uncordon a node | `kubectl uncordon <node>` |
| Taint a node | `kubectl taint node <node> key=value:effect` |
| Remove a taint | `kubectl taint node <node> key=value:effect-` |
| Check Pod QoS class | `kubectl get pod <name> -o jsonpath='{.status.qosClass}'` |
| Cluster-wide recent events | `kubectl get events --sort-by='.lastTimestamp' -A` |
| Live resource usage | `kubectl top nodes` / `kubectl top pods -A` |
| Snapshot etcd | `etcdctl snapshot save <path>` |
| Verify a snapshot | `etcdctl snapshot status <path> --write-out=table` |

## kubectl Command Reference (Volume 5 subset)

`auth can-i`, `cordon`, `drain`, `uncordon`, `taint node`, `label node`, `get events --sort-by`, `top nodes`, `top pods`, `create serviceaccount`, `get rolebindings`, `get clusterrolebindings`, plus `etcdctl snapshot save/status/restore` (run outside `kubectl` directly against etcd).

## YAML Reference (Volume 5 subset)

| Object | apiVersion | Key spec fields |
|---|---|---|
| Role / ClusterRole | rbac.authorization.k8s.io/v1 | rules (apiGroups, resources, verbs) |
| RoleBinding / ClusterRoleBinding | rbac.authorization.k8s.io/v1 | subjects, roleRef |
| ServiceAccount | v1 | (typically minimal; referenced via spec.serviceAccountName on Pods) |
| Pod (affinity) | v1 | spec.affinity.nodeAffinity / podAffinity / podAntiAffinity |
| Pod (tolerations) | v1 | spec.tolerations (key, operator, value, effect, tolerationSeconds) |

---

*End of Volume 5 — Administration & Troubleshooting, and end of the core five-volume course. Continue to the Final Reference Section: Complete Cheat Sheet, 200 kubectl Commands, 100 YAML Examples, Architecture Reference, Best Practices, CKA/CKAD Tips, Interview Preparation Guide, and Glossary.*
