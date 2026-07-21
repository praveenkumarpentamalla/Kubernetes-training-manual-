# Kubernetes Training Manual

## Volume 2 — Pods & Workloads

**A Self-Study Handbook for Beginners to Intermediate Learners**

---

### Copyright Notice

© 2026. This training manual is an original work created for self-study purposes. All commands, YAML examples, diagrams, and explanations in this document have been written from scratch. Kubernetes®, Docker®, and related marks are trademarks of their respective owners (The Linux Foundation, Docker Inc.). This manual is not affiliated with or endorsed by any of these organizations.

---

### About This Volume

Volume 1 taught you to create and manage raw Pods by hand. In real clusters, you almost never do that — instead you describe your workload using a higher-level **controller** object, and Kubernetes creates, monitors, heals, and scales the underlying Pods for you. Volume 2 covers every major workload controller: multi-container Pod patterns, Init Containers, ReplicaSets, Deployments (with rolling updates and rollbacks), DaemonSets, StatefulSets, Jobs, and CronJobs. By the end of this volume you will know which controller to reach for, for almost any real-world workload shape.

**Prerequisites for this volume:** Completion of Volume 1, including a working Kind or Minikube cluster and comfort with `kubectl`, Pod YAML, labels/selectors, and namespaces.

---

## Table of Contents — Volume 2

- Lab 1 — Multi-Container Pods
- Lab 2 — Init Containers
- Lab 3 — ReplicaSets
- Lab 4 — Deployments
- Lab 5 — Rolling Updates
- Lab 6 — Rollbacks
- Lab 7 — DaemonSets
- Lab 8 — StatefulSets
- Lab 9 — Jobs
- Lab 10 — CronJobs & Mini Project
- Volume 2 Review, Exam, and Reference

---

# Lab 1 — Multi-Container Pods

**Estimated Duration:** 50 minutes
**Difficulty Level:** Beginner–Intermediate

### Learning Objectives
1. Explain why and when to run more than one container in a single Pod.
2. Implement the sidecar, ambassador, and adapter patterns conceptually.
3. Configure inter-container communication via `localhost` and shared volumes.
4. Debug a specific container inside a multi-container Pod.

### Prerequisites
- Completion of Volume 1, especially Lab 6 (Creating Your First Pod).

### Theory

Volume 1 introduced Pods as usually running a single container, but a Pod can hold several containers that are always co-scheduled onto the same node, share the same network namespace (so they can reach each other over `localhost`), and can share storage volumes. This is useful only when the containers are **tightly coupled** — if two pieces of an application scale independently or fail independently, they belong in separate Pods, not the same one.

Three common multi-container patterns:

- **Sidecar** — a helper container that extends the main container's functionality, such as a log-shipping agent that reads logs written to a shared volume and forwards them to a central logging system.
- **Ambassador** — a proxy container that simplifies how the main container talks to the outside world, e.g., a local proxy that handles retries or TLS termination on behalf of the main application.
- **Adapter** — a container that standardizes or transforms the main container's output, for example converting a legacy application's non-standard metrics format into the Prometheus format.

Containers within a Pod can share files via an `emptyDir` volume (a temporary directory created when the Pod starts and deleted when it's removed) mounted into more than one container. This is the most common way a sidecar reads what the main container writes.

When more than one container exists in a Pod, most `kubectl` commands (`logs`, `exec`) require you to specify **which** container with `-c <container-name>`, since Kubernetes cannot guess which one you mean.

### Architecture Diagram

```
                    Pod
   +----------------------------------------------+
   |  Shared network namespace + emptyDir volume     |
   |                                                |
   |  +----------------+     +--------------------+ |
   |  | Container:      |     | Container:          | |
   |  | app (writes      | --> | log-shipper         | |
   |  | logs to volume)  |     | (sidecar, reads      | |
   |  |                  |     |  volume, forwards)    | |
   |  +----------------+     +--------------------+ |
   +----------------------------------------------+
```

### Installation
No new installation — uses your existing cluster.

### Hands-on Practice

```bash
kubectl logs sidecar-demo -c app
kubectl logs sidecar-demo -c log-shipper
```
**Purpose:** Views logs from a specific container when a Pod has more than one.
**Syntax:** `-c <container-name>` is required once a Pod has multiple containers; omitting it returns an error listing valid container names.
**Expected Output:** Separate log streams for each container.
**Common mistakes:** Running `kubectl logs sidecar-demo` without `-c` and being confused by the resulting error instead of realizing multiple containers exist.
**Real-world usage:** Standard practice whenever debugging sidecar-based logging or service-mesh proxy setups (e.g., Istio's Envoy sidecar).

```bash
kubectl exec -it sidecar-demo -c log-shipper -- sh
```
**Purpose:** Opens a shell inside a specific container within a multi-container Pod.
**Syntax:** `-c` again required to disambiguate.
**Common mistakes:** Assuming `exec` defaults to the "main" container — Kubernetes has no concept of a "main" container and will error if `-c` is omitted and more than one container exists.
**Real-world usage:** Inspecting a sidecar's shared volume contents to confirm it's reading what the main container is writing.

### YAML Files
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
  labels:
    app: sidecar-demo
spec:
  volumes:
    - name: shared-logs
      emptyDir: {}              # Temporary volume, exists for the Pod's lifetime, shared between containers
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "while true; do echo $(date) 'app is running' >> /var/log/app.log; sleep 5; done"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log    # app writes logs here
    - name: log-shipper
      image: busybox
      command: ["sh", "-c", "tail -f /var/log/app.log"]   # sidecar reads the same files
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log    # same volume, mounted read-effectively by convention (no readOnly enforced here)
```

### Expected Output
```
NAME            READY   STATUS    RESTARTS   AGE
sidecar-demo    2/2     Running   0          15s
```
`kubectl logs sidecar-demo -c log-shipper` should show lines like:
```
Mon Jul 20 09:00:05 UTC 2026 app is running
Mon Jul 20 09:00:10 UTC 2026 app is running
```

### Validation Steps
Confirm `READY` shows `2/2` (both containers healthy), and confirm the `log-shipper` container's logs show the same content the `app` container is writing, proving the shared volume works.

### Common Errors
- **"a container name must be specified"** — occurs on `logs`/`exec` when a Pod has multiple containers and `-c` is omitted.
- **`READY: 1/2`** — one container is healthy but the other is crashing; use `-c` with `describe`/`logs` to isolate which one.
- **Sidecar not seeing any data** — usually a `mountPath` mismatch between the two containers.

### Troubleshooting
Always pass `-c <container-name>` once you have more than one container. For `1/2` readiness, run `kubectl describe pod <name>` and check the `Containers` section for each container's individual state. For a sidecar seeing no data, double-check both containers mount the *same* volume name at consistent paths.

### Practice Tasks
1. Add a third container to the sidecar-demo Pod that also reads `/var/log/app.log` and counts lines.
2. Deliberately mismatch the `mountPath` between the two containers and observe the sidecar stops seeing new data.
3. Practice `kubectl exec -c` into each container separately and confirm they have different filesystems outside the shared volume.
4. Implement a simple ambassador pattern: a busybox container that runs `nc` (netcat) to relay traffic to another container.
5. Delete just the Pod (not the whole namespace) and confirm both containers terminate together, since they cannot exist independently.

### Challenge Lab
Design (write YAML for) a three-container Pod: a `web` container serving files from a shared volume, an `init`-style content-generator container (don't worry — proper Init Containers are covered in Lab 2) that writes an `index.html` into that shared volume, and confirm the web server serves the generated content after Pod startup.

### Best Practices
- Only combine containers in one Pod when they are truly co-dependent and must scale together; otherwise use separate Pods/Deployments.
- Always name containers descriptively (`app`, `log-shipper`, not `container1`, `container2`) since you'll type `-c <name>` constantly while debugging.
- Use `emptyDir` for ephemeral shared data only — it is deleted when the Pod is removed, so it is never suitable for anything you need to persist.

### Interview Questions
1. **Q: When should you use a multi-container Pod instead of separate Pods?** A: Only when containers are tightly coupled and must be co-scheduled, share network/storage, and scale together — e.g., sidecar logging or proxying.
2. **Q: What is the sidecar pattern?** A: A helper container that extends the main container's functionality, such as shipping logs.
3. **Q: What is the ambassador pattern?** A: A proxy container simplifying how the main container talks to external systems.
4. **Q: What is the adapter pattern?** A: A container that transforms/standardizes the main container's output format.
5. **Q: How do containers in a Pod share files?** A: Via a shared volume, most commonly `emptyDir`, mounted into both containers.
6. **Q: How do containers in a Pod communicate over the network?** A: Via `localhost`, since they share the same network namespace.
7. **Q: What happens if you run `kubectl logs` without `-c` on a multi-container Pod?** A: Kubernetes returns an error requiring you to specify a container name.
8. **Q: Is `emptyDir` data persisted if the Pod is deleted?** A: No — it is ephemeral and tied to the Pod's lifetime.
9. **Q: Can containers within a Pod scale independently?** A: No, they are always scheduled and scaled together as one unit.
10. **Q: Give a real-world example of a sidecar container.** A: An Envoy proxy sidecar in a service mesh, or a log-shipping agent like Fluent Bit.

### Quiz
1. Containers in the same Pod share: (a) Nothing (b) Network namespace, optionally volumes (c) Separate IPs (d) Separate nodes — **b**
2. Which pattern involves a helper container extending main functionality? (a) Ambassador (b) Sidecar (c) Adapter (d) Init — **b**
3. Which flag specifies a container for `logs`/`exec` in a multi-container Pod? (a) `--container` only (b) `-c` (c) `--name` (d) `-n` — **b**
4. `emptyDir` volume data persists: (a) Forever (b) Only for the Pod's lifetime (c) Across cluster restarts (d) On a remote disk permanently — **b**
5. Can multi-container Pod containers scale independently? (a) Yes (b) No — **b**
6. The ambassador pattern is used for: (a) Logging (b) Proxying external communication (c) Metrics conversion (d) Health checks — **b**
7. The adapter pattern is used for: (a) Proxying (b) Transforming/standardizing output (c) Scheduling (d) Networking — **b**
8. What error occurs if `-c` is omitted on a multi-container Pod's `kubectl logs`? (a) Silent failure (b) "a container name must be specified" (c) Random container chosen (d) Pod restarts — **b**
9. Multi-container Pods should be used when: (a) Containers are unrelated (b) Containers are tightly coupled and co-dependent (c) You want independent scaling (d) Never — **b**
10. Containers in a Pod communicate over the network via: (a) External IPs (b) localhost (c) DNS only (d) VPN — **b**

### Homework
Build a three-container Pod implementing all three patterns (sidecar, ambassador, adapter) in one manifest, even if simplistically simulated with busybox scripts, and write a short explanation of each container's role.

### Summary
Multi-container Pods are appropriate only for tightly coupled helper patterns like sidecars, ambassadors, and adapters, sharing network and optionally storage. Lab 2 covers Init Containers, a related but distinct concept for setup tasks that must complete before your main containers start.

---

# Lab 2 — Init Containers

**Estimated Duration:** 45 minutes
**Difficulty Level:** Beginner–Intermediate

### Learning Objectives
1. Explain what Init Containers are and how they differ from regular containers.
2. Use Init Containers to perform setup tasks before an application starts.
3. Chain multiple Init Containers and understand their sequential execution order.
4. Debug a Pod stuck due to a failing Init Container.

### Prerequisites
- Completion of Lab 1.

### Theory

An **Init Container** is a special container defined in a Pod spec that runs and must complete **before** any of the Pod's regular (main) containers start. Init Containers always run to completion, one at a time, in the order they are listed. If an Init Container fails, Kubernetes retries it (subject to `restartPolicy`), and the main containers do not start until every Init Container has succeeded.

Common use cases: waiting for a dependency to become available (e.g., blocking until a database is reachable), downloading or generating configuration/content into a shared volume before the main container starts, or running a one-time database migration.

Init Containers differ from sidecars in an important way: sidecars run **alongside** the main container for the Pod's entire lifetime, while Init Containers run and finish **before** the main container even starts, then never run again for that Pod's lifetime.

### Architecture Diagram

```
   Pod startup sequence:

   [Init Container 1] --success--> [Init Container 2] --success--> [Main Container starts]
         |                                |
       failure                         failure
         |                                |
         v                                v
   retried per restartPolicy       retried per restartPolicy
   (main container never starts until ALL init containers succeed)
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl apply -f init-demo.yaml
kubectl get pods -w
```
**Purpose:** Applies a Pod with an Init Container and watches its status transition through `Init:0/1` before reaching `Running`.
**Expected Output:**
```
NAME        READY   STATUS     RESTARTS   AGE
init-demo   0/1     Init:0/1   0          2s
init-demo   0/1     PodInitializing   0    6s
init-demo   1/1     Running    0          7s
```
**Common mistakes:** Assuming a Pod stuck at `Init:0/1` for a long time is broken, without checking `kubectl logs -c <init-container-name>` first.
**Real-world usage:** Waiting for a dependency (e.g., a database Service) to be reachable before starting the main application, avoiding crash loops caused by starting too early.

```bash
kubectl logs init-demo -c wait-for-dependency
```
**Purpose:** Views logs specifically from the (already completed) Init Container, useful for debugging why it took a long time or failed.
**Common mistakes:** Forgetting `-c` and getting logs from the main container instead, missing the actual problem.
**Real-world usage:** First place to look when a Pod is stuck in `Init:X/Y` for longer than expected.

### YAML Files

`init-demo.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
    - name: wait-for-dependency
      image: busybox
      command: ["sh", "-c", "echo waiting; sleep 5; echo dependency ready"]  # simulates waiting for a DB
  containers:
    - name: main-app
      image: nginx:1.25
      ports:
        - containerPort: 80
```

Multiple, sequential Init Containers:
```yaml
initContainers:
  - name: check-network      # runs first
    image: busybox
    command: ["sh", "-c", "echo checking network; sleep 2"]
  - name: fetch-config       # runs second, only after check-network succeeds
    image: busybox
    command: ["sh", "-c", "echo fetching config; sleep 2"]
```

### Expected Output
```
$ kubectl describe pod init-demo
...
Init Containers:
  wait-for-dependency:
    State:  Terminated
      Reason:  Completed
      Exit Code: 0
Containers:
  main-app:
    State: Running
```

### Validation Steps
`kubectl get pod init-demo` should show `READY: 1/1` and `STATUS: Running` only after the Init Container's logs show `dependency ready`, confirming ordering was respected.

### Common Errors
- **Pod stuck at `Init:0/1` indefinitely** — the Init Container is failing repeatedly; check its logs.
- **`Init:Error` or `Init:CrashLoopBackOff`** — the Init Container's command exited non-zero.
- **Main container never starts even though Init Container logs show success** — check for a typo in `initContainers` (must be plural and correctly nested under `spec`).

### Troubleshooting
Run `kubectl logs <pod> -c <init-container-name>` to see exactly why an Init Container is failing or slow. Run `kubectl describe pod <pod>` and inspect the `Init Containers` section for exit codes and reasons. Remember Init Containers respect the Pod's `restartPolicy` just like main containers do.

### Practice Tasks
1. Create a Pod with two sequential Init Containers and confirm via logs that they run in the defined order, not in parallel.
2. Make an Init Container deliberately fail (`exit 1`) and observe the resulting `Init:Error`/`Init:CrashLoopBackOff` status.
3. Use an Init Container to write a file into an `emptyDir` volume that the main container then serves (combine with Lab 1 concepts).
4. Time how long a Pod takes to become `Running` with a 10-second-sleep Init Container versus none.
5. Use `kubectl describe pod` to find the exact exit code of a failed Init Container.

### Challenge Lab
Build a Pod where an Init Container downloads (simulate with a busybox `wget` against a small public URL, or generate the content in-line with `echo`) an `index.html` into a shared `emptyDir` volume, and an `nginx` main container serves it. Confirm the generated content appears when you `curl` the Pod's IP.

### Best Practices
- Use Init Containers for setup logic that truly must complete before the app starts — don't overuse them for things a readiness probe could instead handle.
- Keep Init Containers small and fast; they add directly to Pod startup latency.
- Log clearly from Init Containers, since their output is often the only clue when a Pod is stuck initializing.

### Interview Questions
1. **Q: What is an Init Container?** A: A container that runs to completion before any main containers in the Pod start.
2. **Q: How do multiple Init Containers execute?** A: Sequentially, in the order listed, each must succeed before the next runs.
3. **Q: What happens if an Init Container fails?** A: It is retried according to the Pod's restartPolicy; main containers never start until all Init Containers succeed.
4. **Q: How is an Init Container different from a sidecar?** A: An Init Container runs and finishes before the main container starts; a sidecar runs alongside the main container for the Pod's whole lifetime.
5. **Q: Give a real-world use case for an Init Container.** A: Waiting for a database to become reachable before starting the application, or fetching configuration/content into a shared volume.
6. **Q: What status does `kubectl get pods` show while an Init Container is running?** A: `Init:X/Y`, e.g., `Init:0/1`.
7. **Q: How do you view logs from an Init Container?** A: `kubectl logs <pod> -c <init-container-name>`.
8. **Q: Can Init Containers share volumes with main containers?** A: Yes, using the same volume-mounting mechanism as regular containers.
9. **Q: Do Init Containers count toward the Pod's `READY` count?** A: No, `READY` reflects only main containers.
10. **Q: Why should Init Containers be kept fast?** A: Because they directly delay Pod startup — the main container cannot start until they finish.

### Quiz
1. Init Containers run: (a) Alongside main containers forever (b) Before main containers, to completion (c) After main containers (d) Never — **b**
2. Multiple Init Containers execute: (a) In parallel (b) Sequentially in listed order (c) Randomly (d) Only the first one runs — **b**
3. What status shows a Pod initializing? (a) Running (b) Init:X/Y (c) Pending only (d) Succeeded — **b**
4. How do you view Init Container logs? (a) kubectl logs (no flag needed) (b) kubectl logs <pod> -c <init-container-name> (c) kubectl describe only (d) Not possible — **b**
5. Init Containers differ from sidecars because: (a) They run forever like sidecars (b) They finish before main containers start (c) They cannot use volumes (d) They are the same thing — **b**
6. If an Init Container fails, the main container: (a) Starts anyway (b) Does not start until Init Containers succeed (c) Crashes immediately (d) Is skipped — **b**
7. Do Init Containers count toward Pod READY count? (a) Yes (b) No — **b**
8. A common Init Container use case is: (a) Serving persistent traffic (b) Waiting for a dependency to be ready (c) Logging forever (d) Load balancing — **b**
9. Init Containers are defined under which Pod spec field? (a) spec.containers (b) spec.initContainers (c) spec.setup (d) spec.preContainers — **b**
10. Why keep Init Containers fast? (a) No reason (b) They delay Pod startup directly (c) They consume no resources (d) They run in parallel anyway — **b**

### Homework
Build a Pod with three sequential Init Containers simulating: (1) a network check, (2) a config fetch, (3) a database migration — each just using `echo`/`sleep` — then document the exact total startup delay you observed and why.

### Summary
Init Containers provide a clean way to run setup logic that must complete before your application starts, executing sequentially and blocking main container startup until they succeed. Lab 3 moves from Pod-level patterns to your first true workload controller: the ReplicaSet.

---

# Lab 3 — ReplicaSets

**Estimated Duration:** 50 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain the purpose of a ReplicaSet and how it relates to Pods.
2. Write and apply a ReplicaSet manifest.
3. Observe self-healing behavior when a Pod is manually deleted.
4. Scale a ReplicaSet up and down.

### Prerequisites
- Completion of Lab 2.

### Theory

A **ReplicaSet** is a controller whose sole job is to ensure a specified number of identical Pod replicas are running at all times. It does this using the label selector mechanism from Volume 1: the ReplicaSet's `spec.selector` defines which Pods it considers "its own," and its `spec.template` defines exactly what a Pod should look like if the ReplicaSet needs to create a new one.

If a Pod matching the selector is deleted (crashes, is evicted, or is manually removed), the ReplicaSet controller notices the discrepancy between desired count (`spec.replicas`) and actual count, and creates a replacement Pod automatically. This is the self-healing behavior that raw Pods (Volume 1) never had on their own.

In practice, you will almost never create a ReplicaSet directly — you'll use a **Deployment** (Lab 4), which manages ReplicaSets for you and adds rolling update and rollback capabilities on top. Understanding ReplicaSets first, though, makes Deployments far easier to reason about, since a Deployment is essentially "a ReplicaSet manager."

A critical detail: a ReplicaSet's `spec.template.metadata.labels` **must** match `spec.selector.matchLabels` — this is a common source of validation errors for beginners.

### Architecture Diagram

```
   ReplicaSet (desired: 3 replicas, selector: app=web)
        |
        | continuously compares desired vs actual
        v
   +--------+   +--------+   +--------+
   | Pod: web |   | Pod: web |   | Pod: web |     <- 3 Pods matching selector app=web
   | (1/1)    |   | (1/1)    |   | (1/1)    |
   +--------+   +--------+   +--------+

   If one Pod is deleted -> ReplicaSet detects only 2/3 -> creates a replacement automatically
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl apply -f web-replicaset.yaml
kubectl get replicasets
kubectl get pods -l app=web
```
**Purpose:** Creates the ReplicaSet and confirms it created the desired number of matching Pods.
**Expected Output:**
```
NAME   DESIRED   CURRENT   READY   AGE
web    3         3         3       10s
```
**Common mistakes:** Mismatched `selector` and `template` labels, which is rejected outright by the API server with a validation error.
**Real-world usage:** Rare to manage directly, but essential for understanding what a Deployment does under the hood.

```bash
kubectl delete pod <one-of-the-web-pods>
kubectl get pods -l app=web -w
```
**Purpose:** Demonstrates self-healing — deletes one Pod manually and watches the ReplicaSet immediately create a replacement.
**Expected Output:** Pod count briefly drops to 2, then a new Pod appears within seconds, returning to 3.
**Common mistakes:** Expecting the *same* Pod name to come back — it won't; a brand-new Pod with a new name and IP is created.
**Real-world usage:** This is the exact mechanism that keeps production services running despite node failures or crashes.

```bash
kubectl scale replicaset web --replicas=5
```
**Purpose:** Changes the desired replica count imperatively.
**Expected Output:** `replicaset.apps/web scaled`, followed by 2 additional Pods being created.
**Real-world usage:** Manual scaling for sudden traffic spikes, though Horizontal Pod Autoscaling (a more advanced topic) automates this in production.

### YAML Files

`web-replicaset.yaml`:
```yaml
apiVersion: apps/v1              # ReplicaSets live in the "apps/v1" API group, not core "v1"
kind: ReplicaSet
metadata:
  name: web
spec:
  replicas: 3                    # Desired number of Pod copies
  selector:
    matchLabels:
      app: web                   # Which Pods this ReplicaSet manages
  template:                      # Blueprint used to create new Pods
    metadata:
      labels:
        app: web                 # MUST match spec.selector.matchLabels above
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests: { cpu: "100m", memory: "64Mi" }
            limits: { cpu: "250m", memory: "128Mi" }
```

### Expected Output
```
NAME   DESIRED   CURRENT   READY   AGE
web    3         3         3       30s
```

### Validation Steps
1. `kubectl get pods -l app=web` shows exactly 3 (or your scaled count) Pods, all `Running`.
2. After deleting one Pod, confirm a new one appears automatically without any manual `kubectl apply` needed.

### Common Errors
- **"selector does not match template labels"** — the API server rejects a ReplicaSet where `spec.selector.matchLabels` and `spec.template.metadata.labels` differ.
- **Pods not being replaced after deletion** — usually means you deleted the ReplicaSet itself, not just a Pod, or labels drifted.
- **Too many Pods running** — an old ReplicaSet with an overlapping selector wasn't cleaned up, causing two controllers to "fight" over the same Pods.

### Troubleshooting
For selector mismatches, ensure the two label blocks are identical. If Pods aren't replaced, confirm with `kubectl get replicaset` that the ReplicaSet itself still exists. For "fighting" controllers, ensure each ReplicaSet has a unique, non-overlapping selector.

### Practice Tasks
1. Create a ReplicaSet with 4 replicas and scale it down to 1, observing which Pods get terminated.
2. Manually delete two Pods simultaneously and confirm both are replaced.
3. Change the ReplicaSet's `template` image version and apply it — observe that existing Pods are *not* automatically updated (ReplicaSets do not perform rolling updates; this limitation motivates Deployments in Lab 4).
4. Intentionally create a selector/template label mismatch and read the resulting validation error carefully.
5. Delete the ReplicaSet (not just a Pod) and confirm all managed Pods are also deleted.

### Challenge Lab
Create two ReplicaSets with different `app` label values (`app=web-v1` and `app=web-v2`) both running nginx, scale each independently, and use label selectors to prove `kubectl get pods` can distinguish and target each group separately.

### Best Practices
- Avoid creating ReplicaSets directly in production; use Deployments instead, which manage ReplicaSets and add safe rollout capabilities.
- Never let two ReplicaSets share an overlapping selector — this causes unpredictable Pod ownership conflicts.
- Always double check that `selector.matchLabels` and `template.metadata.labels` match exactly before applying.

### Interview Questions
1. **Q: What does a ReplicaSet do?** A: Ensures a specified number of identical Pod replicas are running at all times, replacing failed ones automatically.
2. **Q: How does a ReplicaSet know which Pods belong to it?** A: Via its label selector matching Pod labels.
3. **Q: What happens if you manually delete a Pod managed by a ReplicaSet?** A: The ReplicaSet detects the count discrepancy and creates a new replacement Pod.
4. **Q: Can a ReplicaSet perform rolling updates?** A: No — updating the Pod template does not automatically update existing Pods; this is why Deployments exist.
5. **Q: What must match between `selector.matchLabels` and `template.metadata.labels`?** A: They must be consistent, or the API server rejects the ReplicaSet.
6. **Q: How do you scale a ReplicaSet imperatively?** A: `kubectl scale replicaset <name> --replicas=<n>`.
7. **Q: What API group do ReplicaSets belong to?** A: `apps/v1`.
8. **Q: Why do most engineers use Deployments instead of ReplicaSets directly?** A: Deployments manage ReplicaSets automatically and add rolling updates/rollbacks on top.
9. **Q: What happens if two ReplicaSets have overlapping selectors?** A: They can conflict over ownership of the same Pods, causing unpredictable behavior.
10. **Q: Does the replaced Pod reuse the old Pod's name and IP?** A: No, a brand-new Pod with a new name and IP address is created.

### Quiz
1. What does a ReplicaSet guarantee? (a) Rolling updates (b) A specified number of Pod replicas running (c) Node health (d) Namespace isolation — **b**
2. ReplicaSets belong to which API group? (a) v1 (b) apps/v1 (c) batch/v1 (d) networking/v1 — **b**
3. If a managed Pod is deleted, the ReplicaSet: (a) Does nothing (b) Creates a replacement automatically (c) Deletes all Pods (d) Pauses — **b**
4. Can updating a ReplicaSet's template auto-update existing Pods? (a) Yes (b) No — **b**
5. What must selector.matchLabels match? (a) Node labels (b) template.metadata.labels (c) Namespace name (d) Nothing — **b**
6. Command to scale a ReplicaSet: (a) kubectl resize (b) kubectl scale replicaset <name> --replicas=<n> (c) kubectl replicas set (d) kubectl update replicaset — **b**
7. What replaces a deleted Pod's identity? (a) Same name/IP (b) A brand new name/IP (c) No replacement (d) Manual recreation only — **b**
8. Why prefer Deployments over ReplicaSets directly? (a) Deployments are slower (b) Deployments add rolling updates/rollbacks (c) ReplicaSets can't scale (d) No difference — **b**
9. Overlapping selectors between two ReplicaSets cause: (a) Improved performance (b) Ownership conflicts (c) Automatic merging (d) No effect — **b**
10. ReplicaSet's core self-healing mechanism relies on: (a) Manual intervention (b) Continuous desired vs actual state comparison (c) Node restarts (d) DNS — **b**

### Homework
Create a ReplicaSet with 3 replicas, then write a small bash loop that deletes a random Pod from it every 10 seconds for one minute, and document how quickly and reliably the ReplicaSet replaces each one.

### Summary
ReplicaSets provide the self-healing and fixed-replica-count guarantees that raw Pods lack, using the label selector mechanism from Volume 1. Their key limitation — no rolling update support — is exactly what Deployments, covered next in Lab 4, are built to solve.

---

# Lab 4 — Deployments

**Estimated Duration:** 60 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain how a Deployment relates to and manages ReplicaSets.
2. Create, scale, and update a Deployment.
3. Understand the `.spec.strategy` field and its defaults.
4. Inspect Deployment status and rollout history.

### Prerequisites
- Completion of Lab 3.

### Theory

A **Deployment** is a controller that manages ReplicaSets on your behalf, adding the crucial capability ReplicaSets lack: safe, controlled updates to your application. When you change a Deployment's Pod template (e.g., updating the container image), the Deployment creates a **new** ReplicaSet with the updated template and gradually shifts traffic from the old ReplicaSet to the new one, rather than replacing everything at once.

This is the object you will use for the vast majority of stateless application workloads in Kubernetes — web servers, APIs, background workers — anything where individual Pod identity doesn't matter and any replica can serve any request.

Every Deployment keeps a **revision history** of its ReplicaSets (by default the last 10, controlled by `spec.revisionHistoryLimit`), which is what enables the rollback capability covered in Lab 6.

Key fields: `spec.replicas` (desired Pod count), `spec.selector` (must match `template.metadata.labels`, same rule as ReplicaSets), `spec.template` (the Pod blueprint), and `spec.strategy` (defaults to `RollingUpdate`, covered fully in Lab 5).

### Architecture Diagram

```
   Deployment: web (manages ReplicaSets)
        |
        | creates/owns
        v
   ReplicaSet: web-7d9f8c (old version)  --being scaled down-->  0 Pods
   ReplicaSet: web-6b7a1e (new version)  --being scaled up  -->  3 Pods

   You never touch ReplicaSets directly; the Deployment orchestrates
   the transition between old and new ReplicaSets automatically.
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl get deployments
kubectl get replicasets
kubectl get pods
```
**Purpose:** Imperatively creates a Deployment (equivalent to applying the YAML below) and shows the resulting ReplicaSet and Pods it created automatically.
**Expected Output:**
```
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    3/3     3            3           15s
```
**Common mistakes:** Trying to `kubectl edit` a Pod created by a Deployment directly — changes will be reverted, since the Deployment/ReplicaSet controller enforces the template as the source of truth.
**Real-world usage:** The default way to launch nearly any stateless application onto a cluster.

```bash
kubectl rollout status deployment/web
```
**Purpose:** Watches a Deployment's rollout progress until it completes or fails.
**Expected Output:** `deployment "web" successfully rolled out`
**Real-world usage:** Used in CI/CD pipelines as a blocking step to confirm a deploy succeeded before proceeding.

```bash
kubectl describe deployment web
```
**Purpose:** Shows full Deployment details including strategy, conditions, and recent Events (including which ReplicaSets were scaled up/down).
**Real-world usage:** Primary tool for diagnosing a stuck or failing rollout.

### YAML Files

`web-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 3
  revisionHistoryLimit: 10        # How many old ReplicaSets to retain for rollback purposes
  selector:
    matchLabels:
      app: web                    # Must match template labels below
  strategy:
    type: RollingUpdate            # Default strategy; the alternative is "Recreate" (covered in Lab 5)
    rollingUpdate:
      maxUnavailable: 1            # At most 1 Pod below desired count during rollout
      maxSurge: 1                  # At most 1 extra Pod above desired count during rollout
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests: { cpu: "100m", memory: "64Mi" }
            limits: { cpu: "250m", memory: "128Mi" }
          readinessProbe:
            httpGet: { path: /, port: 80 }
            initialDelaySeconds: 5
```

### Expected Output
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/web   3/3   3            3           20s

NAME                          DESIRED   CURRENT   READY   AGE
replicaset.apps/web-6b7a1e    3         3         3       20s
```

### Validation Steps
1. `kubectl get deployment web` shows `READY: 3/3`.
2. `kubectl get replicasets -l app=web` shows exactly one ReplicaSet owning all 3 Pods.
3. `kubectl rollout status deployment/web` returns success immediately (no pending rollout).

### Common Errors
- **Deployment stuck at `READY: 2/3`** — one Pod failing readiness probe; check `kubectl describe pod`.
- **Multiple ReplicaSets both showing active Pods** — a rollout is in progress or stuck partway.
- **"error: Scaled object ... failed"** — usually a resource quota or scheduling constraint preventing new Pods.

### Troubleshooting
For stuck rollouts, run `kubectl rollout status deployment/<name>` to see exactly where it's blocked, then `kubectl describe pod <failing-pod>` for root cause (often a failing readiness probe or resource shortage). For quota errors, check `kubectl describe resourcequota -n <namespace>` (ResourceQuotas are covered fully in Volume 4).

### Practice Tasks
1. Create a Deployment via YAML (not the imperative `kubectl create deployment` command) with 4 replicas.
2. Scale it up and down using `kubectl scale deployment web --replicas=<n>`.
3. Run `kubectl get pods -l app=web -o wide` and confirm all Pods share the same `app=web` label inherited from the template.
4. Delete one of the Pods a Deployment manages and confirm it's replaced (proving the underlying ReplicaSet is still doing its job).
5. Compare `kubectl edit deployment web` (which works and persists) versus `kubectl edit pod <pod-name>` (whose changes get reverted) to internalize the ownership hierarchy.

### Challenge Lab
Create a Deployment with `maxUnavailable: 0` and `maxSurge: 2`, apply an image update, and observe (via `kubectl get pods -w`) that the rollout always keeps at least the full desired count available throughout, using extra "surge" Pods instead of ever dropping below the desired count.

### Best Practices
- Always define readiness probes on Deployment Pods — without them, the rolling update strategy can't safely determine when a new Pod is actually ready to receive traffic.
- Never edit Pods managed by a Deployment directly; always change the Deployment.
- Set explicit `resources.requests/limits` on every container so autoscaling and scheduling behave predictably.

### Interview Questions
1. **Q: What is the relationship between a Deployment and a ReplicaSet?** A: A Deployment manages one or more ReplicaSets, creating a new one for each template change and orchestrating the transition between them.
2. **Q: What capability does a Deployment add on top of a ReplicaSet?** A: Safe, controlled rolling updates and rollback support.
3. **Q: What is the default update strategy?** A: RollingUpdate.
4. **Q: What does `maxUnavailable` control?** A: The maximum number of Pods that can be unavailable below the desired count during a rollout.
5. **Q: What does `maxSurge` control?** A: The maximum number of extra Pods allowed above the desired count during a rollout.
6. **Q: How do you check a Deployment's rollout status?** A: `kubectl rollout status deployment/<name>`.
7. **Q: What happens if you edit a Pod managed by a Deployment directly?** A: The change is eventually reverted, since the Deployment/ReplicaSet controller enforces its template as the source of truth.
8. **Q: How many old ReplicaSets does a Deployment retain by default?** A: Controlled by `revisionHistoryLimit`, default 10.
9. **Q: What's the most common object used to run stateless workloads in Kubernetes?** A: Deployment.
10. **Q: How do you scale a Deployment?** A: `kubectl scale deployment <name> --replicas=<n>`, or by editing `spec.replicas`.

### Quiz
1. Deployments manage: (a) Nodes (b) ReplicaSets (c) Namespaces (d) etcd — **b**
2. Default update strategy is: (a) Recreate (b) RollingUpdate (c) BlueGreen (d) Canary — **b**
3. `maxUnavailable` controls: (a) Extra Pods above desired count (b) Pods allowed below desired count during rollout (c) Node count (d) Namespace quota — **b**
4. `maxSurge` controls: (a) Pods allowed below desired count (b) Extra Pods allowed above desired count (c) Node scaling (d) Volume size — **b**
5. Command to check rollout progress: (a) kubectl get rollout (b) kubectl rollout status deployment/<name> (c) kubectl deployment status (d) kubectl watch rollout — **b**
6. Editing a Deployment-managed Pod directly: (a) Persists forever (b) Is eventually reverted by the controller (c) Deletes the Deployment (d) Is required for updates — **b**
7. `revisionHistoryLimit` default is: (a) 1 (b) 5 (c) 10 (d) Unlimited — **c**
8. Deployments are the standard object for: (a) Stateful databases (b) Stateless workloads (c) Node management (d) DNS — **b**
9. Which field must match `template.metadata.labels`? (a) spec.strategy (b) spec.selector.matchLabels (c) metadata.name (d) spec.replicas — **b**
10. Command to scale a Deployment: (a) kubectl resize deployment (b) kubectl scale deployment <name> --replicas=<n> (c) kubectl deployment scale (d) kubectl set replicas — **b**

### Homework
Create a Deployment with 5 replicas, then over the course of 10 minutes: scale it up, scale it down, delete a random Pod, and update its image tag — documenting the ReplicaSet and Pod changes you observe at each step using `kubectl get rs,pods -l app=<name>`.

### Summary
Deployments are the standard controller for stateless workloads, managing ReplicaSets automatically and enabling safe rolling updates. Lab 5 dives deep into exactly how the RollingUpdate strategy works, and Lab 6 covers rolling back a bad deployment.

---

# Lab 5 — Rolling Updates

**Estimated Duration:** 55 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Trigger and observe a rolling update in detail.
2. Tune `maxUnavailable` and `maxSurge` for different risk tolerances.
3. Understand the `Recreate` strategy and when it's appropriate instead.
4. Pause and resume a rollout mid-flight.

### Prerequisites
- Completion of Lab 4.

### Theory

A **rolling update** replaces old Pods with new ones incrementally, rather than all at once, to avoid downtime. When you update a Deployment's `spec.template` (most commonly, the container image), the Deployment creates a new ReplicaSet and begins scaling it up while scaling the old ReplicaSet down, respecting the `maxUnavailable` and `maxSurge` bounds from Lab 4.

Kubernetes only considers a new Pod "available" for the purposes of a rollout once it passes its readiness probe **and** stays ready for `minReadySeconds` (default 0). This is why readiness probes are not optional in any serious Deployment — without them, Kubernetes has no reliable signal for when it's safe to continue the rollout.

The alternative strategy, `Recreate`, terminates **all** old Pods before creating any new ones. This causes downtime but guarantees old and new versions never run simultaneously — necessary for applications that cannot tolerate two versions running at once (e.g., due to an incompatible database schema migration).

You can pause a rollout mid-flight with `kubectl rollout pause`, make further template changes, then resume with `kubectl rollout resume` — useful for canary-style manual verification steps.

### Architecture Diagram

```
   Time -->

   T0: [old][old][old]                      (3 old Pods, 0 new)
   T1: [old][old][old][new]                 (maxSurge=1: 1 extra Pod allowed)
   T2: [old][old][new]                      (1 old terminated once new is Ready)
   T3: [old][new][new]
   T4: [new][new][new]                      (rollout complete)
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl set image deployment/web nginx=nginx:1.26
kubectl rollout status deployment/web
```
**Purpose:** Triggers a rolling update by changing the container image, then watches until completion.
**Syntax:** `set image deployment/<name> <container-name>=<new-image>`.
**Expected Output:**
```
Waiting for deployment "web" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "web" rollout to finish: 2 out of 3 new replicas have been updated...
deployment "web" successfully rolled out
```
**Common mistakes:** Using the wrong container name (must match `spec.containers[].name` in the Pod template exactly).
**Real-world usage:** The most common way to deploy a new application version via CI/CD.

```bash
kubectl rollout pause deployment/web
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout resume deployment/web
```
**Purpose:** Demonstrates pausing a rollout, batching further changes, then resuming as a single combined rollout.
**Real-world usage:** Manual canary verification — pause after a small percentage of Pods update, verify metrics look healthy, then resume.

```bash
kubectl get pods -l app=web -w
```
**Purpose:** Watches Pods live during a rollout to visually observe old Pods terminating as new ones become Ready.

### YAML Files

Tuning update behavior:
```yaml
spec:
  minReadySeconds: 10             # New Pod must stay Ready for 10s before being considered available
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0            # Never drop below desired count — zero-downtime guarantee
      maxSurge: 1                  # Only 1 extra Pod at a time
```

The `Recreate` strategy:
```yaml
spec:
  strategy:
    type: Recreate                # All old Pods terminated before any new ones are created; causes downtime
```

### Expected Output
```
$ kubectl rollout history deployment/web
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployment/web nginx=nginx:1.26
```

### Validation Steps
1. During the rollout, `kubectl get pods -l app=web` should never show fewer than `replicas - maxUnavailable` Pods `Running`.
2. After completion, `kubectl get pods -l app=web -o jsonpath='{.items[*].spec.containers[*].image}'` should show only the new image on every Pod.

### Common Errors
- **Rollout stuck at "X out of Y new replicas updated"** — new Pods failing readiness probe, blocking further progress.
- **Wrong container name in `kubectl set image`** — silently does nothing or errors, since the name must match the template exactly.
- **Unexpected full downtime during what should be a rolling update** — likely `strategy.type: Recreate` was set unintentionally.

### Troubleshooting
For a stuck rollout, run `kubectl describe pod <new-pod>` to see why readiness is failing (often a health check misconfiguration or a genuinely broken new image). Confirm container names with `kubectl get deployment web -o jsonpath='{.spec.template.spec.containers[*].name}'` before using `set image`. Confirm the strategy type explicitly if downtime is unexpected.

### Practice Tasks
1. Trigger a rolling update to a deliberately broken image tag and observe the rollout getting stuck partway (some old Pods remain since the new ones never become Ready).
2. Set `maxUnavailable: 0, maxSurge: 1` and trigger an update, confirming zero dip below the desired replica count.
3. Switch a Deployment to `Recreate` strategy and observe the brief full-downtime window during an update.
4. Practice `kubectl rollout pause` / `resume` around an image change.
5. Use `kubectl rollout history deployment/web` and add a `--record`-style change-cause using an annotation.

### Challenge Lab
Simulate a bad rollout: update to an image that fails its readiness probe, watch the rollout hang using `kubectl rollout status`, and practice recognizing the exact signals (Events, Pod state) that would tell an on-call engineer this needs a rollback — which is the subject of Lab 6.

### Best Practices
- Always set `maxUnavailable: 0` for user-facing services where any capacity reduction is unacceptable, accepting the tradeoff of needing more surge capacity.
- Use `Recreate` only when truly necessary (e.g., incompatible schema changes) since it causes downtime.
- Always pair rolling updates with solid readiness probes — this is the single most common cause of "rollouts don't work correctly" issues in the real world.

### Interview Questions
1. **Q: How does a rolling update avoid downtime?** A: By incrementally replacing old Pods with new ones, keeping a minimum number available throughout, governed by `maxUnavailable`/`maxSurge`.
2. **Q: What signal does Kubernetes use to know a new Pod is safe to route traffic to during a rollout?** A: The readiness probe (plus `minReadySeconds`).
3. **Q: What does the Recreate strategy do differently?** A: Terminates all old Pods before creating any new ones, causing downtime but avoiding version overlap.
4. **Q: How do you trigger a rolling update via image change?** A: `kubectl set image deployment/<name> <container>=<new-image>`.
5. **Q: How can you pause a rollout mid-flight?** A: `kubectl rollout pause deployment/<name>`.
6. **Q: Why might a rollout get stuck?** A: New Pods are failing their readiness probe, so the controller won't proceed further per the `maxUnavailable`/`maxSurge` bounds.
7. **Q: What does `minReadySeconds` do?** A: Requires a new Pod to stay Ready for that many seconds before being counted as available.
8. **Q: When would you choose Recreate over RollingUpdate?** A: When old and new versions absolutely cannot run simultaneously, e.g., incompatible database migrations.
9. **Q: How do you view rollout history?** A: `kubectl rollout history deployment/<name>`.
10. **Q: What happens if `maxUnavailable: 0`?** A: The rollout never drops below the desired replica count, relying entirely on surge Pods instead.

### Quiz
1. Rolling updates avoid downtime by: (a) Stopping all Pods at once (b) Incrementally replacing Pods (c) Ignoring readiness (d) Scaling to zero — **b**
2. Kubernetes determines Pod readiness for rollout purposes via: (a) Liveness probe only (b) Readiness probe (c) CPU usage (d) Node status — **b**
3. Recreate strategy: (a) Never causes downtime (b) Terminates all old Pods before creating new ones (c) Is the default (d) Ignores replicas — **b**
4. Command to trigger an image-based rollout: (a) kubectl update image (b) kubectl set image deployment/<name> <container>=<image> (c) kubectl rollout image (d) kubectl deploy image — **b**
5. Command to pause a rollout: (a) kubectl rollout stop (b) kubectl rollout pause deployment/<name> (c) kubectl pause (d) kubectl deployment freeze — **b**
6. `minReadySeconds` default value: (a) 10 (b) 30 (c) 0 (d) 60 — **c**
7. maxUnavailable: 0 means: (a) All Pods can be down (b) Never drop below desired replica count (c) No surge allowed (d) Rollout is disabled — **b**
8. Recreate strategy is best for: (a) Zero-downtime web apps (b) Apps that can't run two versions simultaneously (c) All apps by default (d) Stateless apps only — **b**
9. Command to view rollout history: (a) kubectl get history (b) kubectl rollout history deployment/<name> (c) kubectl history deployment (d) kubectl deployment log — **b**
10. A stuck rollout is most commonly caused by: (a) Node failure (b) New Pods failing readiness checks (c) DNS misconfiguration (d) Namespace deletion — **b**

### Homework
Perform three rolling updates in a row on the same Deployment (each to a different valid nginx tag), recording the `kubectl rollout history` output after each, and explain in your own words what each revision number represents.

### Summary
Rolling updates, governed by `maxUnavailable`/`maxSurge` and gated by readiness probes, let you deploy new versions with zero or minimal downtime. Lab 6 covers what to do when a rollout goes wrong: rolling back to a previous, known-good revision.

---

# Lab 6 — Rollbacks

**Estimated Duration:** 45 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Identify when a rollback is necessary.
2. Roll back a Deployment to the previous revision or a specific one.
3. Understand how revision history and `CHANGE-CAUSE` annotations work.
4. Practice a full "bad deploy, detect, rollback" incident cycle.

### Prerequisites
- Completion of Lab 5.

### Theory

Every rolling update creates a new revision in a Deployment's history. If a deployed version turns out to be broken — crashing, failing health checks, or causing errors — Kubernetes lets you **roll back** to a previous revision just as easily as rolling forward. Internally, a rollback is really just another rolling update, except the target template is an older, already-known-good one instead of a new one — the same `maxUnavailable`/`maxSurge` safety mechanisms from Lab 5 apply.

`kubectl rollout undo deployment/<name>` rolls back to the immediately previous revision. `kubectl rollout undo deployment/<name> --to-revision=<N>` rolls back to a specific, older revision, as shown by `kubectl rollout history`.

To make rollback history genuinely useful, it helps to record *why* each revision was created. While the old `--record` flag is deprecated, teams commonly annotate revisions manually, e.g., `kubectl annotate deployment/web kubernetes.io/change-cause="fix login bug" --overwrite` before applying a change, so `kubectl rollout history` shows meaningful `CHANGE-CAUSE` entries rather than `<none>`.

### Architecture Diagram

```
   Revision 1 (nginx:1.24)  -->  Revision 2 (nginx:1.25)  -->  Revision 3 (nginx:1.26, BROKEN)
                                                                        |
                                                             kubectl rollout undo
                                                                        |
                                                                        v
                                                        Deployment rolls back to Revision 2
                                                        (a brand-new rollout, using the old template)
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl annotate deployment/web kubernetes.io/change-cause="deploy broken image for demo" --overwrite
kubectl set image deployment/web nginx=nginx:doesnotexist
kubectl rollout status deployment/web --timeout=30s
```
**Purpose:** Simulates a bad deployment that will never become Ready, and confirms the rollout hangs/times out.
**Expected Output:** After 30 seconds: `error: timed out waiting for the condition`
**Common mistakes:** Waiting indefinitely instead of setting `--timeout`, especially in CI pipelines where a hung rollout should fail fast.
**Real-world usage:** Exactly how a CI/CD pipeline detects a failed deployment and can trigger an automatic rollback.

```bash
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
```
**Purpose:** Rolls back to the previous good revision and confirms the rollback itself completes successfully.
**Expected Output:** `deployment.apps/web rolled back`, followed by `deployment "web" successfully rolled out`.
**Real-world usage:** The standard incident-response action when a bad deploy is detected in production.

```bash
kubectl rollout history deployment/web
kubectl rollout undo deployment/web --to-revision=1
```
**Purpose:** Lists all revisions with their change-cause, then rolls back to a specific, older revision by number (not just "one back").
**Common mistakes:** Assuming revision numbers are stable forever — old revisions can be pruned per `revisionHistoryLimit` from Lab 4.
**Real-world usage:** Rolling back further than one step, e.g., when the last two revisions were both bad.

### YAML Files
Rollbacks are performed via `kubectl rollout undo`, not by writing new YAML — the Deployment controller reuses the stored template from the target revision automatically. No new manifest is required for this lab.

### Expected Output
```
$ kubectl rollout history deployment/web
REVISION  CHANGE-CAUSE
1         <none>
2         deploy broken image for demo
3         (current, rolled back to revision 1's template)
```

### Validation Steps
After `kubectl rollout undo`, confirm `kubectl get pods -l app=web -o jsonpath='{.items[*].spec.containers[*].image}'` shows the previous, known-good image again on every Pod.

### Common Errors
- **"error: no rollout history found"** — happens if `revisionHistoryLimit` was set to 0, or the Deployment was just created with no prior revisions.
- **Rolling back to a revision that no longer exists** — old revisions beyond `revisionHistoryLimit` are pruned and cannot be restored.
- **Rollback appears to "hang"** — it's actually a normal rolling update in the opposite direction; check `kubectl rollout status` for real progress.

### Troubleshooting
For missing history, check `spec.revisionHistoryLimit` on the Deployment. For pruned revisions, you cannot roll back to them — you'd need to manually reapply a saved YAML manifest of that version if you have one (a strong argument for storing all manifests in Git). For a "hanging" rollback, treat it like any other rolling update — inspect Pod readiness with `describe`.

### Practice Tasks
1. Perform three sequential updates, then roll back exactly two revisions using `--to-revision`.
2. Set `revisionHistoryLimit: 2` and observe that older revisions eventually become unavailable for rollback.
3. Practice writing meaningful `change-cause` annotations before each change.
4. Simulate a bad deploy, detect it via `kubectl rollout status --timeout`, and script the rollback command that would follow.
5. Compare the Events shown by `kubectl describe deployment web` before and after a rollback.

### Challenge Lab
Build a small bash script that: applies an image update, waits on `kubectl rollout status --timeout=20s`, and automatically runs `kubectl rollout undo` if the timeout is exceeded — a simplified version of what real CI/CD pipelines do.

### Best Practices
- Always store Deployment manifests in version control (Git) as the ultimate source of truth — never rely solely on cluster-side revision history, which can be pruned.
- Write meaningful `change-cause` annotations for every deployment so `rollout history` is actually useful during an incident.
- Set a reasonable `--timeout` on `kubectl rollout status` in CI/CD so failed deployments are detected and rolled back automatically rather than hanging indefinitely.

### Interview Questions
1. **Q: How do you roll back a Deployment to the previous revision?** A: `kubectl rollout undo deployment/<name>`.
2. **Q: How do you roll back to a specific, older revision?** A: `kubectl rollout undo deployment/<name> --to-revision=<N>`.
3. **Q: How is a rollback implemented internally?** A: As another rolling update, targeting the older, known-good template instead of a new one.
4. **Q: How do you view a Deployment's revision history?** A: `kubectl rollout history deployment/<name>`.
5. **Q: What controls how many old revisions are retained?** A: `spec.revisionHistoryLimit`.
6. **Q: What happens if you try to roll back to a pruned revision?** A: It's not possible — that revision's data is gone; you'd need a saved manifest to reapply manually.
7. **Q: How do you add a meaningful description to a revision?** A: Annotate the Deployment with `kubernetes.io/change-cause` before applying the change.
8. **Q: Why should Deployment manifests also live in Git, not just cluster history?** A: Cluster-side revision history can be pruned, but Git provides a permanent, authoritative record.
9. **Q: How can a CI/CD pipeline automatically detect a failed rollout?** A: By running `kubectl rollout status --timeout=<duration>` and checking for a non-zero exit code.
10. **Q: Does a rollback respect the same maxUnavailable/maxSurge safety bounds as a forward rollout?** A: Yes, since it's implemented as a normal rolling update in reverse.

### Quiz
1. Command to undo the last rollout: (a) kubectl undo (b) kubectl rollout undo deployment/<name> (c) kubectl rollback (d) kubectl revert deployment — **b**
2. Command to roll back to a specific revision: (a) kubectl rollout undo --revision=N (b) kubectl rollout undo deployment/<name> --to-revision=N (c) kubectl revert --rev=N (d) kubectl set revision N — **b**
3. Rollbacks are implemented as: (a) A special non-rolling operation (b) Another rolling update using an older template (c) A full cluster restart (d) Manual Pod editing — **b**
4. What controls retained revision count? (a) maxSurge (b) revisionHistoryLimit (c) maxUnavailable (d) replicas — **b**
5. Command to view revision history: (a) kubectl get history (b) kubectl rollout history deployment/<name> (c) kubectl deployment history (d) kubectl log rollout — **b**
6. Meaningful change descriptions are added via: (a) Labels (b) The kubernetes.io/change-cause annotation (c) ConfigMaps (d) Secrets — **b**
7. Can you roll back to a pruned revision? (a) Yes, always (b) No (c) Only with --force (d) Only for StatefulSets — **b**
8. CI/CD pipelines detect failed rollouts using: (a) kubectl get pods only (b) kubectl rollout status --timeout (c) Manual checking only (d) kubectl logs — **b**
9. Why keep manifests in Git in addition to cluster history? (a) No reason (b) Cluster history can be pruned; Git is permanent (c) Git is faster (d) Required by Kubernetes — **b**
10. Do rollbacks respect maxUnavailable/maxSurge? (a) No (b) Yes — **b**

### Homework
Simulate a full incident: deploy a broken image, detect the failure using `rollout status --timeout`, roll back, and write a short "postmortem" (half a page) documenting what happened, how it was detected, and how it was resolved — good practice for real on-call scenarios.

### Summary
Rollbacks provide a safety net for bad deployments, implemented as ordinary rolling updates targeting a previous revision. Combined with meaningful change-cause annotations and Git-stored manifests, this gives you a reliable, auditable deployment process. Lab 7 shifts focus to DaemonSets, a controller for a very different workload shape: one Pod per node.

---

# Lab 7 — DaemonSets

**Estimated Duration:** 45 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain what a DaemonSet guarantees and how it differs from a Deployment.
2. Deploy a DaemonSet and confirm one Pod runs per node.
3. Use node selectors/tolerations to target a subset of nodes.
4. Identify real-world use cases for DaemonSets.

### Prerequisites
- Completion of Lab 6.
- A cluster with at least 2 nodes (e.g., a Kind cluster with a worker node, from Volume 1 Lab 3).

### Theory

A **DaemonSet** ensures that exactly one copy of a Pod runs on every node in the cluster (or every node matching a given selector), automatically adding a Pod when a new node joins and removing it when a node leaves. This is fundamentally different from a Deployment, which manages a fixed **total** replica count without regard to which or how many nodes exist.

DaemonSets are the right tool whenever you need node-level infrastructure, not application-level replicas. Classic real-world examples: log collection agents (Fluent Bit, Filebeat) that must run on every node to gather that node's logs; monitoring agents (Node Exporter) that expose per-node metrics; and networking components (kube-proxy itself is deployed as a DaemonSet in many cluster distributions, and CNI plugins like Calico or Cilium run this way too).

You can restrict a DaemonSet to a subset of nodes using `nodeSelector` (matching node labels) — for example, running a GPU monitoring agent only on nodes labeled `gpu=true`. DaemonSet Pods can also be configured to tolerate the taints normally placed on control-plane nodes (Volume 5 covers taints and tolerations fully), which is how system DaemonSets manage to run even on nodes that regular workloads are excluded from.

### Architecture Diagram

```
   Node 1              Node 2              Node 3
   +----------+        +----------+        +----------+
   | DaemonSet |        | DaemonSet |        | DaemonSet |
   | Pod       |        | Pod       |        | Pod       |
   +----------+        +----------+        +----------+

   A 4th node joins the cluster -> DaemonSet automatically
   schedules a new Pod onto it, with no manual action required.
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl apply -f log-agent-daemonset.yaml
kubectl get daemonsets
kubectl get pods -l app=log-agent -o wide
```
**Purpose:** Deploys the DaemonSet and confirms one Pod landed on every node.
**Expected Output:**
```
NAME        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   AGE
log-agent   2         2         2       2            2           10s
```
**Common mistakes:** Expecting `spec.replicas` to control the count — DaemonSets have no `replicas` field at all; the count is always "one per matching node," determined automatically.
**Real-world usage:** Deploying Fluent Bit, Node Exporter, or a CNI plugin cluster-wide.

```bash
kubectl label node <node-name> gpu=true
kubectl apply -f gpu-agent-daemonset.yaml
kubectl get pods -l app=gpu-agent -o wide
```
**Purpose:** Demonstrates restricting a DaemonSet to only nodes matching a label via `nodeSelector`.
**Expected Output:** Only the labeled node shows a `gpu-agent` Pod; other nodes show none.
**Real-world usage:** Running specialized agents only where relevant hardware or roles exist.

### YAML Files

`log-agent-daemonset.yaml`:
```yaml
apiVersion: apps/v1
kind: DaemonSet             # No "replicas" field exists for DaemonSets
metadata:
  name: log-agent
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      containers:
        - name: log-agent
          image: busybox
          command: ["sh", "-c", "while true; do echo collecting logs on $(hostname); sleep 30; done"]
          resources:
            requests: { cpu: "50m", memory: "32Mi" }
            limits: { cpu: "100m", memory: "64Mi" }
```

Restricting to labeled nodes:
```yaml
spec:
  template:
    spec:
      nodeSelector:
        gpu: "true"          # Only schedule onto nodes with this label
      containers:
        - name: gpu-agent
          image: busybox
          command: ["sh", "-c", "echo monitoring GPU; sleep 3600"]
```

### Expected Output
```
NAME             READY   STATUS    RESTARTS   AGE   NODE
log-agent-4f8j2  1/1     Running   0          20s   my-cluster-control-plane
log-agent-9k2lm  1/1     Running   0          20s   my-cluster-worker
```

### Validation Steps
Confirm the number of Running Pods equals the number of nodes matching the DaemonSet's (optional) `nodeSelector`; adding a new matching node should automatically produce a new Pod without any manual `kubectl` command.

### Common Errors
- **DaemonSet Pods not scheduling onto control-plane nodes** — control-plane nodes typically have a `NoSchedule` taint; a DaemonSet needs a matching `toleration` to run there (relevant for cluster-wide agents; covered fully in Volume 5).
- **Trying to set `spec.replicas`** — this field doesn't exist on DaemonSets and will be rejected or ignored.
- **Fewer Pods than expected** — check `nodeSelector` isn't unintentionally excluding nodes.

### Troubleshooting
For missing Pods on control-plane nodes, this is often expected behavior unless the DaemonSet explicitly tolerates the control-plane taint — many system DaemonSets like `kube-proxy` include this toleration by default. Check `kubectl describe daemonset <name>` for scheduling-related Events, and `kubectl get nodes --show-labels` to verify your `nodeSelector` matches what you expect.

### Practice Tasks
1. Deploy a DaemonSet with no `nodeSelector` and confirm it lands on every node, including the control-plane node if untainted in your test cluster.
2. Add a `nodeSelector` restricting it to one specific node and confirm only that node gets a Pod.
3. Delete a DaemonSet-managed Pod manually and confirm it's automatically recreated on the same node (not a different one — this is a key DaemonSet-specific behavior).
4. Attempt to set `spec.replicas` on a DaemonSet manifest and observe the validation behavior.
5. Compare `kubectl get pods -o wide` output for a DaemonSet versus a Deployment with the same replica-equivalent count, noting the difference in node placement guarantees.

### Challenge Lab
Deploy two DaemonSets simultaneously — one with no selector (runs everywhere) and one restricted via `nodeSelector` to a labeled subset — and write the exact `kubectl get pods -o wide` commands you'd use to prove both are behaving correctly relative to your node labels.

### Best Practices
- Use DaemonSets only for genuine node-level infrastructure concerns (logging, monitoring, networking agents) — never for regular application scaling, which belongs to Deployments.
- Always set resource requests/limits on DaemonSet Pods, since they run on every node and can silently consume significant aggregate cluster resources if unconstrained.
- Combine `nodeSelector` (or the more expressive node affinity, Volume 5) with tolerations when you need agents on specialized or otherwise-restricted nodes.

### Interview Questions
1. **Q: What does a DaemonSet guarantee?** A: Exactly one Pod per node (or per matching node), automatically added/removed as nodes join/leave.
2. **Q: How does a DaemonSet differ from a Deployment?** A: A Deployment manages a fixed total replica count regardless of node count; a DaemonSet's count is tied directly to the number of matching nodes.
3. **Q: Does a DaemonSet have a `replicas` field?** A: No.
4. **Q: Give three real-world DaemonSet use cases.** A: Log collection agents, monitoring/metrics agents, and CNI/networking plugins.
5. **Q: How do you restrict a DaemonSet to specific nodes?** A: Using `nodeSelector` (or more advanced node affinity) in the Pod template.
6. **Q: Why might a DaemonSet Pod not schedule onto a control-plane node?** A: Control-plane nodes typically have a taint that repels regular Pods unless the DaemonSet includes a matching toleration.
7. **Q: If you delete a DaemonSet-managed Pod, where does its replacement get scheduled?** A: On the same node it was removed from.
8. **Q: What happens when a new node joins the cluster?** A: Matching DaemonSets automatically schedule a new Pod onto it with no manual action.
9. **Q: Is kube-proxy itself commonly deployed as a DaemonSet?** A: Yes, in many cluster distributions.
10. **Q: Why must resource limits be taken seriously on DaemonSets?** A: Because they run on every matching node, so unconstrained resource usage multiplies across the whole cluster.

### Quiz
1. A DaemonSet guarantees: (a) A fixed replica count (b) One Pod per matching node (c) Zero Pods by default (d) Random placement — **b**
2. Does a DaemonSet have a replicas field? (a) Yes (b) No — **b**
3. Example DaemonSet use case: (a) Stateless web app (b) Log collection agent (c) Batch job (d) Database primary — **b**
4. To restrict a DaemonSet to specific nodes, use: (a) replicas (b) nodeSelector (c) selector.matchLabels only (d) namespace — **b**
5. When a new node joins, a matching DaemonSet: (a) Requires manual Pod creation (b) Automatically schedules a Pod there (c) Ignores it (d) Errors out — **b**
6. Deleting a DaemonSet Pod causes its replacement to schedule: (a) On a random node (b) On the same node (c) On the control-plane only (d) Nowhere — **b**
7. Control-plane nodes often block DaemonSet Pods unless: (a) A toleration is set (b) replicas is increased (c) A Service is created (d) Nothing needed — **a**
8. kube-proxy is commonly deployed as a: (a) Deployment (b) DaemonSet (c) Job (d) StatefulSet — **b**
9. Why are resource limits especially important on DaemonSets? (a) They aren't (b) They run on every matching node, multiplying resource use (c) DaemonSets ignore limits (d) Only for Deployments — **b**
10. DaemonSets are best suited for: (a) Application scaling (b) Node-level infrastructure agents (c) Batch processing (d) Stateful databases — **b**

### Homework
Deploy a DaemonSet representing a fictional monitoring agent across your cluster, label one node to be excluded using a `nodeSelector`-based approach (i.e., select all nodes except the excluded one via a label the excluded node lacks), and document the resulting Pod placement.

### Summary
DaemonSets guarantee one Pod per matching node, ideal for node-level infrastructure like logging, monitoring, and networking agents — a fundamentally different guarantee than a Deployment's fixed total replica count. Lab 8 introduces StatefulSets, for workloads where individual Pod identity and stable storage matter.

---

# Lab 8 — StatefulSets

**Estimated Duration:** 60 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain what problems StatefulSets solve that Deployments cannot.
2. Understand stable network identity and ordered, sequential Pod creation/deletion.
3. Deploy a StatefulSet with a headless Service for stable DNS names.
4. Understand the relationship between StatefulSets and per-Pod persistent storage.

### Prerequisites
- Completion of Lab 7.
- Familiarity with Services is helpful but not required (Services are covered fully in Volume 3); this lab introduces just enough to make StatefulSets work.

### Theory

Deployments treat every Pod as interchangeable — any replica can serve any request, and if one Pod dies, a brand-new one with a new name and IP replaces it with no consequence. Some applications, most notably databases and other clustered stateful systems (PostgreSQL replicas, Kafka brokers, Elasticsearch nodes, etcd itself), need something Deployments cannot provide: **stable, predictable identity** for each Pod, plus **ordered** startup and shutdown.

A **StatefulSet** gives each Pod a stable, predictable name of the form `<statefulset-name>-<ordinal>`, starting at 0 (e.g., `db-0`, `db-1`, `db-2`), and creates/deletes them strictly in order (0, then 1, then 2 for creation; reverse order for deletion by default). Each Pod also gets a stable network identity via a **headless Service** (`clusterIP: None`), which gives each Pod its own resolvable DNS name — `db-0.db-headless.default.svc.cluster.local` — rather than sharing one load-balanced Service IP like Deployment Pods typically do.

StatefulSets are almost always paired with a `volumeClaimTemplates` field, which automatically provisions a distinct PersistentVolumeClaim for each Pod ordinal (`db-0` always reattaches to its own specific volume, even after being rescheduled) — this is covered in more depth alongside general storage concepts in Volume 4, but is introduced here since it's such a core part of what makes StatefulSets useful.

### Architecture Diagram

```
   Headless Service: db-headless (clusterIP: None)
        |
   +----------+   +----------+   +----------+
   |  db-0     |   |  db-1     |   |  db-2     |
   |  (own PVC)|   |  (own PVC)|   |  (own PVC)|
   +----------+   +----------+   +----------+
   Created in order: db-0 -> db-1 -> db-2
   Deleted in reverse order: db-2 -> db-1 -> db-0
   Each reachable at: db-N.db-headless.<namespace>.svc.cluster.local
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl apply -f headless-service.yaml
kubectl apply -f db-statefulset.yaml
kubectl get pods -l app=db -w
```
**Purpose:** Deploys the headless Service and StatefulSet, watching Pods appear strictly in order.
**Expected Output:**
```
db-0   0/1   Pending
db-0   1/1   Running
db-1   0/1   Pending    <- only starts creating after db-0 is Running
db-1   1/1   Running
db-2   0/1   Pending
db-2   1/1   Running
```
**Common mistakes:** Expecting all Pods to appear simultaneously like a Deployment — StatefulSets are intentionally sequential by default.
**Real-world usage:** Deploying any clustered database or system where node 0 (e.g., a primary/leader) must exist before others join.

```bash
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup db-1.db-headless
```
**Purpose:** Confirms each StatefulSet Pod has its own resolvable, stable DNS name via the headless Service.
**Expected Output:** Resolves to `db-1`'s specific Pod IP.
**Real-world usage:** How clustered applications discover and address specific peers by stable name, e.g., a Kafka broker referring to its peers.

```bash
kubectl delete pod db-1
kubectl get pods -l app=db -w
```
**Purpose:** Demonstrates that a deleted StatefulSet Pod is replaced with the **same name** (`db-1`), unlike a Deployment Pod which gets a brand-new name.
**Real-world usage:** Critical for stateful systems where peers reference each other by these stable names.

### YAML Files

`headless-service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None        # "Headless" - no load-balancing IP; enables per-Pod DNS instead
  selector:
    app: db
  ports:
    - port: 5432
```

`db-statefulset.yaml`:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-headless      # Must reference the headless Service above
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: postgres
          image: postgres:16
          env:
            - name: POSTGRES_PASSWORD
              value: "demo-password-not-for-prod"   # Secrets covered properly in Volume 4
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:                 # Creates a distinct PVC per Pod ordinal (db-0, db-1, db-2)
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

### Expected Output
```
NAME   READY   STATUS    RESTARTS   AGE
db-0   1/1     Running   0          40s
db-1   1/1     Running   0          25s
db-2   1/1     Running   0          10s
```

### Validation Steps
1. Confirm Pods were created in strict order by comparing their `AGE` values (db-0 oldest, db-2 newest).
2. Confirm `nslookup db-0.db-headless` and `nslookup db-1.db-headless` resolve to different, specific Pod IPs.
3. Delete `db-1` and confirm its replacement is also named `db-1`, not a new random name.

### Common Errors
- **StatefulSet stuck waiting on Pod 0** — if `db-0` never becomes Ready, `db-1` and `db-2` will never be created, since ordering is strict by default.
- **PVC not found / storage errors** — no default StorageClass configured in the cluster (StorageClasses are covered in Volume 4).
- **DNS resolution failing** — the headless Service's `clusterIP: None` was omitted, or its selector doesn't match the StatefulSet's Pod labels.

### Troubleshooting
If Pod 0 never becomes Ready, focus all debugging there first — later Pods are blocked by design. For PVC/storage errors on Kind/Minikube, confirm a default StorageClass exists with `kubectl get storageclass` (both tools usually provide one automatically). For DNS issues, verify the Service's `selector` matches `app: db` exactly and that `clusterIP: None` is present.

### Practice Tasks
1. Deploy the 3-replica StatefulSet and confirm strict ordering by watching Pod creation timestamps.
2. Delete the whole StatefulSet, then delete just its Pods (not the object) and observe self-healing versus full removal differ.
3. Resolve each Pod's individual DNS name using `nslookup` from a debug Pod.
4. Scale the StatefulSet down to 1 replica and observe Pods being removed in reverse order (`db-2` first, then `db-1`).
5. Inspect `kubectl get pvc` and confirm each Pod ordinal has its own distinct PersistentVolumeClaim that survives Pod deletion.

### Challenge Lab
Scale your 3-replica StatefulSet down to 0 and back up to 3, and confirm (via `kubectl get pvc`) that the *same* PersistentVolumeClaims are reused for `db-0`, `db-1`, and `db-2` rather than new ones being created — demonstrating StatefulSets' stable storage guarantee.

### Best Practices
- Use StatefulSets only for genuinely stateful, identity-sensitive workloads; using them for simple stateless apps adds unnecessary complexity and slower rollout behavior (sequential, not parallel).
- Always pair a StatefulSet with a headless Service using a matching `serviceName`.
- Understand that deleting a StatefulSet does **not** automatically delete its PersistentVolumeClaims by default — this is intentional, to prevent accidental data loss, but means manual cleanup is required if you truly want to remove the data.

### Interview Questions
1. **Q: What problem do StatefulSets solve that Deployments don't?** A: Providing stable, predictable Pod identity, stable per-Pod storage, and ordered startup/shutdown for stateful applications.
2. **Q: What naming pattern do StatefulSet Pods follow?** A: `<statefulset-name>-<ordinal>`, starting at 0.
3. **Q: In what order are StatefulSet Pods created and deleted by default?** A: Created in ascending order (0,1,2...); deleted in descending order.
4. **Q: What is a headless Service, and why does a StatefulSet need one?** A: A Service with `clusterIP: None`, which enables stable, individual DNS names per Pod instead of one shared load-balanced IP.
5. **Q: What does `volumeClaimTemplates` do?** A: Automatically provisions a distinct PersistentVolumeClaim per Pod ordinal, reused across Pod rescheduling.
6. **Q: If you delete a StatefulSet Pod, what happens to its replacement's name?** A: It keeps the exact same name (e.g., `db-1` stays `db-1`), unlike Deployment Pods.
7. **Q: Does deleting a StatefulSet delete its PersistentVolumeClaims?** A: No, by default, to prevent accidental data loss.
8. **Q: Give three real-world examples of StatefulSet workloads.** A: PostgreSQL/MySQL replica sets, Kafka brokers, Elasticsearch clusters (etcd itself is another example).
9. **Q: Why is Pod 0 particularly important in a StatefulSet rollout?** A: Later Pods will not be created until Pod 0 becomes Ready, since ordering is strict by default.
10. **Q: How does a Pod resolve a specific StatefulSet peer's address?** A: Via its stable DNS name: `<pod-name>.<headless-service-name>.<namespace>.svc.cluster.local`.

### Quiz
1. StatefulSet Pods are named: (a) Randomly (b) <name>-<ordinal> starting at 0 (c) Always "pod-1" (d) By UUID — **b**
2. Default creation order is: (a) Random (b) Ascending (0,1,2...) (c) Descending (d) Simultaneous — **b**
3. A headless Service has: (a) clusterIP: None (b) Two IPs (c) No selector (d) No ports — **a**
4. volumeClaimTemplates provisions: (a) One shared PVC for all Pods (b) A distinct PVC per Pod ordinal (c) No storage (d) ConfigMaps — **b**
5. Deleting a StatefulSet Pod causes its replacement to: (a) Get a new random name (b) Keep the same name (c) Not be recreated (d) Move to another StatefulSet — **b**
6. Does deleting a StatefulSet delete its PVCs by default? (a) Yes (b) No — **b**
7. Example StatefulSet workload: (a) Stateless web frontend (b) Kafka broker cluster (c) Static file server (d) CI runner — **b**
8. If Pod 0 never becomes Ready: (a) Pod 1 starts anyway (b) Later Pods are blocked from starting (c) StatefulSet deletes itself (d) Nothing happens — **b**
9. Stable per-Pod DNS format is: (a) <pod>.<namespace>.pod (b) <pod-name>.<headless-svc>.<namespace>.svc.cluster.local (c) <pod>.cluster (d) Random UUID — **b**
10. StatefulSets are best suited for: (a) Stateless apps (b) Identity-sensitive stateful applications (c) Batch jobs (d) DaemonSet replacements — **b**

### Homework
Deploy a 3-node StatefulSet, delete `db-0` specifically (not the last ordinal), and document in detail what happens to `db-1` and `db-2` during that event, plus how long `db-0`'s replacement takes to become Ready and reattach to its original volume.

### Summary
StatefulSets solve the stable-identity and ordered-lifecycle problems that Deployments intentionally don't address, making them essential for databases and other clustered stateful systems. Lab 9 shifts to a different workload shape entirely: Jobs, for run-to-completion tasks rather than long-running services.

---

# Lab 9 — Jobs

**Estimated Duration:** 45 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain how a Job differs from a Deployment or ReplicaSet.
2. Run a Job to completion and inspect its result.
3. Configure parallelism and completion counts for batch workloads.
4. Configure retry behavior and backoff limits.

### Prerequisites
- Completion of Lab 8.

### Theory

A **Job** creates one or more Pods and ensures a specified number of them **run to successful completion**, rather than running forever like a Deployment's Pods. This is the correct tool for batch processing, one-off data migrations, report generation, or any "run this task, then stop" workload.

Key fields: `spec.completions` (how many successful Pod completions are needed overall — default 1), `spec.parallelism` (how many Pods may run at once — default 1), and `spec.backoffLimit` (how many retries are allowed before the Job is marked `Failed` — default 6).

Three common Job patterns:
- **Single completion** (default): run exactly one Pod to completion.
- **Fixed completion count**: `completions: 5` — run until 5 total Pods have succeeded, useful for a fixed batch of independent work items.
- **Work queue style**: `parallelism: 3` with no fixed `completions` — multiple worker Pods pull from a shared queue until the queue is empty, then all exit (requires application-level coordination, e.g., via a message queue).

Unlike Deployment Pods, a Job's Pod uses `restartPolicy: Never` or `restartPolicy: OnFailure` (never `Always`, since the whole point is to eventually stop). Once a Job succeeds, its Pods are left in a `Completed` state by default rather than being deleted, useful for inspecting logs after the fact, though you can clean them up with `spec.ttlSecondsAfterFinished`.

### Architecture Diagram

```
   Job (completions: 3, parallelism: 2)
        |
   +--------+   +--------+          +--------+
   | Pod 1   |   | Pod 2   |   -->    | Pod 3   |   (starts once a slot frees up)
   | running |   | running |          | running |
   +--------+   +--------+          +--------+
        |             |                   |
     Completed     Completed           Completed
        \_____________|___________________/
                       |
              Job marked "Complete" once 3/3 succeeded
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl apply -f report-job.yaml
kubectl get jobs -w
```
**Purpose:** Runs a batch Job and watches it until completion.
**Expected Output:**
```
NAME     COMPLETIONS   DURATION   AGE
report   0/1           5s         5s
report   1/1           12s        12s
```
**Common mistakes:** Using `restartPolicy: Always` (the Deployment default) on a Job — this is invalid and rejected, since a Job's Pods must eventually stop.
**Real-world usage:** Database migrations, nightly report generation, one-off data backfills.

```bash
kubectl logs job/report
```
**Purpose:** Views the completed Job's output directly by referencing the Job (kubectl resolves this to its Pod automatically for single-Pod Jobs).
**Real-world usage:** Checking the result/output of a completed batch task.

```bash
kubectl delete job report
```
**Purpose:** Cleans up a completed Job and its Pods once you no longer need to inspect them.
**Common mistakes:** Forgetting completed Jobs' Pods stick around by default, silently accumulating over time without `ttlSecondsAfterFinished` or manual cleanup.

### YAML Files

`report-job.yaml`:
```yaml
apiVersion: batch/v1              # Jobs live in the "batch/v1" API group
kind: Job
metadata:
  name: report
spec:
  backoffLimit: 3                 # Retry up to 3 times before marking the Job Failed
  ttlSecondsAfterFinished: 3600   # Auto-delete the Job (and its Pods) 1 hour after completion
  template:
    spec:
      restartPolicy: OnFailure    # Required for Jobs: Never or OnFailure, never Always
      containers:
        - name: report-generator
          image: busybox
          command: ["sh", "-c", "echo generating report; sleep 5; echo report complete"]
```

Fixed completion count with parallelism:
```yaml
spec:
  completions: 5      # Total successful Pod completions required
  parallelism: 2       # At most 2 Pods running at once
  backoffLimit: 4
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: worker
          image: busybox
          command: ["sh", "-c", "echo processing item; sleep 3"]
```

### Expected Output
```
NAME     COMPLETIONS   DURATION   AGE
report   1/1           12s        12s
```

### Validation Steps
`kubectl get job report -o jsonpath='{.status.succeeded}'` should return `1` (or your configured `completions` count) once finished.

### Common Errors
- **"Job has reached the specified backoff limit"** — the Pod's command kept failing until retries were exhausted; the Job is marked `Failed`.
- **Job "hangs" forever** — `parallelism`/`completions` misconfigured, or the container never actually exits.
- **`restartPolicy: Always` rejected** — this policy is invalid for Job Pod templates.

### Troubleshooting
For a Failed Job hitting its backoff limit, run `kubectl logs job/<name>` (or target a specific failed Pod by name) to see why the command kept failing. For a hanging Job, confirm the container's command actually exits with a status code rather than running forever like a service would. Always use `OnFailure` or `Never` for `restartPolicy`.

### Practice Tasks
1. Run a Job with `completions: 5, parallelism: 2` and time how long it takes versus `parallelism: 5`.
2. Create a Job whose command deliberately fails, and watch it hit `backoffLimit` and become `Failed`.
3. Set `ttlSecondsAfterFinished: 30` and confirm the Job and its Pods are auto-deleted 30 seconds after completion.
4. Compare `kubectl get pods` output for a Job versus a Deployment — note the `Completed` status unique to Jobs.
5. Extract just the exit code of a Job's underlying Pod using `kubectl get pod <job-pod> -o jsonpath='{.status.containerStatuses[0].state.terminated.exitCode}'`.

### Challenge Lab
Design a Job that simulates processing 10 independent work items using `completions: 10, parallelism: 3`, each taking a random 1–5 second delay (`sleep $((RANDOM % 5 + 1))`), and measure the total wall-clock time versus what `parallelism: 1` would have taken.

### Best Practices
- Always set `backoffLimit` deliberately rather than relying on the default of 6, based on how expensive/risky retries are for your specific task.
- Use `ttlSecondsAfterFinished` to avoid accumulating completed Job Pods indefinitely in a busy cluster.
- Never use a Job for anything that should run continuously — that's what Deployments are for.

### Interview Questions
1. **Q: How does a Job differ from a Deployment?** A: A Job runs Pods to successful completion and then stops; a Deployment keeps Pods running indefinitely.
2. **Q: What does `spec.completions` control?** A: The total number of successful Pod completions required for the Job to be considered done.
3. **Q: What does `spec.parallelism` control?** A: How many Pods may run concurrently at once.
4. **Q: What does `backoffLimit` control?** A: How many retries are allowed before the Job is marked Failed.
5. **Q: What restartPolicy values are valid for a Job?** A: `OnFailure` or `Never` — never `Always`.
6. **Q: What API group do Jobs belong to?** A: `batch/v1`.
7. **Q: What happens to a Job's Pods after it completes?** A: They remain (in `Completed` state) by default unless `ttlSecondsAfterFinished` is set or they're manually deleted.
8. **Q: Give three real-world Job use cases.** A: Database migrations, report generation, one-off data backfills/batch processing.
9. **Q: What field automatically cleans up a finished Job?** A: `ttlSecondsAfterFinished`.
10. **Q: What happens if a Job's Pod command never exits?** A: The Job never completes; it will run indefinitely (or until manually intervened).

### Quiz
1. A Job's Pods are meant to: (a) Run forever (b) Run to successful completion, then stop (c) Never start (d) Restart continuously — **b**
2. Valid restartPolicy values for a Job: (a) Always only (b) OnFailure or Never (c) Random (d) None allowed — **b**
3. `completions` controls: (a) Parallel Pods (b) Total required successful completions (c) Retry count (d) Node count — **b**
4. `parallelism` controls: (a) Total completions (b) Concurrent Pods allowed (c) Backoff time (d) Namespace — **b**
5. `backoffLimit` controls: (a) Parallel Pods (b) Retry attempts before marking Failed (c) Completion count (d) TTL — **b**
6. Jobs belong to which API group? (a) apps/v1 (b) batch/v1 (c) core/v1 (d) networking/v1 — **b**
7. After a Job completes, its Pods: (a) Are always deleted immediately (b) Remain unless TTL/manual cleanup (c) Restart automatically (d) Turn into a Deployment — **b**
8. Field to auto-clean finished Jobs: (a) autoDelete (b) ttlSecondsAfterFinished (c) cleanupAfter (d) expireSeconds — **b**
9. A Job whose Pod command never exits: (a) Completes immediately (b) Runs indefinitely (c) Auto-fails after 1 minute (d) Converts to a Deployment — **b**
10. Best use case for a Job: (a) Long-running web server (b) One-off batch data migration (c) DNS resolution (d) Node monitoring — **b**

### Homework
Design and run a Job simulating a nightly batch report with `completions: 3, parallelism: 1, backoffLimit: 2`, intentionally make the second attempt fail once before succeeding (e.g., using a counter file in a shared volume), and document the retry behavior you observe.

### Summary
Jobs handle run-to-completion batch workloads with configurable parallelism, completion counts, and retry behavior — a fundamentally different lifecycle than the always-running Pods managed by Deployments, DaemonSets, or StatefulSets. Lab 10 introduces CronJobs, which schedule Jobs to run automatically on a recurring basis, and wraps up the volume with a combined mini project.

---

# Lab 10 — CronJobs & Mini Project

**Estimated Duration:** 90 minutes
**Difficulty Level:** Intermediate (capstone for Volume 2)

### Learning Objectives
1. Explain how CronJobs schedule Jobs on a recurring basis.
2. Write and interpret cron schedule expressions.
3. Configure concurrency policy and history limits for recurring Jobs.
4. Combine every controller from this volume into one realistic mini project.

### Prerequisites
- Completion of Labs 1–9.

### Theory

A **CronJob** creates a new Job automatically on a recurring schedule, defined using standard cron syntax: five fields representing minute, hour, day-of-month, month, and day-of-week (`* * * * *`). For example, `0 2 * * *` means "run at 2:00 AM every day," and `*/15 * * * *` means "run every 15 minutes."

Two fields control what happens if a scheduled run is still going when the next one is due: `spec.concurrencyPolicy` — `Allow` (default, runs overlap), `Forbid` (skip the new run if the previous one is still active), or `Replace` (cancel the still-running one and start the new one). Most real-world CronJobs, especially anything touching shared resources like a database, should use `Forbid` to avoid overlapping runs causing conflicts.

`spec.successfulJobsHistoryLimit` and `spec.failedJobsHistoryLimit` (defaulting to 3 and 1 respectively) control how many old Job objects are retained for inspection, similar in spirit to a Deployment's `revisionHistoryLimit`.

`spec.startingDeadlineSeconds` handles missed schedules (e.g., if the control plane was down at the scheduled time) — if the deadline is exceeded, that particular run is skipped entirely rather than run late.

This volume's capstone mini project combines a Deployment (a small web API), a DaemonSet (a per-node log agent), and a CronJob (a nightly cleanup task) into one coherent, realistic small system — bringing together nearly every controller covered in this volume.

### Architecture Diagram

```
   CronJob (schedule: "0 2 * * *")
        |
        | at 2:00 AM every day, creates a new:
        v
      Job  ---> Pod ---> runs to completion ---> Job marked Complete
        |
   old Job objects retained per successfulJobsHistoryLimit / failedJobsHistoryLimit
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl apply -f cleanup-cronjob.yaml
kubectl get cronjobs
```
**Purpose:** Deploys a CronJob and shows its schedule and last-run time.
**Expected Output:**
```
NAME      SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cleanup   */2 * * * *   False     0        <none>          5s
```
**Common mistakes:** Writing an invalid cron expression, which the API server rejects at creation time.
**Real-world usage:** Nightly backups, log rotation, expired-data cleanup, scheduled report emails.

```bash
kubectl create job --from=cronjob/cleanup manual-test-run
kubectl get jobs
```
**Purpose:** Manually triggers a CronJob's Job template immediately, useful for testing without waiting for the schedule.
**Real-world usage:** Verifying a CronJob's logic works correctly before trusting it to a live schedule.

```bash
kubectl patch cronjob cleanup -p '{"spec":{"suspend":true}}'
```
**Purpose:** Pauses future scheduled runs without deleting the CronJob definition.
**Real-world usage:** Temporarily disabling a scheduled task during a maintenance window.

### YAML Files

`cleanup-cronjob.yaml`:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup
spec:
  schedule: "*/2 * * * *"                 # Every 2 minutes, for demo purposes (production might use "0 2 * * *")
  concurrencyPolicy: Forbid                # Skip a new run if the previous one is still active
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 60              # Skip a run if it's more than 60s late
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cleanup
              image: busybox
              command: ["sh", "-c", "echo cleaning up old data; sleep 5; echo done"]
```

**Volume 2 Mini Project manifests** (`mini-project.yaml`) — a Deployment, a DaemonSet, and a CronJob together:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels: { app: api }
spec:
  replicas: 3
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      containers:
        - name: api
          image: hashicorp/http-echo
          args: ["-text=hello from api"]
          ports: [{ containerPort: 5678 }]
          resources: { requests: { cpu: "50m", memory: "32Mi" }, limits: { cpu: "100m", memory: "64Mi" } }
          readinessProbe: { httpGet: { path: /, port: 5678 }, initialDelaySeconds: 3 }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-logger
spec:
  selector: { matchLabels: { app: node-logger } }
  template:
    metadata: { labels: { app: node-logger } }
    spec:
      containers:
        - name: node-logger
          image: busybox
          command: ["sh", "-c", "while true; do echo logging on $(hostname); sleep 30; done"]
          resources: { requests: { cpu: "20m", memory: "16Mi" }, limits: { cpu: "50m", memory: "32Mi" } }
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-cleanup
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cleanup
              image: busybox
              command: ["sh", "-c", "echo running nightly cleanup; sleep 3"]
```

### Expected Output
```
NAME                       READY   STATUS    RESTARTS   AGE
pod/api-6d9f8c-abcde       1/1     Running   0          30s
pod/api-6d9f8c-fghij       1/1     Running   0          30s
pod/api-6d9f8c-klmno       1/1     Running   0          30s
pod/node-logger-x1         1/1     Running   0          30s
pod/node-logger-x2         1/1     Running   0          30s

NAME                       SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE
cronjob/nightly-cleanup    */5 * * * *   False     0        <none>
```

### Validation Steps
1. `kubectl get deployment api` shows `READY: 3/3`.
2. `kubectl get daemonset node-logger` shows one Pod per node.
3. After 5 minutes, `kubectl get jobs` shows at least one Job created by `nightly-cleanup`.
4. `kubectl get cronjob nightly-cleanup` shows a non-`<none>` `LAST SCHEDULE` after the first run.

### Common Errors
- **CronJob never triggers** — invalid schedule syntax, or the CronJob was created with `suspend: true`.
- **Overlapping cleanup runs conflicting** — `concurrencyPolicy` was left as `Allow` when `Forbid` was intended.
- **DaemonSet and Deployment Pods both scheduling but api never reachable** — a missing/misconfigured readiness probe (this is fully resolved once Services are introduced in Volume 3).

### Troubleshooting
For a CronJob that never fires, double check the five-field cron syntax and confirm `spec.suspend` is not `true`. For unwanted overlaps, set `concurrencyPolicy: Forbid`. Since this volume hasn't covered Services yet, don't worry that `api` Pods aren't reachable via a stable address — that's precisely the gap Volume 3 fills.

### Practice Tasks
1. Change the CronJob schedule to run every minute and confirm timing with `kubectl get jobs -w`.
2. Manually trigger the CronJob's Job template on demand using `kubectl create job --from=cronjob/...`.
3. Suspend and then resume the CronJob, confirming no runs occur while suspended.
4. Scale the `api` Deployment and confirm the DaemonSet's Pod count is unaffected (proving the two controllers' scaling models are entirely independent).
5. Delete a `node-logger` Pod and confirm it's replaced on the *same* node, matching DaemonSet behavior from Lab 7.

### Challenge Lab
Extend the mini project with a second CronJob that only runs on the first day of the month (cron: `0 0 1 * *`) to simulate a monthly report, and explain in writing why you would (or would not) set `concurrencyPolicy: Forbid` for that particular job versus the more frequent cleanup job.

### Best Practices
- Always test a CronJob's logic manually (`kubectl create job --from=cronjob/...`) before trusting its schedule in production.
- Default to `concurrencyPolicy: Forbid` unless you have a specific reason to allow overlapping runs.
- Keep `successfulJobsHistoryLimit`/`failedJobsHistoryLimit` reasonably low in busy clusters to avoid accumulating clutter, but high enough to debug recent failures.

### Interview Questions
1. **Q: What does a CronJob do?** A: Creates a new Job automatically on a recurring schedule defined by cron syntax.
2. **Q: What are the three concurrencyPolicy options?** A: Allow, Forbid, Replace.
3. **Q: What's the safest default concurrencyPolicy for most real-world CronJobs?** A: Forbid, to avoid overlapping runs conflicting.
4. **Q: How do you manually trigger a CronJob's Job immediately?** A: `kubectl create job --from=cronjob/<name> <new-job-name>`.
5. **Q: How do you pause a CronJob without deleting it?** A: Set `spec.suspend: true`.
6. **Q: What does `startingDeadlineSeconds` control?** A: How late a missed scheduled run can be before it's skipped entirely rather than run late.
7. **Q: What controllers were combined in this volume's mini project?** A: A Deployment, a DaemonSet, and a CronJob.
8. **Q: Why is a Deployment appropriate for the `api` component but not the `node-logger` component?** A: The API is a stateless, fixed-count workload; the logger needs exactly one Pod per node, which only a DaemonSet guarantees.
9. **Q: What fields control retained Job history for a CronJob?** A: `successfulJobsHistoryLimit` and `failedJobsHistoryLimit`.
10. **Q: Why should you test a CronJob manually before relying on its schedule?** A: To verify its logic and container behave correctly without waiting for (or risking) a live scheduled run.

### Quiz
1. A CronJob creates: (a) Pods directly (b) Jobs on a schedule (c) Deployments (d) Nodes — **b**
2. concurrencyPolicy options are: (a) Allow, Forbid, Replace (b) Yes, No (c) Sync, Async (d) Fast, Slow — **a**
3. Safest default concurrencyPolicy for most CronJobs: (a) Allow (b) Forbid (c) Replace (d) None — **b**
4. Command to manually trigger a CronJob's Job now: (a) kubectl run cronjob (b) kubectl create job --from=cronjob/<name> <name> (c) kubectl trigger (d) kubectl cronjob run — **b**
5. Field to pause a CronJob: (a) spec.pause (b) spec.suspend (c) spec.disabled (d) spec.stop — **b**
6. startingDeadlineSeconds controls: (a) Job timeout (b) How late a missed run can be before being skipped (c) Retry count (d) Parallelism — **b**
7. Which controller guarantees one Pod per node in the mini project? (a) Deployment (b) DaemonSet (c) CronJob (d) Job — **b**
8. Which controller manages the stateless `api` component? (a) DaemonSet (b) Deployment (c) CronJob (d) StatefulSet — **b**
9. Fields controlling retained CronJob history: (a) revisionHistoryLimit (b) successfulJobsHistoryLimit / failedJobsHistoryLimit (c) ttlSecondsAfterFinished only (d) None — **b**
10. Why test a CronJob manually first? (a) Not necessary (b) Verify logic works before trusting the live schedule (c) Required by Kubernetes (d) To disable it — **b**

### Homework
Build the full mini project (Deployment + DaemonSet + CronJob) from scratch without copying the YAML directly, run it for at least 10 minutes so the CronJob fires at least twice, and write a short report covering: final Pod counts per controller, CronJob run history, and one thing you'd change before calling this "production-ready" (hint: connectivity between components, covered next in Volume 3).

### Summary
CronJobs schedule recurring Jobs using cron syntax, with configurable concurrency handling and history retention. This lab's mini project combined a Deployment, DaemonSet, and CronJob into one realistic small system, exercising nearly every controller covered in Volume 2. You are now ready for Volume 3, which introduces Services, Ingress, and DNS — the networking layer that lets these components actually talk to each other and to the outside world.

---

# Volume 2 — End of Volume Materials

## Volume Review

Volume 2 moved from raw Pods (Volume 1) to the full family of Kubernetes workload controllers:

- **Lab 1–2:** Multi-container Pod patterns (sidecar, ambassador, adapter) and Init Containers for setup tasks.
- **Lab 3–4:** ReplicaSets for self-healing fixed-count Pods, and Deployments, which manage ReplicaSets and add rolling updates.
- **Lab 5–6:** The mechanics of rolling updates (`maxUnavailable`/`maxSurge`) and rollbacks (`rollout undo`).
- **Lab 7:** DaemonSets, for exactly-one-Pod-per-node infrastructure workloads.
- **Lab 8:** StatefulSets, for identity-sensitive, stateful workloads with stable storage and DNS.
- **Lab 9–10:** Jobs and CronJobs for run-to-completion and scheduled batch work, combined in a full mini project.

You should now be able to look at any workload's requirements — stateless and scalable, node-level infrastructure, stateful and identity-sensitive, or run-to-completion — and immediately identify the correct controller. Volume 3 builds the networking layer (Services, Ingress, DNS, NetworkPolicies) that lets these workloads communicate with each other and the outside world.

## Practice Exam (25 Questions)

1. Name the three multi-container Pod patterns. — Sidecar, ambassador, adapter.
2. What must be true of containers sharing a Pod? — They are tightly coupled and must be co-scheduled/scale together.
3. What does an Init Container guarantee? — It runs to completion before any main container starts.
4. How do multiple Init Containers execute? — Sequentially, in listed order.
5. What does a ReplicaSet guarantee? — A specified number of identical Pod replicas running at all times.
6. What must match between a ReplicaSet's selector and template? — The label sets must be consistent.
7. What does a Deployment add on top of a ReplicaSet? — Safe rolling updates and rollback support.
8. What is the default Deployment update strategy? — RollingUpdate.
9. What do `maxUnavailable`/`maxSurge` control? — How many Pods may be below/above the desired count during a rollout.
10. What is the Recreate strategy's tradeoff? — Causes downtime but avoids old/new version overlap.
11. How do you trigger an image-based rolling update? — `kubectl set image deployment/<name> <container>=<image>`.
12. How do you roll back to the previous revision? — `kubectl rollout undo deployment/<name>`.
13. How do you roll back to a specific revision? — `kubectl rollout undo deployment/<name> --to-revision=<N>`.
14. What does a DaemonSet guarantee? — One Pod per matching node.
15. Does a DaemonSet have a `replicas` field? — No.
16. Give a real-world DaemonSet use case. — Log collection, monitoring agents, or CNI/networking plugins.
17. What problem do StatefulSets solve? — Stable Pod identity, ordered lifecycle, and stable per-Pod storage for stateful apps.
18. What naming pattern do StatefulSet Pods follow? — `<name>-<ordinal>`, starting at 0.
19. What kind of Service does a StatefulSet require for stable DNS? — A headless Service (`clusterIP: None`).
20. What does `volumeClaimTemplates` provide? — A distinct PersistentVolumeClaim per Pod ordinal.
21. What does a Job guarantee? — A specified number of Pods run to successful completion.
22. What restartPolicy values are valid for a Job? — OnFailure or Never.
23. What does a CronJob do? — Creates Jobs automatically on a recurring cron schedule.
24. What are the three concurrencyPolicy options for a CronJob? — Allow, Forbid, Replace.
25. Which controller is correct for a stateless, horizontally-scalable web API? — Deployment.

## Practical Assessment

Without referring back to the labs:
1. Deploy a 3-replica Deployment running `nginx:1.25` with a readiness probe.
2. Trigger a rolling update to `nginx:1.26` and watch it complete successfully.
3. Roll it back to `nginx:1.25` and confirm via Pod image inspection.
4. Deploy a DaemonSet running a simple logging container across all nodes.
5. Deploy a 2-replica StatefulSet with a headless Service and confirm stable Pod naming and DNS resolution.
6. Run a Job with `completions: 3, parallelism: 1` and confirm all three complete successfully.
7. Deploy a CronJob running every 2 minutes and confirm at least one Job is created automatically.
8. Clean up every resource created in this assessment.

**Self-grading:** You should be able to complete every step from memory, correctly choosing the right controller for each requirement without hints, in under 30 minutes.

## Mini Project Reference
See Lab 10 for the full combined Deployment + DaemonSet + CronJob mini project. Use it as your reference implementation for the Practical Assessment above.

## Troubleshooting Exercises

1. **Scenario:** A Deployment rollout is stuck at "1 out of 3 new replicas updated." **Diagnosis:** `kubectl describe pod <new-pod>` — likely a failing readiness probe on the new image.
2. **Scenario:** A StatefulSet's `db-1` and `db-2` never appear. **Diagnosis:** Check `db-0`'s readiness first — StatefulSets block later ordinals until earlier ones succeed.
3. **Scenario:** A DaemonSet has fewer Pods than nodes. **Diagnosis:** Check `nodeSelector` and node taints/tolerations.
4. **Scenario:** A Job keeps retrying and eventually shows `Failed`. **Diagnosis:** `kubectl logs job/<name>` and check `backoffLimit`.
5. **Scenario:** A CronJob's `LAST SCHEDULE` stays `<none>` forever. **Diagnosis:** Check cron syntax validity and `spec.suspend`.

## Volume 2 Cheat Sheet

| Task | Command |
|---|---|
| Create a Deployment | `kubectl create deployment <name> --image=<img> --replicas=<n>` |
| Scale a Deployment | `kubectl scale deployment <name> --replicas=<n>` |
| Trigger a rolling update | `kubectl set image deployment/<name> <container>=<image>` |
| Check rollout status | `kubectl rollout status deployment/<name>` |
| View rollout history | `kubectl rollout history deployment/<name>` |
| Roll back one revision | `kubectl rollout undo deployment/<name>` |
| Roll back to specific revision | `kubectl rollout undo deployment/<name> --to-revision=<N>` |
| Pause / resume rollout | `kubectl rollout pause / resume deployment/<name>` |
| List DaemonSets | `kubectl get daemonsets` |
| List StatefulSets | `kubectl get statefulsets` |
| Trigger a CronJob now | `kubectl create job --from=cronjob/<name> <job-name>` |
| Suspend a CronJob | `kubectl patch cronjob <name> -p '{"spec":{"suspend":true}}'` |
| List Jobs | `kubectl get jobs` |

## kubectl Command Reference (Volume 2 subset)

`rollout status`, `rollout history`, `rollout undo`, `rollout pause`, `rollout resume`, `set image`, `scale`, `create deployment`, `create job`, `annotate` (for change-cause), `patch` (for suspend), plus all Volume 1 commands applied to these new resource types (`get`, `describe`, `logs -c`, `exec -c`).

## YAML Reference (Volume 2 subset)

| Object | apiVersion | Key spec fields |
|---|---|---|
| ReplicaSet | apps/v1 | replicas, selector, template |
| Deployment | apps/v1 | replicas, selector, template, strategy |
| DaemonSet | apps/v1 | selector, template, (no replicas), nodeSelector |
| StatefulSet | apps/v1 | serviceName, replicas, selector, template, volumeClaimTemplates |
| Job | batch/v1 | completions, parallelism, backoffLimit, template.spec.restartPolicy |
| CronJob | batch/v1 | schedule, concurrencyPolicy, jobTemplate |

---

*End of Volume 2 — Pods & Workloads. Continue to Volume 3: Networking & Services.*
