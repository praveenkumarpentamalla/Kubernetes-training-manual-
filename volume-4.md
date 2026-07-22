# Kubernetes Training Manual

## Volume 4 — Storage & Configuration

**A Self-Study Handbook for Beginners to Intermediate Learners**

---

### Copyright Notice

© 2026. This training manual is an original work created for self-study purposes. All commands, YAML examples, diagrams, and explanations in this document have been written from scratch. Kubernetes®, Docker®, and related marks are trademarks of their respective owners (The Linux Foundation, Docker Inc.). This manual is not affiliated with or endorsed by any of these organizations.

---

### About This Volume

Volumes 1–3 covered compute (Pods, controllers) and networking (Services, Ingress, DNS). Volume 4 covers the two remaining essentials every real application needs: **configuration** (ConfigMaps, Secrets, environment variables) and **storage** (Volumes, PersistentVolumes, PersistentVolumeClaims, StorageClasses, CSI drivers, and dynamic provisioning). You'll also learn resource requests/limits in depth, tying scheduling and reliability back to concrete numbers.

**Prerequisites for this volume:** Completion of Volumes 1–3, including comfort with Deployments, StatefulSets, and Services.

---

## Table of Contents — Volume 4

- Lab 1 — ConfigMaps
- Lab 2 — Secrets
- Lab 3 — Volumes
- Lab 4 — Persistent Volumes
- Lab 5 — Persistent Volume Claims
- Lab 6 — Storage Classes
- Lab 7 — CSI Drivers
- Lab 8 — Dynamic Provisioning
- Lab 9 — Environment Variables
- Lab 10 — Resource Limits & Mini Project
- Volume 4 Review, Exam, and Reference

---

# Lab 1 — ConfigMaps

**Estimated Duration:** 50 minutes
**Difficulty Level:** Beginner–Intermediate

### Learning Objectives
1. Explain why configuration should be separated from container images.
2. Create ConfigMaps from literals, files, and YAML manifests.
3. Consume ConfigMap data as environment variables and as mounted files.
4. Understand what happens when a ConfigMap is updated after Pods are already running.

### Prerequisites
- Completion of Volume 3.

### Theory

Hard-coding configuration values (a database hostname, a feature flag, a log level) directly into a container image is a bad practice: it means rebuilding and redeploying the image just to change a setting, and it prevents the exact same image from being reused across dev, staging, and production with different configuration. A **ConfigMap** solves this by storing non-sensitive configuration data as key-value pairs, separate from your application's container image, which Pods can then consume in one of several ways.

There are three primary ways a Pod can consume a ConfigMap: as **environment variables** (each key becomes an env var, either individually via `valueFrom.configMapKeyRef` or in bulk via `envFrom`), as **command-line arguments** (by first injecting as environment variables, then referencing them in the container's `command`/`args`), or as **mounted files** (each key becomes a filename inside a mounted volume, with the value as that file's contents) — the mounted-file approach is especially useful for entire configuration files (e.g., an `nginx.conf`) rather than individual key-value settings.

An important operational detail: environment variables injected from a ConfigMap are set **once**, at container start — updating the ConfigMap afterward does **not** update already-running containers' environment variables; the Pod must be restarted. Mounted ConfigMap **files**, by contrast, are automatically updated inside the container within roughly a minute of the ConfigMap changing (though the application itself still needs its own logic to notice and reload the changed file — Kubernetes doesn't force that).

### Architecture Diagram

```
   ConfigMap: app-config
   {
     LOG_LEVEL: "debug"
     APP_MODE: "production"
     app.properties: "key1=value1\nkey2=value2"
   }
        |
        +---> as environment variables --> container sees LOG_LEVEL=debug
        |
        +---> as a mounted volume --> /etc/config/app.properties file appears inside the container
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl create configmap app-config --from-literal=LOG_LEVEL=debug --from-literal=APP_MODE=production
kubectl get configmap app-config -o yaml
```
**Purpose:** Creates a ConfigMap imperatively from literal key-value pairs and inspects the resulting object.
**Common mistakes:** Assuming ConfigMaps are encrypted or access-restricted like Secrets — they are stored in plaintext in etcd and are not intended for sensitive data (Lab 2 covers the correct object for that).
**Real-world usage:** Quick way to inject a handful of simple settings during local development or testing.

```bash
kubectl create configmap nginx-config --from-file=nginx.conf
```
**Purpose:** Creates a ConfigMap from an entire file's contents, keyed by the filename.
**Real-world usage:** Standard way to supply a full configuration file (nginx, application YAML/properties files) to a container without baking it into the image.

```bash
kubectl apply -f app-deployment.yaml
kubectl exec -it <pod-name> -- env | grep LOG_LEVEL
kubectl exec -it <pod-name> -- cat /etc/config/app.properties
```
**Purpose:** Confirms a Pod receives ConfigMap data both as an environment variable and as a mounted file.
**Common mistakes:** Editing the ConfigMap and expecting already-running Pods' environment variables to reflect the change without a restart — they won't.

### YAML Files

`app-config.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "debug"                       # Simple key-value pairs, consumable as env vars
  APP_MODE: "production"
  app.properties: |                        # A multi-line value, consumable as a mounted file
    key1=value1
    key2=value2
```

`app-deployment.yaml` (consuming the ConfigMap both ways):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1
  selector: { matchLabels: { app: app } }
  template:
    metadata: { labels: { app: app } }
    spec:
      containers:
        - name: app
          image: busybox
          command: ["sh", "-c", "env; sleep 3600"]
          envFrom:
            - configMapRef:
                name: app-config             # Injects ALL keys as environment variables at once
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config          # Where the ConfigMap's keys appear as files
      volumes:
        - name: config-volume
          configMap:
            name: app-config
```

### Expected Output
```
$ kubectl exec -it <pod> -- env | grep LOG_LEVEL
LOG_LEVEL=debug

$ kubectl exec -it <pod> -- cat /etc/config/app.properties
key1=value1
key2=value2
```

### Validation Steps
1. `env` inside the container shows `LOG_LEVEL=debug` and `APP_MODE=production`.
2. `/etc/config/` inside the container contains a file per ConfigMap key, including `app.properties` with the exact multi-line content.

### Common Errors
- **`CreateContainerConfigError`** — the Pod references a ConfigMap (or a specific key within one) that doesn't exist.
- **Environment variable not updating after a ConfigMap edit** — expected behavior; the Pod must be restarted.
- **Mounted file not appearing** — `volumeMounts.mountPath` and `volumes.configMap.name` mismatch, or a typo in the ConfigMap name.

### Troubleshooting
For `CreateContainerConfigError`, run `kubectl describe pod <name>` and check the Events section, which names the exact missing ConfigMap/key. For stale environment variables, this is expected — restart the Pod (e.g., `kubectl rollout restart deployment/<name>`) to pick up changes. For missing mounted files, double-check the ConfigMap name referenced under `volumes` matches an actual, existing ConfigMap exactly.

### Practice Tasks
1. Create a ConfigMap with five different key-value pairs and inject all of them via `envFrom`.
2. Create a ConfigMap from a real local file and mount it into a Pod, confirming the file's exact contents appear.
3. Update a ConfigMap's data and confirm a mounted file updates within about a minute, without restarting the Pod.
4. Update the same ConfigMap and confirm an environment-variable-injected value does **not** change until the Pod is restarted.
5. Reference a single specific key (not the whole ConfigMap) as one environment variable using `valueFrom.configMapKeyRef`.

### Challenge Lab
Build a ConfigMap containing a complete, realistic `nginx.conf`, mount it into an `nginx` container at the correct path (overriding the image's default config), and confirm via `kubectl exec ... -- nginx -T` that your custom configuration is actually in effect.

### Best Practices
- Never store sensitive data (passwords, API keys, certificates) in a ConfigMap — use Secrets (Lab 2) instead.
- Prefer mounting whole configuration files via ConfigMaps for complex config, and environment variables for simple, individual settings.
- Remember environment-variable-injected ConfigMap data requires a Pod restart to pick up changes — plan rollout automation accordingly if you need config changes to take effect promptly.

### Interview Questions
1. **Q: What is a ConfigMap used for?** A: Storing non-sensitive configuration data separately from container images.
2. **Q: What are the three main ways to consume a ConfigMap in a Pod?** A: As environment variables, as command-line arguments (via env vars), or as mounted files.
3. **Q: Is data in a ConfigMap encrypted?** A: No, it's stored in plaintext; Secrets exist for sensitive data.
4. **Q: What happens to already-running Pods when you update a ConfigMap they consume as environment variables?** A: Nothing — env vars are set once at container start; the Pod must be restarted to pick up changes.
5. **Q: What happens to mounted ConfigMap files when the ConfigMap is updated?** A: They're automatically updated inside the container within roughly a minute, though the app must have its own logic to reload them.
6. **Q: How do you inject all keys from a ConfigMap as environment variables at once?** A: Using `envFrom.configMapRef`.
7. **Q: How do you inject just one specific key as an environment variable?** A: Using `env[].valueFrom.configMapKeyRef`.
8. **Q: What error occurs if a Pod references a nonexistent ConfigMap?** A: `CreateContainerConfigError`.
9. **Q: How would you supply a whole configuration file (like nginx.conf) to a container?** A: Create a ConfigMap from that file and mount it as a volume.
10. **Q: Why is separating configuration from container images considered a best practice?** A: It allows the same image to be reused across environments with different configuration, without rebuilding.

### Quiz
1. ConfigMaps store: (a) Sensitive secrets only (b) Non-sensitive configuration data (c) Container images (d) Node metadata — **b**
2. Are ConfigMaps encrypted? (a) Yes (b) No — **b**
3. Three consumption methods are: (a) Env vars, args, mounted files (b) Only env vars (c) Only volumes (d) Only annotations — **a**
4. Env-var-injected ConfigMap changes require: (a) Nothing, auto-updates (b) A Pod restart (c) A node reboot (d) Cluster restart — **b**
5. Mounted ConfigMap file changes: (a) Never update (b) Auto-update within about a minute (c) Require cluster restart (d) Require new image — **b**
6. Injecting all keys at once uses: (a) valueFrom (b) envFrom.configMapRef (c) volumeMounts only (d) annotations — **b**
7. Injecting one specific key uses: (a) envFrom (b) valueFrom.configMapKeyRef (c) volumes only (d) labels — **b**
8. Missing ConfigMap reference causes: (a) ImagePullBackOff (b) CreateContainerConfigError (c) CrashLoopBackOff (d) Pending forever silently — **b**
9. Best use for a whole config file: (a) Environment variables (b) Mounted ConfigMap volume (c) Labels (d) Annotations — **b**
10. ConfigMaps should never store: (a) Log levels (b) Passwords/API keys (c) Feature flags (d) App modes — **b**

### Homework
Create a ConfigMap-backed configuration for a fictional application with at least 8 settings (a mix of simple key-values and one multi-line file), consume it in a Pod both ways (env vars and mounted file), and document exactly which settings you'd choose each method for, and why.

### Summary
ConfigMaps separate non-sensitive configuration from container images, consumable as environment variables (static after container start) or mounted files (auto-updating). Lab 2 covers Secrets — the equivalent mechanism, purpose-built for sensitive data.

---

# Lab 2 — Secrets

**Estimated Duration:** 50 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain how Secrets differ from ConfigMaps and what protection they actually provide.
2. Create Secrets from literals, files, and YAML manifests.
3. Consume Secrets as environment variables and mounted files.
4. Understand built-in Secret types, including `docker-registry` and `tls`.

### Prerequisites
- Completion of Lab 1.

### Theory

A **Secret** is structurally almost identical to a ConfigMap — key-value pairs consumable as environment variables or mounted files — but is intended for sensitive data: passwords, API tokens, TLS certificates, and similar. It's important to understand precisely what protection Kubernetes Secrets provide by default: values are base64-**encoded**, not encrypted, in the API and in etcd unless you specifically enable encryption-at-rest for etcd (a cluster-level configuration, not something enabled by using a Secret object itself). Base64 encoding is trivially reversible by anyone with read access to the Secret — it is **not** a security control, only a way to safely represent binary data in YAML/JSON text.

Real protection comes from **RBAC** (covered fully in Volume 5), restricting *who* can read Secret objects in the first place, combined with etcd encryption-at-rest and, in serious production environments, an external secrets manager (AWS Secrets Manager, HashiCorp Vault, etc.) integrated via tools that sync those values into Kubernetes Secrets rather than storing the ultimate source of truth in Kubernetes at all.

Kubernetes provides several built-in Secret **types** beyond the generic `Opaque` (the default): `kubernetes.io/dockerconfigjson` for private container registry credentials (referenced by Pods via `imagePullSecrets`), and `kubernetes.io/tls` for a certificate/key pair (referenced by Ingress resources, as seen in Volume 3 Lab 4). Using the correct built-in type, rather than always defaulting to `Opaque`, lets Kubernetes and related tooling validate the Secret's structure automatically.

### Architecture Diagram

```
   Secret: db-credentials (type: Opaque)
   {
     username: YWRtaW4=        (base64 of "admin")
     password: cGFzc3dvcmQxMjM= (base64 of "password123")
   }
        |
        +---> as environment variables --> container sees DB_PASSWORD=password123 (decoded automatically)
        |
        +---> as a mounted volume --> /etc/secret/password file contains "password123"

   Protection comes from RBAC + etcd encryption-at-rest, NOT from base64 itself.
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl create secret generic db-credentials --from-literal=username=admin --from-literal=password='password123'
kubectl get secret db-credentials -o yaml
```
**Purpose:** Creates a generic (`Opaque`) Secret and shows its stored, base64-encoded representation.
**Expected Output:** `data: { username: YWRtaW4=, password: cGFzc3dvcmQxMjM= }`
**Common mistakes:** Thinking base64 provides real security — anyone can trivially decode it with `echo YWRtaW4= | base64 -d`.
**Real-world usage:** Standard way to inject database credentials, API keys, and similar sensitive values.

```bash
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=me@example.com
```
**Purpose:** Creates a properly-typed Secret for authenticating to a private container registry.
**Real-world usage:** Referenced via `imagePullSecrets` in a Pod spec when pulling images from a private registry.

```bash
kubectl exec -it <pod-name> -- env | grep DB_PASSWORD
```
**Purpose:** Confirms a Secret value was correctly injected and automatically decoded as a usable environment variable inside the container.
**Common mistakes:** Manually base64-decoding a value that's already been automatically decoded by Kubernetes when injected as an env var — the decoding happens for you.

### YAML Files

`db-credentials.yaml` (values must be pre-base64-encoded when written directly in a `data` field):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque                    # Generic Secret type; default if omitted
data:
  username: YWRtaW4=            # base64("admin")
  password: cGFzc3dvcmQxMjM=    # base64("password123")
```

Using `stringData` instead avoids manual encoding (Kubernetes encodes it for you on creation):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin                # Plaintext here; Kubernetes stores it base64-encoded automatically
  password: password123
```

Consuming a Secret in a Pod:
```yaml
apiVersion: v1
kind: Pod
metadata: { name: app }
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password       # References just this one key from the Secret
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secret
          readOnly: true            # Best practice: mount Secrets read-only
  volumes:
    - name: secret-volume
      secret:
        secretName: db-credentials
  imagePullSecrets:
    - name: regcred                 # References the docker-registry type Secret from above
```

### Expected Output
```
$ kubectl exec -it app -- env | grep DB_PASSWORD
DB_PASSWORD=password123

$ kubectl exec -it app -- cat /etc/secret/password
password123
```

### Validation Steps
1. `kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d` returns the expected plaintext value.
2. The Pod's environment variable and mounted file both show the correctly decoded value.

### Common Errors
- **"illegal base64 data"** — a value was written directly under `data` without actually base64-encoding it first (use `stringData` instead to avoid this class of error).
- **`ImagePullBackOff` despite correct image name** — missing or incorrect `imagePullSecrets` reference for a private registry image.
- **Secret values visible in `kubectl describe pod` output or logs** — accidentally logging environment variables that contain sensitive Secret data.

### Troubleshooting
For base64 errors, switch to `stringData` for plaintext authoring, letting Kubernetes handle encoding. For private registry pull failures, verify the `imagePullSecrets` name matches an actual `docker-registry`-type Secret in the same namespace. Be deliberate about what your application logs — never log full environment variable dumps in production if any of them might carry Secret values.

### Practice Tasks
1. Create a Secret using `stringData` and confirm the resulting `data` field is properly base64-encoded.
2. Decode a Secret's value manually using `base64 -d` to directly observe how little protection encoding alone provides.
3. Create a `docker-registry` type Secret and reference it via `imagePullSecrets` on a Pod pulling from a private registry (or simulate this conceptually if you don't have private registry access).
4. Mount a Secret as a read-only volume and confirm attempting to write to that path fails.
5. Compare `kubectl get secret <name> -o yaml` output structure with `kubectl get configmap <name> -o yaml` from Lab 1, noting the structural similarity.

### Challenge Lab
Create a `kubernetes.io/tls` type Secret from a self-signed certificate and key (generate one locally with `openssl req -x509 -newkey rsa:2048 -nodes -keyout tls.key -out tls.crt -days 365 -subj "/CN=example.com"`), then reference it in an Ingress resource's `tls` block (revisiting Volume 3 Lab 4), completing the loop between these two volumes.

### Best Practices
- Never treat base64 encoding as encryption — always pair Secrets with proper RBAC restrictions and, ideally, etcd encryption-at-rest in production.
- Always mount Secrets as `readOnly: true` volumes.
- Use the correct built-in Secret type (`kubernetes.io/dockerconfigjson`, `kubernetes.io/tls`) rather than always defaulting to `Opaque`, so tooling can validate structure automatically.
- Be careful about what gets logged — accidental Secret leakage into logs is a common, serious real-world incident category.

### Interview Questions
1. **Q: How does a Secret differ structurally from a ConfigMap?** A: It's nearly identical (key-value, same consumption methods) but intended for sensitive data.
2. **Q: Does base64 encoding provide real security for Secret values?** A: No — it's trivially reversible; it's an encoding, not encryption.
3. **Q: What actually protects Secret data in a properly secured cluster?** A: RBAC restricting read access, plus etcd encryption-at-rest.
4. **Q: What does `stringData` do differently from `data` in a Secret manifest?** A: It accepts plaintext values and lets Kubernetes handle base64 encoding automatically.
5. **Q: What Secret type is used for private container registry credentials?** A: `kubernetes.io/dockerconfigjson`, created via `kubectl create secret docker-registry`.
6. **Q: How does a Pod use a docker-registry type Secret?** A: By referencing it in `imagePullSecrets`.
7. **Q: What Secret type is used for TLS certificates?** A: `kubernetes.io/tls`, commonly referenced by Ingress resources.
8. **Q: Why should Secrets be mounted read-only?** A: To prevent accidental or malicious modification of sensitive data from within the container.
9. **Q: What's a common real-world way sensitive Secret data accidentally leaks?** A: Being logged when an application dumps its full environment variables.
10. **Q: In serious production environments, what's often used instead of storing the ultimate source of truth directly in Kubernetes Secrets?** A: An external secrets manager (e.g., AWS Secrets Manager, HashiCorp Vault) integrated to sync values in.

### Quiz
1. Are Secret values encrypted by default? (a) Yes (b) No, only base64-encoded — **b**
2. What actually protects Secrets in production? (a) Base64 alone (b) RBAC plus etcd encryption-at-rest (c) Nothing needed (d) Namespaces alone — **b**
3. stringData allows: (a) Only base64 values (b) Plaintext authoring, auto-encoded by Kubernetes (c) Encrypted values (d) Binary only — **b**
4. Secret type for private registries: (a) Opaque (b) kubernetes.io/dockerconfigjson (c) kubernetes.io/tls (d) generic — **b**
5. Secret type for TLS certs: (a) Opaque (b) kubernetes.io/tls (c) dockerconfigjson (d) generic — **b**
6. Pods reference registry Secrets via: (a) envFrom (b) imagePullSecrets (c) volumeMounts only (d) labels — **b**
7. Best practice for mounting Secrets: (a) Read-write (b) Read-only (c) No mounting allowed (d) Always as env vars only — **b**
8. A common real leakage vector: (a) DNS (b) Logging full environment variable dumps (c) Node names (d) Namespace labels — **b**
9. "illegal base64 data" error typically results from: (a) Using stringData (b) Manually writing unencoded data under the data field (c) Too many keys (d) Wrong namespace — **b**
10. Serious production setups often use: (a) Only Kubernetes Secrets as source of truth (b) An external secrets manager syncing into Kubernetes (c) No secrets at all (d) ConfigMaps for everything — **b**

### Homework
Create a full set of Secrets for a fictional application (database credentials as Opaque, registry credentials as dockerconfigjson, and a TLS certificate as kubernetes.io/tls), consume each correctly in appropriate places, and write a short paragraph explaining the real security model protecting them (or the gaps you'd need to close before calling this production-ready).

### Summary
Secrets provide the same key-value consumption model as ConfigMaps but are intended for sensitive data — though base64 encoding alone provides no real protection; RBAC and etcd encryption-at-rest do. Lab 3 moves from configuration to storage, starting with basic Volumes.

---

# Lab 3 — Volumes

**Estimated Duration:** 45 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain why container filesystems alone are insufficient for many workloads.
2. Understand the `emptyDir` volume type in depth (revisiting Volume 2).
3. Use `hostPath` volumes and understand their significant risks.
4. Distinguish ephemeral volumes from the persistent storage covered in Labs 4–8.

### Prerequisites
- Completion of Lab 2.

### Theory

By default, a container's filesystem is **ephemeral** — anything written to it is lost the moment the container restarts, even within the same Pod (a crashed and restarted container gets a fresh filesystem). A Kubernetes **Volume** is an abstraction that gives a Pod additional storage beyond the container's own filesystem, with a lifetime and behavior depending on the specific **volume type** used.

You've already used `emptyDir` in Volume 2's multi-container Pod lab — it's the simplest volume type: an empty directory created when the Pod starts, shared across all containers in that Pod, and deleted permanently when the Pod is removed (surviving individual container restarts *within* the Pod's lifetime, but not the Pod's own deletion). This is ideal for scratch space or short-lived data sharing between containers in the same Pod, but never appropriate for anything that must survive a Pod being rescheduled.

A **hostPath** volume mounts a specific path from the underlying **node's own filesystem** directly into the Pod. This ties the Pod's data to one specific physical node — if the Pod is rescheduled elsewhere, the data doesn't follow it, since it never actually left that original node's disk. `hostPath` also carries meaningful security risk, since it can expose sensitive parts of the node's filesystem to a container; it's generally reserved for specific infrastructure use cases (like a DaemonSet needing to read node-level logs or Docker socket access) rather than general application storage.

Both `emptyDir` and `hostPath` are fundamentally **not** persistent in the sense that matters for real applications — true persistence, surviving Pod deletion and rescheduling onto any node, requires PersistentVolumes and PersistentVolumeClaims, covered starting in Lab 4.

### Architecture Diagram

```
   emptyDir:                                hostPath:
   +------------------+                     +------------------+
   | Pod               |                     | Pod               |
   |  Container A ---+  |                     |  Container -----+  |
   |  Container B ---+->| shared emptyDir      |                 |->| /var/log on
   |                  |  | (deleted with Pod)  |                  |  | THIS SPECIFIC NODE
   +------------------+                     +------------------+
                                                    ^
                                       Pod rescheduled elsewhere = data left behind
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl apply -f emptydir-demo.yaml
kubectl exec -it emptydir-demo -c writer -- sh -c "echo hello > /data/test.txt"
kubectl exec -it emptydir-demo -c reader -- cat /data/test.txt
```
**Purpose:** Confirms two containers in the same Pod can share data through an `emptyDir` volume (revisiting and reinforcing Volume 2's multi-container Pod concept, now framed as a storage topic).
**Expected Output:** `hello`
**Real-world usage:** Sidecar log-shipping patterns, or short-lived scratch space for a data-processing step.

```bash
kubectl delete pod emptydir-demo
kubectl apply -f emptydir-demo.yaml
kubectl exec -it emptydir-demo -c reader -- cat /data/test.txt
```
**Purpose:** Confirms `emptyDir` data does **not** survive Pod deletion, even when the "same" Pod name is reapplied.
**Expected Output:** `cat: /data/test.txt: No such file or directory` (the file is gone, since a brand-new Pod means a brand-new, empty `emptyDir`).

```bash
kubectl apply -f hostpath-demo.yaml
kubectl exec -it hostpath-demo -- ls /host-logs
```
**Purpose:** Confirms a Pod can read from the underlying node's filesystem directly.
**Common mistakes:** Using `hostPath` for application data storage rather than its intended narrow infrastructure use cases, creating an unexpected node-affinity dependency and a security exposure.

### YAML Files

`emptydir-demo.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata: { name: emptydir-demo }
spec:
  volumes:
    - name: shared-data
      emptyDir: {}                    # Optionally: emptyDir: { sizeLimit: 100Mi } to cap its size
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - { name: shared-data, mountPath: /data }
    - name: reader
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - { name: shared-data, mountPath: /data }
```

`hostpath-demo.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata: { name: hostpath-demo }
spec:
  containers:
    - name: log-reader
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - { name: node-logs, mountPath: /host-logs, readOnly: true }   # Read-only strongly recommended
  volumes:
    - name: node-logs
      hostPath:
        path: /var/log                # Path on the NODE's own filesystem
        type: Directory                 # Validates the path already exists and is a directory
```

### Expected Output
```
$ kubectl exec -it emptydir-demo -c reader -- cat /data/test.txt
hello
```

### Validation Steps
1. Data written by one container in an `emptyDir`-sharing Pod is immediately readable by the other container.
2. Deleting and recreating the Pod results in an empty (data-free) `emptyDir` — proving it is not persistent across Pod lifetimes.
3. A `hostPath` volume correctly exposes the specified node path's contents inside the container.

### Common Errors
- **`hostPath` mount fails with "no such file or directory"** — the specified path doesn't exist on that particular node, especially likely if `type: Directory` is set and enforced.
- **Assuming `emptyDir` data survives a rolling update** — it doesn't; a new Pod (even from the same Deployment) always gets a fresh, empty `emptyDir`.
- **Using `hostPath` in a multi-node cluster and expecting consistent behavior everywhere** — the path's actual contents differ per node, since each node has its own separate filesystem.

### Troubleshooting
For `hostPath` failures, confirm the exact path exists on the specific node the Pod is scheduled onto (SSH into a Kind/Minikube node container if needed to verify). Never rely on `emptyDir` for anything needing to survive Pod replacement — that's precisely the problem Labs 4–5 solve. Remember `hostPath` behavior is inherently node-specific in a multi-node cluster; it is not a shared or portable storage mechanism.

### Practice Tasks
1. Create an `emptyDir`-based two-container Pod and confirm data sharing works as expected.
2. Set a `sizeLimit` on an `emptyDir` volume and, if possible, observe what happens when that limit is exceeded.
3. Create a `hostPath` volume reading `/etc/hostname` from the node and confirm its contents match the node's actual hostname.
4. Delete and recreate an `emptyDir`-using Pod, confirming data loss directly.
5. Explain, in writing, a legitimate real-world use case for `hostPath` versus an illegitimate one.

### Challenge Lab
Design a DaemonSet (revisiting Volume 2 Lab 7) that uses `hostPath` to read `/var/log` from every node and a sidecar container to summarize log file counts, demonstrating a legitimate, narrow, infrastructure-focused use of `hostPath`.

### Best Practices
- Use `emptyDir` only for genuinely disposable, Pod-lifetime-scoped data — scratch space, temporary caches, inter-container handoffs.
- Avoid `hostPath` for application data; reserve it for specific infrastructure/DaemonSet use cases, and always mount it read-only unless write access is truly required.
- Never assume any volume type covered in this lab provides real persistence — that requires PersistentVolumes, covered next.

### Interview Questions
1. **Q: Why can't containers rely solely on their own filesystem for important data?** A: It's ephemeral — lost on every container restart, even within the same Pod.
2. **Q: What is `emptyDir` and what is its lifetime?** A: An empty, Pod-scoped shared directory; it persists across container restarts within the Pod but is deleted permanently when the Pod itself is deleted.
3. **Q: What is `hostPath` and what's its key limitation?** A: A volume mounting a path from the node's own filesystem; it ties data to one specific node, so it doesn't follow the Pod if rescheduled elsewhere.
4. **Q: Why is `hostPath` considered risky?** A: It can expose sensitive parts of the node's filesystem to a container, and is best reserved for narrow infrastructure use cases.
5. **Q: Does `emptyDir` data survive a Deployment's rolling update?** A: No — each new Pod gets a fresh, empty `emptyDir`.
6. **Q: What real-world use case is `hostPath` legitimately suited for?** A: DaemonSets needing node-level access, e.g., reading node logs or accessing the Docker socket.
7. **Q: What field limits an `emptyDir` volume's size?** A: `sizeLimit`.
8. **Q: Do `emptyDir` and `hostPath` provide true persistence across Pod rescheduling?** A: No, neither does — that requires PersistentVolumes/PersistentVolumeClaims.
9. **Q: Why should `hostPath` volumes typically be mounted read-only?** A: To reduce the security risk of a container modifying sensitive node filesystem contents.
10. **Q: What happens to `hostPath` volume contents if a Pod is rescheduled to a different node?** A: The Pod sees that new node's version of the path, which may have entirely different (or missing) contents.

### Quiz
1. Container filesystems by default are: (a) Persistent forever (b) Ephemeral, lost on restart (c) Backed up automatically (d) Shared across all Pods — **b**
2. emptyDir data survives: (a) Pod deletion (b) Container restarts within the same Pod, but not Pod deletion (c) Nothing at all (d) Node reboot only — **b**
3. hostPath mounts: (a) A cloud disk (b) A path from the node's own filesystem (c) A ConfigMap (d) A remote NFS share automatically — **b**
4. Does hostPath data follow a rescheduled Pod? (a) Yes (b) No — **b**
5. hostPath is best reserved for: (a) General app storage (b) Narrow infrastructure use cases like DaemonSets (c) Databases (d) User uploads — **b**
6. emptyDir size can be limited via: (a) resources.limits (b) sizeLimit field (c) Not possible (d) Node config only — **b**
7. Does emptyDir survive a rolling update? (a) Yes (b) No, fresh empty volume each time — **b**
8. hostPath's key security concern: (a) None (b) Exposing sensitive node filesystem contents (c) Too slow (d) Requires internet access — **b**
9. True persistence across Pod rescheduling requires: (a) emptyDir (b) hostPath (c) PersistentVolumes/PVCs (d) ConfigMaps — **c**
10. hostPath behavior across a multi-node cluster is: (a) Identical everywhere (b) Node-specific, since each node has its own filesystem (c) Automatically synced (d) Not supported — **b**

### Homework
Design (on paper) a storage strategy for three different hypothetical workloads — a batch job needing scratch space, a DaemonSet needing node log access, and a database needing durable storage — explicitly stating which volume type (or, for the database, "none of these — needs Lab 4/5") fits each, with reasoning.

### Summary
`emptyDir` and `hostPath` cover ephemeral, Pod-scoped or node-scoped storage needs but provide no real persistence across Pod rescheduling. Lab 4 introduces PersistentVolumes, the foundation of genuinely durable Kubernetes storage.

---

# Lab 4 — Persistent Volumes

**Estimated Duration:** 50 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain what a PersistentVolume represents and how it differs from the volumes in Lab 3.
2. Understand access modes and reclaim policies.
3. Manually provision a PersistentVolume (static provisioning).
4. Understand the PersistentVolume lifecycle states.

### Prerequisites
- Completion of Lab 3.

### Theory

A **PersistentVolume (PV)** is a cluster-level resource representing a real piece of storage — provisioned from network-attached storage, a cloud disk, or (for local learning) a directory on a node — that exists **independently of any specific Pod's lifecycle**. Unlike `emptyDir` or `hostPath`, a PersistentVolume is a first-class Kubernetes object with its own lifecycle, existing before any Pod claims it and potentially continuing to exist after every Pod using it is gone.

Key PV fields: `capacity.storage` (how much space it provides), `accessModes` (`ReadWriteOnce` — one node can mount read-write; `ReadOnlyMany` — many nodes can mount read-only; `ReadWriteMany` — many nodes can mount read-write, requiring storage backends that support this, like NFS), and `persistentVolumeReclaimPolicy` (`Retain` — keep the underlying storage and data after the claim using it is deleted, requiring manual cleanup; `Delete` — automatically delete the underlying storage when the claim is released, common with dynamically-provisioned cloud disks; `Recycle` — deprecated, avoid).

A PersistentVolume moves through several **phases**: `Available` (provisioned, not yet claimed), `Bound` (successfully matched to a PersistentVolumeClaim, covered in Lab 5), `Released` (the claim was deleted, but the underlying storage hasn't been reclaimed yet, depending on the reclaim policy), and `Failed` (automatic reclamation failed).

This lab covers **static provisioning** — an administrator manually creates PV objects ahead of time, describing real, already-existing storage. Lab 8 covers **dynamic provisioning**, where PVs are created automatically on demand — the far more common approach in real cloud environments, but understanding static provisioning first makes the underlying mechanics much clearer.

### Architecture Diagram

```
   PersistentVolume: pv-data-01
   {
     capacity: 5Gi
     accessModes: [ReadWriteOnce]
     reclaimPolicy: Retain
     hostPath: /mnt/data          (for local learning; real clusters use cloud disks/NFS/etc.)
   }
   Phase: Available
        |
        | (a PersistentVolumeClaim requests matching storage - Lab 5)
        v
   Phase: Bound  --->  (claim deleted)  --->  Phase: Released  --->  (per reclaimPolicy)
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl apply -f pv-data-01.yaml
kubectl get pv
```
**Purpose:** Creates a PersistentVolume and shows its initial phase.
**Expected Output:**
```
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      AGE
pv-data-01   5Gi        RWO            Retain           Available   5s
```
**Common mistakes:** Forgetting PersistentVolumes are cluster-scoped (not namespaced, per Volume 1's namespace lesson) — you'll never see `-n <namespace>` used with `kubectl get pv`.
**Real-world usage:** Rare to hand-write in cloud environments (dynamic provisioning handles it), but essential for understanding what's happening underneath, and still used for specific on-premises/bare-metal scenarios.

```bash
kubectl describe pv pv-data-01
```
**Purpose:** Shows full PV details including current phase, claim binding (if any), and reclaim policy.
**Real-world usage:** First troubleshooting step when a PersistentVolumeClaim (Lab 5) isn't binding to an expected PV.

### YAML Files

`pv-data-01.yaml` (using `hostPath` for local learning purposes only — see the Best Practices note below):
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data-01
spec:
  capacity:
    storage: 5Gi                       # Total storage this PV provides
  accessModes:
    - ReadWriteOnce                     # One node can mount this read-write at a time
  persistentVolumeReclaimPolicy: Retain # Keep data after the claim is deleted; requires manual cleanup
  storageClassName: manual              # Used to match against PVCs requesting this same class name (Lab 5/6)
  hostPath:
    path: /mnt/data                     # For learning only; real PVs typically use cloud disks, NFS, etc.
```

A more realistic example referencing NFS (common in on-premises environments):
```yaml
apiVersion: v1
kind: PersistentVolume
metadata: { name: pv-nfs-01 }
spec:
  capacity: { storage: 10Gi }
  accessModes: [ReadWriteMany]           # NFS supports multiple nodes reading/writing simultaneously
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: nfs-server.example.com
    path: /exports/data
```

### Expected Output
```
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   AGE
pv-data-01   5Gi        RWO            Retain           Available           manual         10s
```

### Validation Steps
`kubectl get pv pv-data-01` shows `STATUS: Available` (not yet bound to any claim) with the correct capacity and access modes as specified.

### Common Errors
- **PV stuck in `Failed` phase** — automatic reclamation (with `Delete` policy) failed, often due to underlying storage backend issues.
- **PV never transitions to `Bound`** — no PersistentVolumeClaim has requested matching capacity/access modes/storageClassName yet (Lab 5).
- **`hostPath`-backed PV data lost after node issues** — expected limitation; `hostPath` PVs share all the node-specific risks from Lab 3, only wrapped in the PV/PVC abstraction.

### Troubleshooting
For a `Failed` PV, check `kubectl describe pv <name>` for the specific reclamation error, which often points to a storage backend problem outside Kubernetes itself. For a PV that never binds, this is expected until Lab 5's PVC is created with genuinely matching requirements — capacity, access mode, and (if specified) storage class must all align. Remember `hostPath`-backed PVs inherit all of Lab 3's node-specific limitations; they are a learning tool, not a production pattern.

### Practice Tasks
1. Create three PersistentVolumes with different capacities and observe them all sitting `Available`.
2. Create a PV with `accessModes: [ReadOnlyMany]` and explain, in writing, what workload shape would actually need this mode.
3. Compare `Retain` versus `Delete` reclaim policy behavior conceptually — what manual step does `Retain` require that `Delete` doesn't?
4. Delete a PV that's still `Available` (not yet bound) and confirm this succeeds cleanly.
5. Read through `kubectl explain persistentvolume.spec` to discover at least two fields not covered in this lab's theory section.

### Challenge Lab
Design a complete static-provisioning setup for a fictional on-premises cluster: three PersistentVolumes of different sizes representing different storage tiers (fast SSD-backed, standard, and archive), each with an appropriate `storageClassName` label distinguishing them, ready for Lab 5's PersistentVolumeClaims to request the correct tier.

### Best Practices
- Reserve `hostPath`-backed PersistentVolumes strictly for local learning; real static provisioning uses actual cloud disks, NFS, or other genuine network/cloud storage backends.
- Choose `Retain` reclaim policy for anything containing genuinely important data where accidental deletion would be costly; reserve `Delete` for cheaply-recreatable or already-backed-up data.
- Remember PersistentVolumes are cluster-scoped, not namespaced — plan naming conventions accordingly, since names must be unique cluster-wide.

### Interview Questions
1. **Q: What is a PersistentVolume?** A: A cluster-level resource representing real storage, existing independently of any specific Pod's lifecycle.
2. **Q: Are PersistentVolumes namespaced?** A: No, they are cluster-scoped.
3. **Q: What are the three access modes?** A: ReadWriteOnce, ReadOnlyMany, ReadWriteMany.
4. **Q: What does `Retain` reclaim policy do?** A: Keeps the underlying storage and data after the claim is deleted, requiring manual cleanup.
5. **Q: What does `Delete` reclaim policy do?** A: Automatically deletes the underlying storage when the claim using it is released.
6. **Q: What phases does a PersistentVolume move through?** A: Available, Bound, Released, and (on failure) Failed.
7. **Q: What is static provisioning?** A: An administrator manually creating PersistentVolume objects ahead of time for already-existing storage.
8. **Q: Why is `hostPath` inappropriate for real production PersistentVolumes?** A: It ties data to one specific node, same limitations as covered for raw hostPath volumes in Lab 3.
9. **Q: What access mode would a shared file system used by many Pods on different nodes need?** A: ReadWriteMany.
10. **Q: What connects a PersistentVolume to an actual Pod's storage request?** A: A PersistentVolumeClaim, covered in Lab 5.

### Quiz
1. Are PersistentVolumes namespaced? (a) Yes (b) No — **b**
2. Access mode allowing one node read-write: (a) ReadOnlyMany (b) ReadWriteOnce (c) ReadWriteMany (d) None — **b**
3. Reclaim policy keeping data after claim deletion: (a) Delete (b) Retain (c) Recycle (d) Ignore — **b**
4. Reclaim policy auto-deleting storage on release: (a) Retain (b) Delete (c) Keep (d) Preserve — **b**
5. PV phases include: (a) Available, Bound, Released, Failed (b) Running, Stopped (c) Pending, Active (d) Open, Closed — **a**
6. Static provisioning means: (a) Automatic PV creation (b) Manual PV creation by an administrator (c) No PVs needed (d) Only cloud-based — **b**
7. hostPath-backed PVs are appropriate for: (a) Production databases (b) Local learning only (c) Multi-node sharing (d) All use cases — **b**
8. ReadWriteMany is typically used with: (a) hostPath only (b) Storage backends like NFS that support it (c) Never usable (d) Only ConfigMaps — **b**
9. What connects a PV to a Pod's actual storage request? (a) Nothing needed (b) A PersistentVolumeClaim (c) A Service (d) An Ingress — **b**
10. A PV in Failed phase usually indicates: (a) Success (b) A reclamation error with the storage backend (c) Normal initial state (d) It's bound — **b**

### Homework
Design and create three PersistentVolumes representing different storage tiers for a fictional application (fast/standard/archive), using distinct `storageClassName` values and appropriate reclaim policies for each tier, and write a short justification for each policy choice.

### Summary
PersistentVolumes represent real, cluster-level storage resources with their own independent lifecycle, access modes, and reclaim policies. Lab 5 covers PersistentVolumeClaims — how Pods actually request and bind to this storage.

---

# Lab 5 — Persistent Volume Claims

**Estimated Duration:** 50 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain how a PersistentVolumeClaim requests and binds to a PersistentVolume.
2. Create a PVC and mount it into a Pod.
3. Understand the binding matching rules (capacity, access mode, storage class).
4. Observe what happens to a PVC-backed Pod when it's rescheduled.

### Prerequisites
- Completion of Lab 4.

### Theory

A **PersistentVolumeClaim (PVC)** is how an application actually requests storage — it's a namespaced object (unlike the cluster-scoped PV) representing a **request** for storage with specific requirements: how much space, which access mode(s), and optionally which storage class. Kubernetes' PV controller watches for new PVCs and attempts to **bind** each one to a suitable, currently-`Available` PersistentVolume that satisfies the request (capacity at least as large as requested, matching access mode, and matching `storageClassName` if specified).

Once bound, that PV/PVC pair is exclusively matched — even if a "better fitting" PV becomes available later, an already-bound PVC stays bound to its original PV. A Pod then references the **PVC** (never the PV directly) in its `volumes` block, exactly as you'd reference a ConfigMap or Secret — this indirection is exactly what makes storage portable across different environments: the same Pod spec, referencing the same PVC name, works whether that PVC happens to bind to a `hostPath`-backed learning PV or a real cloud disk in production.

This is precisely the mechanism StatefulSets used automatically in Volume 2 Lab 8's `volumeClaimTemplates` — each StatefulSet Pod ordinal gets its own PVC created and bound for it automatically, which is why `db-0` always reattaches to the same underlying storage even after being rescheduled, while a Deployment's Pods (which don't use PVCs by default) have no such persistence guarantee at all.

### Architecture Diagram

```
   PersistentVolumeClaim: data-claim
   {
     request: 3Gi, ReadWriteOnce, storageClassName: manual
   }
        |
        | PV controller searches for a matching, Available PV
        v
   PersistentVolume: pv-data-01 (5Gi, RWO, manual)  --- matches, binds ---
        |
   Pod references "data-claim" (never the PV directly)
        |
   Pod rescheduled to a different node --> PVC stays bound to the SAME PV
   --> data persists correctly (assuming the storage backend supports it)
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl apply -f data-claim.yaml
kubectl get pvc data-claim
kubectl get pv pv-data-01
```
**Purpose:** Creates a PVC and confirms it successfully binds to the matching PV from Lab 4.
**Expected Output:**
```
NAME          STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-claim    Bound    pv-data-01   5Gi        RWO            manual         5s
```
**Common mistakes:** Requesting a `storageClassName` that doesn't exactly match any Available PV's, resulting in the PVC staying `Pending` indefinitely.
**Real-world usage:** The standard, everyday way applications request storage — you almost always work with PVCs, rarely PVs directly.

```bash
kubectl apply -f app-with-storage.yaml
kubectl exec -it app-with-storage -- sh -c "echo persisted-data > /data/file.txt"
kubectl delete pod app-with-storage
kubectl apply -f app-with-storage.yaml
kubectl exec -it app-with-storage -- cat /data/file.txt
```
**Purpose:** Proves genuine persistence — unlike Lab 3's `emptyDir`, data written before Pod deletion is still present after the Pod is recreated, because the PVC (and its bound PV) survived independently of the Pod's lifecycle.
**Expected Output:** `persisted-data`

### YAML Files

`data-claim.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-claim
spec:
  accessModes:
    - ReadWriteOnce                 # Must be satisfiable by at least one Available PV
  resources:
    requests:
      storage: 3Gi                  # PV must offer AT LEAST this much capacity
  storageClassName: manual          # Must match a PV's storageClassName exactly (or be dynamically provisioned - Lab 8)
```

`app-with-storage.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata: { name: app-with-storage }
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-claim         # References the PVC by name, never the PV directly
```

### Expected Output
```
NAME          STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-claim    Bound    pv-data-01   5Gi        RWO            manual         30s
```

### Validation Steps
1. `kubectl get pvc data-claim` shows `STATUS: Bound`.
2. Data written to the mounted path survives Pod deletion and recreation, unlike Lab 3's `emptyDir` behavior.

### Common Errors
- **PVC stuck `Pending`** — no Available PV satisfies the requested capacity/access mode/storageClassName combination.
- **PVC bound to an unexpectedly large PV** — Kubernetes binds to the smallest satisfying match by default, but any PV meeting or exceeding the request is eligible; always review actual bindings rather than assuming an exact size match.
- **Attempting to delete a PV that's still Bound** — the deletion is blocked/deferred (behavior depends on finalizers) until the PVC releases it.

### Troubleshooting
For a `Pending` PVC, run `kubectl describe pvc <name>` and check Events for the specific reason (usually "no persistent volumes available for this claim"), then verify your PV's capacity/access mode/storageClassName against exactly what the PVC requested. Always double check `storageClassName` spelling — it must match exactly, with no fuzzy matching.

### Practice Tasks
1. Create a PVC requesting more storage than any existing PV offers, and observe it stay `Pending`.
2. Create a PVC with the correct `storageClassName` and confirm successful binding.
3. Write data to a PVC-backed Pod, delete the Pod, recreate it, and confirm data persistence — the core proof-of-concept for this entire lab.
4. Attempt to delete a still-Bound PV and observe Kubernetes' behavior.
5. Create two PVCs simultaneously competing for a single matching PV, and observe which one wins the binding (and that the other stays `Pending`).

### Challenge Lab
Build a small StatefulSet (revisiting Volume 2 Lab 8) using `volumeClaimTemplates`, and after Pods are running, manually inspect the automatically-created PVCs with `kubectl get pvc`, confirming each Pod ordinal has its own distinct, individually-bound claim — connecting this lab's manual PVC workflow to what StatefulSets automate.

### Best Practices
- Applications should always reference PVCs, never PersistentVolumes directly, to keep Pod specs portable across environments.
- Request only the storage capacity you actually need; over-requesting wastes potentially expensive backing storage, especially in dynamically-provisioned cloud environments (Lab 8).
- Always verify a PVC reaches `Bound` status before assuming a workload depending on it will function correctly — a `Pending` PVC leaves any dependent Pod stuck `Pending` too.

### Interview Questions
1. **Q: What is a PersistentVolumeClaim?** A: A namespaced request for storage with specific capacity, access mode, and optionally storage class requirements.
2. **Q: Is a PVC namespaced or cluster-scoped?** A: Namespaced (unlike PersistentVolumes, which are cluster-scoped).
3. **Q: How does Kubernetes decide which PV a PVC binds to?** A: It finds an Available PV satisfying the requested capacity, access mode, and matching storageClassName.
4. **Q: Does a Pod reference a PV or a PVC?** A: A PVC — never the PV directly.
5. **Q: What happens to a PVC-PV binding once established?** A: It's exclusive and persists, even if a better-fitting PV later becomes available.
6. **Q: Why does referencing a PVC (rather than a PV) make Pod specs portable?** A: The same Pod spec works across environments regardless of what specific storage backend the PVC happens to bind to.
7. **Q: What happens if no PV satisfies a PVC's request?** A: The PVC remains `Pending` indefinitely (in a static-provisioning-only setup).
8. **Q: How did StatefulSets in Volume 2 use PVCs automatically?** A: Via `volumeClaimTemplates`, creating and binding one PVC per Pod ordinal automatically.
9. **Q: Does data written to a PVC-backed volume survive Pod deletion?** A: Yes, since the PVC (and its bound PV) exist independently of any specific Pod.
10. **Q: What field in a PVC spec must match a PV's field exactly for binding to succeed?** A: `storageClassName` (along with satisfying capacity and access mode requirements).

### Quiz
1. Is a PVC namespaced? (a) No (b) Yes — **b**
2. Pods reference: (a) The PV directly (b) The PVC (c) Neither (d) The StorageClass directly — **b**
3. A PVC binds to a PV based on: (a) Random selection (b) Matching capacity, access mode, storageClassName (c) Alphabetical order (d) Node proximity only — **b**
4. If no PV satisfies a PVC: (a) It errors immediately (b) It stays Pending (c) It auto-creates one (without dynamic provisioning) (d) It binds anyway — **b**
5. Does data survive Pod deletion with a PVC-backed volume? (a) No (b) Yes — **b**
6. PVC-PV bindings, once established: (a) Change freely (b) Are exclusive and persist (c) Reset daily (d) Are random — **b**
7. StatefulSets use PVCs via: (a) Manual creation only (b) volumeClaimTemplates (c) ConfigMaps (d) Not at all — **b**
8. Why reference PVCs instead of PVs directly? (a) No reason (b) Keeps Pod specs portable across environments (c) PVs can't be referenced (d) Faster performance — **b**
9. A field that must match exactly for binding: (a) Pod name (b) storageClassName (c) Node name (d) Namespace — **b**
10. Best practice for requested storage size: (a) Always request maximum possible (b) Request only what's actually needed (c) Doesn't matter (d) Always request 1Gi — **b**

### Homework
Build a complete static-provisioning workflow from scratch: two PVs of different sizes, two PVCs with different requirements, observe which binds to which, then deploy a Pod using one of the PVCs and prove data persistence across a Pod deletion/recreation cycle — documenting every step and its output.

### Summary
PersistentVolumeClaims are how applications actually request and consume storage, binding to matching PersistentVolumes while keeping Pod specs portable and storage-backend-agnostic. Lab 6 introduces StorageClasses, which enable this entire binding process to happen automatically rather than requiring pre-created PVs.

---

# Lab 6 — Storage Classes

**Estimated Duration:** 45 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain what a StorageClass represents and how it enables dynamic provisioning.
2. Inspect the default StorageClass on Kind/Minikube.
3. Create a custom StorageClass with specific parameters.
4. Understand `volumeBindingMode` and its scheduling implications.

### Prerequisites
- Completion of Lab 5.

### Theory

A **StorageClass** describes a "class" or "type" of storage available in the cluster — for example, "fast SSD-backed storage" versus "cheap archival storage" — and, critically, references a **provisioner**: the plugin responsible for actually creating new storage on demand when a PVC requests that class, rather than requiring an administrator to manually pre-create PVs as in Labs 4–5. This is the foundation of **dynamic provisioning**, covered fully in Lab 7's CSI drivers and Lab 8's dynamic provisioning workflow — but understanding the StorageClass object itself is the necessary first step.

When a PVC specifies a `storageClassName` that matches an existing StorageClass (rather than a manually pre-created PV), and no existing PV satisfies it, the named StorageClass's provisioner automatically creates a brand-new, appropriately-sized PV to satisfy that specific claim. If a PVC omits `storageClassName` entirely, Kubernetes uses whichever StorageClass is marked as the cluster's **default** (similar in spirit to Volume 3 Lab 5's default IngressClass concept).

Key StorageClass fields beyond the provisioner: `parameters` (provisioner-specific settings, e.g., disk type or replication factor, varying entirely by which provisioner you're using), `reclaimPolicy` (same meaning as on a PV, but as the default applied to every PV this class dynamically creates), and `volumeBindingMode` — `Immediate` (provision the volume as soon as the PVC is created) versus `WaitForFirstConsumer` (delay provisioning until a Pod actually needs it, letting the scheduler consider storage topology, such as availability zone, when placing that Pod — generally the better default in real cloud environments).

### Architecture Diagram

```
   StorageClass: fast-ssd
   {
     provisioner: kubernetes.io/aws-ebs (example)
     parameters: { type: gp3 }
     volumeBindingMode: WaitForFirstConsumer
   }
        |
   PVC requests storageClassName: fast-ssd, no matching PV exists
        |
   Provisioner automatically creates a brand-new PV matching the request
        |
   New PV binds to the PVC automatically -- no administrator action required
```

### Installation
No new installation — Kind and Minikube both ship with a working default StorageClass out of the box.

### Hands-on Practice

```bash
kubectl get storageclass
```
**Purpose:** Lists all StorageClasses and shows which one (if any) is marked default.
**Expected Output (Kind):**
```
NAME                 PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE      DEFAULT
standard (default)   rancher.io/local-path  Delete          WaitForFirstConsumer   true
```
**Common mistakes:** Assuming no StorageClass exists in a fresh local cluster — both Kind and Minikube provide one automatically, unlike a truly bare-metal cluster which would need one manually configured.
**Real-world usage:** First check when confirming what dynamic provisioning options exist in any given cluster.

```bash
kubectl describe storageclass standard
```
**Purpose:** Shows full StorageClass details including its provisioner and parameters.
**Real-world usage:** Understanding exactly what "default" storage behavior a cluster provides before relying on it.

```bash
kubectl apply -f fast-storageclass.yaml
kubectl get storageclass
```
**Purpose:** Creates an additional, custom StorageClass alongside the existing default.
**Real-world usage:** Offering multiple storage tiers (fast/standard/archive) within one cluster for different application needs.

### YAML Files

`fast-storageclass.yaml` (illustrative; actual `provisioner` and `parameters` values depend entirely on your specific cluster/cloud):
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: rancher.io/local-path      # The plugin responsible for actually creating storage (varies by cluster)
reclaimPolicy: Delete                    # Default reclaim policy for PVs this class creates
volumeBindingMode: WaitForFirstConsumer  # Delay provisioning until a Pod actually needs the volume
allowVolumeExpansion: true               # Allows PVCs of this class to be resized later, if the provisioner supports it
```

A PVC requesting this custom class:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: fast-data-claim }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast-ssd            # No PV needs to be manually pre-created for this to work
  resources:
    requests: { storage: 10Gi }
```

### Expected Output
```
NAME                 PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE      DEFAULT
fast-ssd             rancher.io/local-path  Delete          WaitForFirstConsumer
standard (default)   rancher.io/local-path  Delete          WaitForFirstConsumer   true
```

### Validation Steps
1. `kubectl get storageclass` lists both the default and your custom class.
2. A PVC referencing the custom `storageClassName`, with no manually pre-created matching PV, still reaches `Bound` — proof dynamic provisioning is functioning.

### Common Errors
- **PVC referencing a nonexistent StorageClass** — stays `Pending` forever, since no provisioner exists to fulfill it.
- **Confusing `WaitForFirstConsumer` for a stuck/broken PVC** — this is expected behavior; the PVC intentionally stays unbound (though not `Pending` in an error sense) until a Pod actually references it.
- **Assuming every cluster has a default StorageClass** — true for Kind/Minikube, but not guaranteed on a genuinely bare-metal or from-scratch cluster.

### Troubleshooting
For a PVC referencing a missing StorageClass, run `kubectl get storageclass` to confirm the exact name exists (spelling matters). For `WaitForFirstConsumer` confusion, remember this is intentional — check whether a Pod referencing the PVC actually exists yet; the PVC won't fully provision/bind until it does. Never assume default StorageClass availability on unfamiliar or newly-built clusters without checking first.

### Practice Tasks
1. List and describe your cluster's default StorageClass, identifying its provisioner.
2. Create a second, custom StorageClass and a PVC using it, confirming dynamic provisioning works without any manually-created PV.
3. Compare `volumeBindingMode: Immediate` versus `WaitForFirstConsumer` by creating two otherwise-identical StorageClasses differing only in this field, and observing PVC binding timing differences.
4. Mark your custom StorageClass as the cluster default and confirm a PVC omitting `storageClassName` now uses it automatically.
5. Attempt to reference a nonexistent StorageClass name in a PVC and observe the resulting stuck-`Pending` behavior.

### Challenge Lab
Design a three-tier StorageClass scheme (`fast-ssd`, `standard`, `archive`) for a fictional cluster, each with different `reclaimPolicy` and `volumeBindingMode` combinations appropriate to its use case, and write a short justification for each configuration choice.

### Best Practices
- Explicitly set `storageClassName` on every PVC in production rather than relying on an implicit default, for clarity and to avoid surprises if the cluster's default later changes.
- Prefer `volumeBindingMode: WaitForFirstConsumer` in multi-zone cloud clusters, so storage is provisioned in the same availability zone as the Pod that will actually use it.
- Document which StorageClass maps to which real-world storage tier/cost, since this is easy to forget months later when provisioning new workloads.

### Interview Questions
1. **Q: What does a StorageClass represent?** A: A named "class" or type of storage, referencing a provisioner responsible for creating new storage on demand.
2. **Q: What problem does a StorageClass's provisioner solve?** A: It enables dynamic provisioning, automatically creating PVs on demand rather than requiring manual pre-creation.
3. **Q: What happens if a PVC's `storageClassName` is omitted?** A: Kubernetes uses whichever StorageClass is marked as the cluster's default.
4. **Q: What does `volumeBindingMode: Immediate` do?** A: Provisions the volume as soon as the PVC is created.
5. **Q: What does `volumeBindingMode: WaitForFirstConsumer` do, and why is it often preferred?** A: Delays provisioning until a Pod needs the volume, letting the scheduler account for storage topology like availability zone.
6. **Q: Do Kind and Minikube ship with a default StorageClass?** A: Yes, both provide one automatically out of the box.
7. **Q: What happens if a PVC references a StorageClass that doesn't exist?** A: The PVC stays `Pending` indefinitely, since no provisioner can fulfill it.
8. **Q: What does `allowVolumeExpansion: true` enable?** A: Letting PVCs of that class be resized later, if the underlying provisioner supports it.
9. **Q: What determines the reclaim policy of a dynamically-created PV?** A: The `reclaimPolicy` specified on the StorageClass that provisioned it.
10. **Q: Why might a cluster offer multiple StorageClasses?** A: To provide different storage tiers (e.g., fast/standard/archive) suited to different application needs and cost tradeoffs.

### Quiz
1. A StorageClass primarily references: (a) A specific PV (b) A provisioner (c) A Pod (d) A Namespace — **b**
2. If storageClassName is omitted on a PVC: (a) It errors (b) The cluster's default StorageClass is used (c) It stays unbound forever (d) A random one is picked — **b**
3. WaitForFirstConsumer: (a) Provisions immediately (b) Delays provisioning until a Pod needs it (c) Never provisions (d) Requires manual PV creation — **b**
4. Do Kind/Minikube ship a default StorageClass? (a) No (b) Yes — **b**
5. A PVC referencing a nonexistent StorageClass: (a) Errors immediately (b) Stays Pending indefinitely (c) Auto-creates the class (d) Binds anyway — **b**
6. allowVolumeExpansion enables: (a) Deleting PVCs (b) Resizing PVCs later, if supported (c) Faster provisioning (d) Multiple bindings — **b**
7. A dynamically-created PV's reclaim policy comes from: (a) The Pod spec (b) The StorageClass that provisioned it (c) Random default (d) The PVC only — **b**
8. Multiple StorageClasses are useful for: (a) No real reason (b) Offering different storage tiers/costs (c) Only for testing (d) Required by Kubernetes — **b**
9. Immediate binding mode: (a) Waits for a Pod (b) Provisions as soon as the PVC is created (c) Never provisions (d) Requires two PVCs — **b**
10. StorageClasses enable: (a) Only static provisioning (b) Dynamic provisioning (c) Neither (d) Only ConfigMaps — **b**

### Homework
Inspect your own cluster's default StorageClass in full detail, create at least two additional custom StorageClasses with meaningfully different configurations, and write a one-page storage tiering strategy document for a fictional small company's Kubernetes cluster.

### Summary
StorageClasses describe available storage tiers and, via their provisioner, enable automatic, on-demand creation of PersistentVolumes — eliminating the manual PV pre-creation from Labs 4–5. Lab 7 goes one level deeper into CSI drivers, the actual plugin mechanism that implements provisioners.

---

# Lab 7 — CSI Drivers

**Estimated Duration:** 45 minutes
**Difficulty Level:** Intermediate–Advanced

### Learning Objectives
1. Explain what the Container Storage Interface (CSI) is and why it exists.
2. Identify the components that make up a typical CSI driver deployment.
3. Inspect installed CSI drivers in a cluster.
4. Understand the relationship between CSI drivers, provisioners, and StorageClasses.

### Prerequisites
- Completion of Lab 6.

### Theory

Before CSI, storage vendor integrations were built directly into Kubernetes' own codebase ("in-tree" plugins) — meaning every new storage backend required a Kubernetes core code change and release cycle, tightly coupling storage vendors to the Kubernetes release schedule. The **Container Storage Interface (CSI)** is a standardized API that decouples this entirely: any storage vendor (cloud providers, NFS, Ceph, and dozens of others) can implement a CSI driver as an independent, out-of-tree plugin that works with any CSI-compatible container orchestrator, not just Kubernetes.

A typical CSI driver deployment consists of two main pieces: a **controller** component (often a Deployment, since it doesn't need to run on every node) handling cluster-level operations like provisioning and deleting volumes, and a **node** component (a DaemonSet, since every node needs it — directly connecting to Volume 2 Lab 7's DaemonSet concept) handling node-level operations like actually mounting a provisioned volume into a Pod.

The connection back to Lab 6: a StorageClass's `provisioner` field names a specific installed CSI driver (e.g., `ebs.csi.aws.com` for AWS's official EBS CSI driver, `disk.csi.azure.com` for Azure). Installing a new CSI driver into a cluster is what makes new `provisioner` values available for StorageClasses to reference — the driver is the actual implementation; the StorageClass is just configuration pointing at it.

### Architecture Diagram

```
   CSI Driver: ebs.csi.aws.com
   +---------------------------------------------+
   |  Controller (Deployment)                       |
   |   - handles CreateVolume / DeleteVolume calls    |
   |                                               |
   |  Node plugin (DaemonSet, one Pod per node)       |
   |   - handles mounting the volume onto the node    |
   +---------------------------------------------+
                    ^
                    | referenced by
   StorageClass: fast-ssd { provisioner: ebs.csi.aws.com }
```

### Installation
No new installation for local learning (Kind/Minikube's default `rancher.io/local-path` provisioner is already a CSI-adjacent local-path implementation). For reference, a cloud CSI driver installation typically looks like:
```bash
# Example only — actual command/manifest depends entirely on the specific cloud/driver
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/deploy/kubernetes/base/
```

### Hands-on Practice

```bash
kubectl get csidrivers
```
**Purpose:** Lists all CSI drivers currently registered in the cluster.
**Expected Output (varies significantly by cluster):**
```
NAME                    ATTACHREQUIRED   PODINFOONMOUNT   AGE
rancher.io/local-path   false            false            2d
```
**Common mistakes:** Expecting to see major cloud provider CSI drivers (EBS, Azure Disk, etc.) on a local Kind/Minikube cluster — those are cloud-specific and won't appear here.
**Real-world usage:** Confirming exactly which storage backends a given cluster can actually provision against.

```bash
kubectl get pods -n kube-system | grep -i csi
```
**Purpose:** Locates the controller and node Pods implementing a given CSI driver (naming/namespace conventions vary by driver and installation method).
**Real-world usage:** First troubleshooting step for provisioning failures — confirming the driver's own Pods are healthy before investigating the specific PVC/StorageClass further.

```bash
kubectl describe pvc <name>
```
**Purpose:** Revisits PVC troubleshooting from Lab 5, now with CSI-specific awareness — Events here will reference the specific CSI driver name if a provisioning attempt fails at the driver level.

### YAML Files
CSI drivers are typically installed via vendor-provided manifests rather than authored from scratch; no new hand-written YAML is introduced in this lab. For reference, the `CSIDriver` object itself (usually created automatically by the driver's own installation manifest) looks like:
```yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: ebs.csi.aws.com          # This exact name is what a StorageClass's "provisioner" field references
spec:
  attachRequired: true
  podInfoOnMount: true
```

### Expected Output
```
NAME                    ATTACHREQUIRED   PODINFOONMOUNT   AGE
rancher.io/local-path   false            false            5d
```

### Validation Steps
`kubectl get csidrivers` returns at least one entry, and its name matches exactly what one or more StorageClasses reference in their `provisioner` field (confirmable via `kubectl get storageclass -o jsonpath='{.items[*].provisioner}'`).

### Common Errors
- **StorageClass references a provisioner with no matching installed CSI driver** — PVCs using that class stay `Pending` forever, since nothing can actually fulfill provisioning requests.
- **CSI controller Pod crashing** — often cloud credential/permission misconfiguration, preventing the driver from authenticating to the actual cloud storage API.
- **CSI node plugin missing on some nodes** — since it's typically a DaemonSet, a Pod scheduled onto a node lacking a healthy node-plugin instance may fail to mount an otherwise-successfully-provisioned volume.

### Troubleshooting
For provisioner mismatches, cross-reference `kubectl get csidrivers` against every StorageClass's `provisioner` field to confirm they align. For a crashing CSI controller, check its logs for authentication/permission errors against the underlying cloud API — this is a cloud-IAM problem, not a Kubernetes-YAML problem, in the overwhelming majority of cases. For missing node plugins, confirm the DaemonSet (Volume 2 Lab 7 concepts apply directly) has a healthy Pod on every relevant node.

### Practice Tasks
1. List your cluster's installed CSI drivers and record their exact names.
2. Cross-reference every StorageClass's `provisioner` field against the installed CSI driver list, confirming they all match.
3. Research (via official documentation) what components make up one real cloud CSI driver (e.g., AWS EBS CSI driver) beyond what's covered in this lab's theory.
4. Explain, in writing, why CSI decoupled storage vendors from Kubernetes' own release cycle, and why that mattered for the ecosystem.
5. Identify which Kubernetes component from Volume 1's architecture lab (the scheduler, kubelet, etc.) most directly interacts with a CSI node plugin during Pod startup.

### Challenge Lab
Research and write a short comparison (half a page) of three different real-world CSI drivers (e.g., AWS EBS, Azure Disk, and one for a non-cloud backend like Ceph or NFS), covering what access modes each supports and what its typical use case is.

### Best Practices
- Always confirm a CSI driver is genuinely healthy (both controller and node components) before troubleshooting StorageClass or PVC configuration — a broken driver explains every downstream symptom.
- When adopting a new CSI driver in production, review its exact supported access modes and features (snapshots, resizing, cloning) since these vary significantly between drivers.
- Keep CSI driver versions reasonably current, since storage vendors frequently ship important bug fixes and new capabilities.

### Interview Questions
1. **Q: What is the Container Storage Interface (CSI)?** A: A standardized API letting storage vendors implement drivers as independent, out-of-tree plugins usable by any CSI-compatible orchestrator.
2. **Q: What problem did CSI solve compared to the old "in-tree" plugin model?** A: It decoupled storage vendor code from Kubernetes' own codebase and release cycle.
3. **Q: What are the two main components of a typical CSI driver deployment?** A: A controller component (cluster-level operations) and a node component (per-node mounting operations).
4. **Q: Why is the CSI node component typically a DaemonSet?** A: Because every node needs it to handle mounting volumes locally.
5. **Q: How does a StorageClass reference a specific CSI driver?** A: Via its `provisioner` field, matching the driver's registered name exactly.
6. **Q: What command lists installed CSI drivers in a cluster?** A: `kubectl get csidrivers`.
7. **Q: What typically causes a CSI controller Pod to crash?** A: Cloud credential or permission misconfiguration preventing authentication to the underlying storage API.
8. **Q: If a StorageClass references a provisioner with no matching CSI driver installed, what happens to PVCs using it?** A: They stay Pending indefinitely.
9. **Q: Do all CSI drivers support the same access modes and features?** A: No, this varies significantly between drivers and must be checked per-driver.
10. **Q: What's the relationship between a CSIDriver object, a StorageClass, and a PVC?** A: The CSIDriver registers an available driver; the StorageClass references it via `provisioner`; the PVC requests a StorageClass by name.

### Quiz
1. CSI stands for: (a) Cluster Storage Index (b) Container Storage Interface (c) Cloud Storage Integration (d) Core Storage Infrastructure — **b**
2. CSI decoupled storage vendors from: (a) DNS (b) Kubernetes' own release cycle (c) Networking (d) RBAC — **b**
3. CSI driver's controller component is typically: (a) A DaemonSet (b) A Deployment (c) A Job (d) A CronJob — **b**
4. CSI driver's node component is typically: (a) A Deployment (b) A DaemonSet (c) A StatefulSet (d) A Job — **b**
5. StorageClass references a CSI driver via: (a) metadata.name (b) provisioner field (c) accessModes (d) reclaimPolicy — **b**
6. Command to list installed CSI drivers: (a) kubectl get drivers (b) kubectl get csidrivers (c) kubectl describe storage (d) kubectl get plugins — **b**
7. A crashing CSI controller is often caused by: (a) DNS issues (b) Cloud credential/permission problems (c) Too many Pods (d) Node labels — **b**
8. If a provisioner has no matching CSI driver installed: (a) PVCs bind anyway (b) PVCs stay Pending indefinitely (c) An error is thrown immediately (d) A default is used — **b**
9. Do all CSI drivers support identical features? (a) Yes (b) No, varies significantly by driver — **b**
10. The CSIDriver object relates to StorageClass by: (a) No relation (b) Being referenced via the provisioner field (c) Replacing it entirely (d) Only used for Secrets — **b**

### Homework
Research three real CSI drivers used by major cloud providers (AWS, GCP, Azure) or a popular self-hosted option (Longhorn, Ceph-CSI), and write a comparison table covering: provisioner name, supported access modes, and one standout feature (snapshots, resizing, encryption) each offers.

### Summary
CSI drivers are the actual plugin implementations behind StorageClass provisioners, decoupling storage vendor code from Kubernetes core and consisting of controller and per-node components. Lab 8 ties Labs 6–7 together into the complete, end-to-end dynamic provisioning workflow.

---

# Lab 8 — Dynamic Provisioning

**Estimated Duration:** 50 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Walk through the complete dynamic provisioning workflow end-to-end.
2. Provision storage for a StatefulSet dynamically, without any manually-created PV.
3. Resize a PVC using volume expansion.
4. Compare static versus dynamic provisioning tradeoffs directly.

### Prerequisites
- Completion of Lab 7.

### Theory

This lab connects everything from Labs 4–7 into the complete, real-world-standard workflow: **dynamic provisioning**. Rather than an administrator manually pre-creating PersistentVolumes (Lab 4's static provisioning), a PVC simply requests a StorageClass (Lab 6), whose CSI driver (Lab 7) automatically creates a brand-new, precisely-sized PersistentVolume on demand, binding it to that PVC — with zero manual PV authoring required. This is how essentially all real cloud-based Kubernetes storage works in practice; static provisioning (Labs 4–5) remains valuable primarily for understanding the underlying mechanics and for specific on-premises/legacy scenarios.

**Volume expansion** lets you increase (never decrease) an existing PVC's size after creation, provided the StorageClass has `allowVolumeExpansion: true` (from Lab 6) and the underlying CSI driver supports it. You simply edit the PVC's `resources.requests.storage` to a larger value — Kubernetes and the CSI driver handle the rest, though depending on the driver and filesystem, the consuming Pod may need to be restarted for the filesystem itself to actually recognize the new size.

Side-by-side comparison: **static provisioning** gives an administrator precise, manual control over exactly what storage exists, useful for constrained on-premises environments with a fixed pool of real disks; **dynamic provisioning** trades that manual control for massive operational convenience and scalability, letting developers self-service exactly the storage they need without waiting on an administrator — the clear default choice for cloud-native applications and the approach StatefulSets (Volume 2 Lab 8) assume by default via `volumeClaimTemplates`.

### Architecture Diagram

```
   Static provisioning (Labs 4-5):           Dynamic provisioning (Lab 8):
   Admin creates PV manually                  PVC requests a StorageClass
        |                                            |
   PVC requests matching PV                   CSI driver auto-creates a new,
        |                                     precisely-sized PV on demand
   Manual match required                             |
                                              PV binds to PVC automatically
                                              -- zero manual PV authoring
```

### Installation
No new installation — this lab exercises Kind/Minikube's existing default StorageClass and CSI-adjacent provisioner from Labs 6–7.

### Hands-on Practice

```bash
kubectl apply -f dynamic-claim.yaml
kubectl get pvc dynamic-claim
kubectl get pv
```
**Purpose:** Creates a PVC with no manually pre-created matching PV, and confirms a new PV is automatically created and bound.
**Expected Output:**
```
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
dynamic-claim    Bound    pvc-8f3a2b1c-...(auto-generated name)       2Gi        RWO            standard       5s
```
**Common mistakes:** Looking for a PV you created yourself — with dynamic provisioning, the PV's name is auto-generated and you'll never have manually written its YAML.
**Real-world usage:** The default, everyday storage request pattern for virtually all cloud-native Kubernetes applications.

```bash
kubectl apply -f statefulset-dynamic.yaml
kubectl get pvc -l app=db-dynamic
```
**Purpose:** Confirms a StatefulSet's `volumeClaimTemplates` (Volume 2 Lab 8) automatically triggers dynamic provisioning for each Pod ordinal, with zero manual PV creation.
**Real-world usage:** How virtually every real-world StatefulSet (databases, message queues) obtains its storage in practice.

```bash
kubectl patch pvc dynamic-claim -p '{"spec":{"resources":{"requests":{"storage":"5Gi"}}}}'
kubectl get pvc dynamic-claim
```
**Purpose:** Expands an existing PVC's size in place.
**Common mistakes:** Attempting to shrink a PVC — Kubernetes only supports growing volumes, never shrinking them.
**Real-world usage:** Handling an application whose storage needs grew beyond its original provisioning without any data migration.

### YAML Files

`dynamic-claim.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: dynamic-claim }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: standard      # References an existing StorageClass; NO pre-created PV needed
  resources:
    requests:
      storage: 2Gi
```

`statefulset-dynamic.yaml` (revisiting Volume 2 Lab 8's `volumeClaimTemplates`, now understood fully as dynamic provisioning):
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: db-dynamic }
spec:
  serviceName: db-dynamic-headless
  replicas: 2
  selector: { matchLabels: { app: db-dynamic } }
  template:
    metadata: { labels: { app: db-dynamic } }
    spec:
      containers:
        - name: db
          image: busybox
          command: ["sh", "-c", "sleep 3600"]
          volumeMounts:
            - { name: data, mountPath: /var/lib/data }
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: standard    # Dynamic provisioning: no manual PV needed for db-dynamic-0 or db-dynamic-1
        resources:
          requests: { storage: 1Gi }
```

### Expected Output
```
NAME                    STATUS   VOLUME                                    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-db-dynamic-0       Bound    pvc-a1b2c3d4-...                          1Gi        RWO            standard       20s
data-db-dynamic-1       Bound    pvc-e5f6g7h8-...                          1Gi        RWO            standard       15s
```

### Validation Steps
1. `kubectl get pv` shows auto-generated PVs (long, randomly-generated names) with no corresponding hand-written YAML anywhere.
2. Each StatefulSet Pod ordinal has its own distinct, automatically-provisioned and bound PVC.
3. After patching a PVC's storage request upward, `kubectl get pvc` eventually reflects the new, larger capacity.

### Common Errors
- **PVC stays `Pending` even with dynamic provisioning available** — the referenced StorageClass name doesn't exist, or its CSI driver (Lab 7) isn't healthy.
- **Volume expansion silently has no effect** — the StorageClass lacks `allowVolumeExpansion: true`, or the specific CSI driver doesn't support resizing.
- **Application inside the Pod still sees the old, smaller size after expansion** — the filesystem itself often needs the Pod restarted to recognize the change, depending on the driver.

### Troubleshooting
For a `Pending` PVC despite dynamic provisioning, revisit Lab 6/7 troubleshooting — confirm the StorageClass exists and its CSI driver is healthy. For expansion having no visible effect, check `allowVolumeExpansion` on the StorageClass first; not all drivers support resizing at all. For a Pod not seeing the new size, try restarting it — many drivers require this final step to complete the filesystem-level resize.

### Practice Tasks
1. Create a dynamically-provisioned PVC and confirm no manual PV creation was needed at any point.
2. Deploy a small StatefulSet using `volumeClaimTemplates` and confirm each Pod ordinal gets its own dynamically-provisioned PVC.
3. Expand a PVC's storage request and confirm the change is eventually reflected.
4. Attempt to shrink a PVC and observe Kubernetes' rejection of the request.
5. Compare the total hands-on effort of this lab's dynamic provisioning workflow against Labs 4–5's static provisioning workflow, and write a short reflection on the tradeoffs.

### Challenge Lab
Design a complete storage strategy document (as if for a real team) for a fictional application needing both a StatefulSet database (dynamically provisioned, `fast-ssd` class) and a batch Job needing large, cheap scratch storage (dynamically provisioned, `archive` or `standard` class), including the exact StorageClass and PVC/volumeClaimTemplate YAML for each.

### Best Practices
- Default to dynamic provisioning for essentially all new cloud-native applications; reserve static provisioning for specific legacy or tightly-constrained on-premises scenarios.
- Always verify `allowVolumeExpansion` support before promising "storage can grow later" to application teams relying on your StorageClass.
- Plan for the Pod restart that volume expansion often requires — it's rarely a fully zero-downtime operation depending on the driver.

### Interview Questions
1. **Q: What is dynamic provisioning?** A: Automatically creating a new PersistentVolume on demand when a PVC requests a StorageClass, with no manual PV authoring required.
2. **Q: What are the three components (from Labs 6–8) that together implement dynamic provisioning?** A: A StorageClass (Lab 6), its referenced CSI driver (Lab 7), and the PVC requesting it (Lab 8).
3. **Q: How do StatefulSets leverage dynamic provisioning?** A: Via `volumeClaimTemplates`, automatically creating one dynamically-provisioned PVC per Pod ordinal.
4. **Q: What field enables volume expansion on a StorageClass?** A: `allowVolumeExpansion: true`.
5. **Q: Can a PVC be shrunk?** A: No, Kubernetes only supports growing volumes, never shrinking them.
6. **Q: Why might a Pod need to be restarted after a PVC expansion?** A: The underlying filesystem often needs this final step to recognize and use the newly available space, depending on the CSI driver.
7. **Q: What's the primary real-world tradeoff of dynamic versus static provisioning?** A: Dynamic trades precise manual administrator control for massive self-service convenience and scalability.
8. **Q: What's the auto-generated PV's naming convention typically like under dynamic provisioning?** A: A long, randomly-generated name (e.g., `pvc-8f3a2b1c-...`), unlike a hand-chosen static PV name.
9. **Q: Is dynamic or static provisioning the default assumption for most real-world cloud-native applications?** A: Dynamic provisioning.
10. **Q: What happens if a PVC requests dynamic provisioning from a StorageClass whose CSI driver is unhealthy?** A: The PVC stays `Pending`, since the driver can't fulfill the provisioning request.

### Quiz
1. Dynamic provisioning eliminates the need for: (a) StorageClasses (b) Manual PV pre-creation (c) PVCs (d) CSI drivers — **b**
2. Which three Lab concepts combine to implement dynamic provisioning? (a) Pods, Services, Ingress (b) StorageClass, CSI driver, PVC (c) ConfigMap, Secret, Volume (d) Node, kubelet, kube-proxy — **b**
3. StatefulSets leverage dynamic provisioning via: (a) Manual PVs (b) volumeClaimTemplates (c) hostPath (d) emptyDir — **b**
4. Volume expansion is enabled by: (a) reclaimPolicy (b) allowVolumeExpansion: true on the StorageClass (c) Nothing needed (d) Pod restarts alone — **b**
5. Can PVCs be shrunk? (a) Yes (b) No — **b**
6. After expansion, a Pod may need: (a) Nothing further (b) A restart for the filesystem to recognize new space (c) A new PVC (d) A new StorageClass — **b**
7. Dynamic provisioning's key tradeoff versus static: (a) Less convenient (b) Trades manual control for self-service convenience (c) No difference (d) Requires more YAML — **b**
8. Auto-generated dynamic PV names are typically: (a) Chosen by the admin (b) Long, randomly-generated (c) Always "pv-1" (d) The same as the PVC name — **b**
9. Which is the default assumption for most cloud-native apps? (a) Static provisioning (b) Dynamic provisioning (c) No provisioning (d) Manual only — **b**
10. A Pending PVC under dynamic provisioning usually means: (a) Success (b) StorageClass/CSI driver issue (c) Normal behavior always (d) PVC is bound — **b**

### Homework
Build a complete dynamic-provisioning-based storage setup for a fictional two-component application (one Deployment with a single dynamically-provisioned PVC, one StatefulSet using volumeClaimTemplates), perform a volume expansion on one of them, and document every command and observed state change from start to finish.

### Summary
Dynamic provisioning — combining StorageClasses, CSI drivers, and PVCs — is the standard, real-world way Kubernetes storage is provisioned, eliminating manual PV creation and enabling self-service scalability, including volume expansion. Lab 9 shifts back to configuration, covering environment variables in depth as the final piece connecting ConfigMaps/Secrets to running containers.

---

# Lab 9 — Environment Variables

**Estimated Duration:** 40 minutes
**Difficulty Level:** Beginner–Intermediate

### Learning Objectives
1. Set static environment variables directly in a Pod spec.
2. Reference ConfigMap and Secret values as environment variables (consolidating Labs 1–2).
3. Use the Downward API to expose Pod metadata as environment variables.
4. Understand environment variable precedence and common pitfalls.

### Prerequisites
- Completion of Lab 8.

### Theory

This lab consolidates everything you've learned about getting configuration into a running container, focused specifically on environment variables as the delivery mechanism. There are four distinct sources: **static values** written directly in the Pod spec (`env[].value`), **ConfigMap references** (Lab 1's `valueFrom.configMapKeyRef` or bulk `envFrom.configMapRef`), **Secret references** (Lab 2's `valueFrom.secretKeyRef` or bulk `envFrom.secretRef`), and the **Downward API** — a mechanism for exposing information about the Pod itself (its name, namespace, IP, labels, or resource limits) as environment variables, without needing to hardcode or separately look up that information.

The Downward API is particularly useful for applications that need to be aware of their own runtime context — for example, including the Pod's own name in structured log output for easier correlation during debugging, or an application self-limiting its internal thread pool based on its own configured CPU limit rather than a separately-configured, potentially-inconsistent value.

A subtle but important precedence rule: if you use both `envFrom` (bulk injection) and an individual `env[]` entry with the *same* variable name, the individual `env[]` entry always wins — this lets you bulk-import a ConfigMap's settings while still selectively overriding one or two specific values inline, without needing a second, near-duplicate ConfigMap.

### Architecture Diagram

```
   Pod's final environment variables, assembled from up to 4 sources:

   1. env[].value            (static, hardcoded in the Pod spec)
   2. env[].valueFrom         (ConfigMap or Secret, one key at a time)
   3. envFrom                 (ConfigMap or Secret, ALL keys at once)
   4. Downward API             (Pod's own name, namespace, IP, labels, resource limits)

   If a name collision occurs between envFrom and an individual env[] entry,
   the individual env[] entry wins.
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl apply -f full-env-demo.yaml
kubectl exec -it full-env-demo -- env
```
**Purpose:** Deploys a Pod combining all four environment variable sources and shows the fully assembled result.
**Common mistakes:** Expecting a `valueFrom.fieldRef` (Downward API) entry to show real-time updates if the referenced value later changes — most Downward API env var references are, like other env vars, fixed at container start.
**Real-world usage:** Almost every real application combines at least two or three of these sources in practice.

```bash
kubectl exec -it full-env-demo -- sh -c 'echo "My pod is $POD_NAME in namespace $POD_NAMESPACE"'
```
**Purpose:** Confirms Downward-API-sourced environment variables are usable exactly like any other environment variable inside the container.
**Real-world usage:** Structured application logs that include the Pod's own identity for easier cross-referencing during incident debugging.

### YAML Files

`full-env-demo.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: full-env-demo
  labels: { app: full-env-demo, tier: demo }
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      env:
        - name: STATIC_VALUE
          value: "hello-world"                    # 1. Static, hardcoded value
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef: { name: app-config, key: LOG_LEVEL }   # 2. One specific ConfigMap key
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef: { name: db-credentials, key: password }   # 2. One specific Secret key
        - name: POD_NAME
          valueFrom:
            fieldRef: { fieldPath: metadata.name }        # 4. Downward API: the Pod's own name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef: { fieldPath: metadata.namespace }    # 4. Downward API: the Pod's own namespace
        - name: POD_IP
          valueFrom:
            fieldRef: { fieldPath: status.podIP }           # 4. Downward API: the Pod's own IP
        - name: CPU_LIMIT
          valueFrom:
            resourceFieldRef: { containerName: app, resource: limits.cpu }  # 4. Downward API: this container's own CPU limit
      envFrom:
        - configMapRef: { name: app-config }               # 3. ALL keys from app-config at once
      resources:
        requests: { cpu: "100m", memory: "64Mi" }
        limits: { cpu: "250m", memory: "128Mi" }
```

### Expected Output
```
STATIC_VALUE=hello-world
LOG_LEVEL=debug
DB_PASSWORD=password123
POD_NAME=full-env-demo
POD_NAMESPACE=default
POD_IP=10.244.1.12
CPU_LIMIT=1                         # Downward API rounds fractional CPU limits up to the nearest whole core by default
APP_MODE=production                 # From envFrom, bulk-injected from app-config
```

### Validation Steps
`kubectl exec -it full-env-demo -- env` shows all expected variables from all four sources correctly assembled, with no missing or unexpectedly-overridden values.

### Common Errors
- **`CreateContainerConfigError`** — same root cause as Labs 1–2, a referenced ConfigMap/Secret key doesn't exist.
- **Downward API `resourceFieldRef` shows an unexpected integer instead of the exact configured limit** — CPU resource fields are rounded up to the nearest whole core unless a `divisor` is explicitly specified.
- **Expected override doesn't take effect** — double-check the precedence rule; an `envFrom`-sourced value only loses to an individual `env[]` entry with the exact same name, not the reverse.

### Troubleshooting
For `CreateContainerConfigError`, revisit the Lab 1/2 troubleshooting steps — check `kubectl describe pod` Events for the exact missing key. For unexpected Downward API CPU values, add an explicit `divisor` field (e.g., `divisor: "1m"` for millicore precision) if whole-core rounding isn't what you want. For override confusion, re-verify which direction the precedence rule actually applies.

### Practice Tasks
1. Build a Pod combining static values, one ConfigMap key, one Secret key, and at least two Downward API fields.
2. Deliberately create a naming collision between an `envFrom`-bulk-injected key and an individual `env[]` entry, and confirm which one wins.
3. Add a `resourceFieldRef` for memory limits and observe its exact reported value/units.
4. Reference a Pod label via the Downward API as an environment variable.
5. Reference a nonexistent ConfigMap key and observe the resulting error precisely.

### Challenge Lab
Build a small application (using busybox and a shell script for simplicity) that logs a structured startup message including its own Pod name, namespace, IP, and CPU limit — all sourced via the Downward API — simulating a realistic production logging pattern.

### Best Practices
- Prefer the Downward API over hardcoding Pod metadata that could easily become stale or inconsistent (e.g., manually setting a "pod name" environment variable that could drift from the Pod's actual name).
- Use `envFrom` for bulk configuration import, reserving individual `env[]` entries for the specific values you need to override or that need Downward API/Secret sourcing.
- Be explicit about `divisor` when using `resourceFieldRef` for CPU/memory, to avoid surprising unit-rounding behavior.

### Interview Questions
1. **Q: What are the four sources of environment variables covered in this lab?** A: Static values, ConfigMap/Secret references (individual or bulk), and the Downward API.
2. **Q: What does the Downward API expose?** A: Information about the Pod itself — name, namespace, IP, labels, annotations, and resource limits/requests — as environment variables (or mounted files).
3. **Q: What's a real-world use case for Downward API environment variables?** A: Including the Pod's own name in structured application logs for easier debugging correlation.
4. **Q: What happens if `envFrom` and an individual `env[]` entry share the same variable name?** A: The individual `env[]` entry wins.
5. **Q: How do you bulk-inject every key from a ConfigMap as environment variables?** A: Using `envFrom.configMapRef`.
6. **Q: What field lets a container see its own configured CPU/memory limit as an environment variable?** A: `valueFrom.resourceFieldRef`.
7. **Q: Why might a `resourceFieldRef` for CPU show a rounded whole number unexpectedly?** A: CPU resource fields round up to the nearest whole core by default unless an explicit `divisor` is set.
8. **Q: Do environment variables sourced from the Downward API update automatically if the underlying Pod field changes?** A: No, like other environment variables, they're generally fixed at container start.
9. **Q: What error occurs if an environment variable references a nonexistent ConfigMap/Secret key?** A: `CreateContainerConfigError`.
10. **Q: Why prefer the Downward API over manually hardcoding Pod metadata?** A: It avoids drift/staleness, since the value is always accurate to the actual, current Pod.

### Quiz
1. Four environment variable sources are: (a) Static, ConfigMap, Secret, Downward API (b) Only static (c) Only Secrets (d) Only ConfigMaps — **a**
2. Downward API exposes: (a) External API data (b) The Pod's own metadata/resource info (c) Node hardware specs (d) Cluster-wide statistics — **b**
3. If envFrom and env[] share a name: (a) envFrom wins (b) env[] wins (c) Error occurs (d) Both are ignored — **b**
4. Bulk ConfigMap injection uses: (a) valueFrom (b) envFrom.configMapRef (c) resourceFieldRef (d) fieldRef only — **b**
5. Container's own CPU limit is exposed via: (a) fieldRef (b) resourceFieldRef (c) configMapKeyRef (d) secretKeyRef — **b**
6. CPU resourceFieldRef rounds: (a) Down always (b) Up to nearest whole core by default (c) Never rounds (d) Randomly — **b**
7. Do Downward API env vars auto-update if the Pod field changes later? (a) Yes (b) No — **b**
8. Missing ConfigMap/Secret key reference causes: (a) Silent success (b) CreateContainerConfigError (c) ImagePullBackOff (d) Pod deletion — **b**
9. Real-world Downward API use case: (a) External API calls (b) Structured logs including the Pod's own name (c) DNS resolution (d) Image pulling — **b**
10. Best practice for resourceFieldRef precision: (a) Ignore units (b) Set an explicit divisor (c) Never use it (d) Always round manually — **b**

### Homework
Build a Pod using all four environment variable sources simultaneously (at least 8 total variables), deliberately include one naming collision to test precedence, and document the fully resolved `env` output with an explanation of where each value came from.

### Summary
Environment variables can be assembled from static values, ConfigMaps, Secrets, and the Downward API, with individual `env[]` entries taking precedence over bulk `envFrom` imports on naming collisions. Lab 10 closes this volume with resource limits in depth, plus a full mini project combining configuration and storage into one realistic system.

---

# Lab 10 — Resource Limits & Mini Project

**Estimated Duration:** 100 minutes
**Difficulty Level:** Intermediate (capstone for Volume 4)

### Learning Objectives
1. Understand resource requests and limits in depth, including QoS classes.
2. Explain CPU throttling versus OOMKilled behavior.
3. Configure namespace-wide ResourceQuotas and LimitRanges.
4. Combine ConfigMaps, Secrets, and dynamically-provisioned storage into one complete mini project.

### Prerequisites
- Completion of Labs 1–9.

### Theory

You've set `resources.requests`/`limits` throughout this manual without a full explanation — this lab provides it. A **request** is what the scheduler (Volume 1) guarantees is available on whatever node a Pod lands on, and is used purely for scheduling decisions. A **limit** is the hard ceiling the container is never allowed to exceed. CPU and memory behave very differently when a limit is hit: **CPU is throttled** — the container isn't killed, just slowed down, since CPU is a "compressible" resource that can simply be rationed over time; **memory causes an OOMKill** — the container is forcibly terminated, since memory is "incompressible" and can't simply be rationed once physically exhausted.

Based on how requests/limits are set, every Pod is assigned a **Quality of Service (QoS) class**, directly affecting eviction priority under node resource pressure: **Guaranteed** (every container's requests equal its limits, for both CPU and memory — highest priority, evicted last), **Burstable** (requests are set but are lower than limits, or only some resources have limits — medium priority), and **BestEffort** (no requests or limits set at all — lowest priority, evicted first under any pressure).

At the namespace level, a **ResourceQuota** caps the total resources (CPU, memory, object counts) consumable across an entire namespace, while a **LimitRange** sets default and min/max constraints for individual containers within a namespace — useful for enforcing that every Pod has at least some request/limit set, even if a developer forgets to specify one explicitly.

This volume's capstone mini project combines every concept: a Deployment consuming a ConfigMap and Secret (Labs 1–2, 9), backed by a dynamically-provisioned PersistentVolumeClaim (Lab 8), with carefully tuned resource requests/limits (this lab) — a complete, realistic application configuration.

### Architecture Diagram

```
   Node with 4 CPU, 8Gi memory total
        |
   Pod A: requests 1 CPU/2Gi, limit 2 CPU/2Gi  --> Burstable (CPU limit > request, mem request=limit)
   Pod B: requests 1 CPU/1Gi, limit 1 CPU/1Gi  --> Guaranteed (both match exactly)
   Pod C: no requests/limits set at all         --> BestEffort

   Under memory pressure, eviction order: BestEffort first, then Burstable, then Guaranteed last
   Under CPU pressure alone: no eviction, just throttling for any Pod exceeding its CPU limit
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl apply -f qos-demo.yaml
kubectl get pod qos-demo -o jsonpath='{.status.qosClass}'
```
**Purpose:** Deploys a Pod with `requests` exactly equal to `limits` and confirms it receives the `Guaranteed` QoS class.
**Expected Output:** `Guaranteed`
**Common mistakes:** Assuming QoS class is something you set directly — it's always automatically derived from the requests/limits configuration.
**Real-world usage:** Understanding QoS class is essential for predicting which Pods survive node memory pressure incidents.

```bash
kubectl apply -f resourcequota.yaml -n team-a
kubectl describe resourcequota -n team-a
```
**Purpose:** Applies a namespace-wide cap on total resource consumption and shows current usage against it.
**Real-world usage:** Preventing one team/namespace from consuming an entire shared cluster's resources.

```bash
kubectl apply -f limitrange.yaml -n team-a
kubectl run no-limits-pod --image=nginx -n team-a
kubectl get pod no-limits-pod -n team-a -o jsonpath='{.spec.containers[0].resources}'
```
**Purpose:** Confirms a LimitRange automatically applies default requests/limits to a Pod that didn't specify any itself.
**Real-world usage:** Safety net ensuring no Pod accidentally runs with completely unconstrained resource usage.

### YAML Files

`qos-demo.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata: { name: qos-demo }
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      resources:
        requests: { cpu: "200m", memory: "128Mi" }
        limits:   { cpu: "200m", memory: "128Mi" }   # Requests == limits --> Guaranteed QoS
```

`resourcequota.yaml`:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { name: team-a-quota }
spec:
  hard:
    requests.cpu: "4"          # Total across ALL Pods in this namespace
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"                  # Also caps total object counts, not just resource amounts
```

`limitrange.yaml`:
```yaml
apiVersion: v1
kind: LimitRange
metadata: { name: team-a-defaults }
spec:
  limits:
    - type: Container
      default: { cpu: "250m", memory: "128Mi" }        # Applied if a container omits limits
      defaultRequest: { cpu: "100m", memory: "64Mi" }   # Applied if a container omits requests
      max: { cpu: "1", memory: "512Mi" }                 # No container may request/limit above this
      min: { cpu: "50m", memory: "32Mi" }                 # No container may request/limit below this
```

**Volume 4 Mini Project** (`mini-project.yaml` — combining ConfigMap, Secret, dynamic PVC, and tuned resources):
```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: mp-config }
data:
  APP_MODE: "production"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata: { name: mp-secret }
type: Opaque
stringData:
  API_KEY: "demo-key-not-for-real-use"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: mp-data }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: standard        # Dynamically provisioned, per Lab 8
  resources: { requests: { storage: 1Gi } }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: mp-app }
spec:
  replicas: 2
  selector: { matchLabels: { app: mp-app } }
  template:
    metadata: { labels: { app: mp-app } }
    spec:
      containers:
        - name: app
          image: busybox
          command: ["sh", "-c", "sleep 3600"]
          envFrom:
            - configMapRef: { name: mp-config }
          env:
            - name: API_KEY
              valueFrom: { secretKeyRef: { name: mp-secret, key: API_KEY } }
          volumeMounts:
            - { name: data, mountPath: /var/lib/data }
          resources:
            requests: { cpu: "100m", memory: "64Mi" }
            limits:   { cpu: "200m", memory: "128Mi" }
      volumes:
        - name: data
          persistentVolumeClaim: { claimName: mp-data }
```

### Expected Output
```
$ kubectl get pod qos-demo -o jsonpath='{.status.qosClass}'
Guaranteed

$ kubectl get deployment mp-app
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
mp-app   2/2     2            2           30s

$ kubectl get pvc mp-data
NAME      STATUS   VOLUME                CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mp-data   Bound    pvc-...(auto)          1Gi        RWO            standard       30s
```

### Validation Steps
1. `qos-demo` reports QoS class `Guaranteed`.
2. `mp-app` Deployment shows `READY: 2/2`.
3. `mp-data` PVC shows `STATUS: Bound` with no manually-created PV.
4. `kubectl exec` into an `mp-app` Pod confirms `APP_MODE`, `LOG_LEVEL`, and `API_KEY` environment variables are all present and correctly sourced.

### Common Errors
- **Pods rejected for exceeding LimitRange max** — validation error at creation time if requests/limits fall outside the configured min/max bounds.
- **ResourceQuota blocking new Pod creation** — "exceeded quota" error when total namespace usage would surpass the configured hard limits.
- **Mini project's PVC stuck Pending** — revisit Lab 6/8 troubleshooting; confirm the referenced StorageClass exists and its provisioner is healthy.

### Troubleshooting
For LimitRange rejections, adjust the Pod's requests/limits to fall within the configured min/max bounds, or adjust the LimitRange itself if the bounds were set incorrectly. For ResourceQuota blocks, check `kubectl describe resourcequota` to see current usage versus the hard cap, and either free up capacity or request a quota increase. For a stuck mini-project PVC, apply the full troubleshooting chain from Labs 6–8 (StorageClass existence, CSI driver health).

### Practice Tasks
1. Create Pods demonstrating all three QoS classes (Guaranteed, Burstable, BestEffort) and confirm each via `jsonpath`.
2. Apply a ResourceQuota to a namespace and attempt to exceed it, observing the rejection.
3. Apply a LimitRange and create a Pod with no resources specified, confirming defaults are applied automatically.
4. Deliberately exceed a container's memory limit (e.g., a script allocating more memory than permitted) and observe the resulting OOMKilled status.
5. Build the full Volume 4 mini project from scratch and verify every component (ConfigMap, Secret, PVC, Deployment) independently.

### Challenge Lab
Extend the mini project with a ResourceQuota and LimitRange for its namespace, tuned specifically to accommodate exactly the `mp-app` Deployment's replica count and resource profile with a small amount of headroom, and document your capacity-planning reasoning.

### Best Practices
- Set `requests` realistically based on actual observed usage, not guesses — over-requesting wastes cluster capacity, under-requesting risks poor scheduling decisions and resource contention.
- Use `Guaranteed` QoS for genuinely critical workloads where eviction under pressure would be especially costly.
- Always pair application-level Deployments with namespace-level ResourceQuotas/LimitRanges in shared, multi-tenant clusters, to prevent any single team from starving others.

### Interview Questions
1. **Q: What's the difference between a resource request and a resource limit?** A: A request is what the scheduler guarantees is available for scheduling purposes; a limit is the hard ceiling the container cannot exceed.
2. **Q: What happens when a container exceeds its CPU limit?** A: It is throttled (slowed down), not killed, since CPU is a compressible resource.
3. **Q: What happens when a container exceeds its memory limit?** A: It is OOMKilled (forcibly terminated), since memory is incompressible.
4. **Q: What are the three QoS classes?** A: Guaranteed, Burstable, BestEffort.
5. **Q: What requests/limits configuration produces Guaranteed QoS?** A: Every container's requests exactly equal its limits, for both CPU and memory.
6. **Q: What's the eviction order under node resource pressure?** A: BestEffort evicted first, then Burstable, then Guaranteed last.
7. **Q: What does a ResourceQuota control?** A: Total resource consumption (and/or object counts) across an entire namespace.
8. **Q: What does a LimitRange control?** A: Default and min/max resource constraints for individual containers within a namespace.
9. **Q: Why combine a ConfigMap, Secret, and PVC in one Deployment, as in this volume's mini project?** A: To represent a complete, realistic application needing configuration, sensitive data, and persistent storage together.
10. **Q: Why is QoS class not something you set directly?** A: It's automatically derived by Kubernetes from how requests/limits are configured.

### Quiz
1. What happens when a CPU limit is exceeded? (a) OOMKilled (b) Throttled (c) Pod deleted (d) Nothing — **b**
2. What happens when a memory limit is exceeded? (a) Throttled (b) OOMKilled (c) Nothing (d) CPU reduced — **b**
3. Guaranteed QoS requires: (a) No requests set (b) Requests exactly equal limits for CPU and memory (c) Only memory limits set (d) Random configuration — **b**
4. Eviction order under pressure: (a) Guaranteed first (b) BestEffort first, Guaranteed last (c) Random (d) Burstable first always — **b**
5. ResourceQuota controls: (a) Individual container defaults (b) Total namespace-wide resource consumption (c) Node hardware (d) DNS — **b**
6. LimitRange controls: (a) Namespace totals (b) Per-container default/min/max constraints (c) Cluster-wide limits (d) Node counts — **b**
7. BestEffort QoS means: (a) Highest priority (b) No requests/limits set at all; lowest priority (c) Guaranteed resources (d) Not allowed — **b**
8. QoS class is: (a) Manually set by the user (b) Automatically derived from requests/limits (c) Fixed for all Pods (d) Set by the node — **b**
9. A Pod exceeding LimitRange max at creation: (a) Is created anyway (b) Is rejected with a validation error (c) Silently ignored (d) Auto-adjusted without error — **b**
10. Why pair ResourceQuotas/LimitRanges with Deployments in shared clusters? (a) Not necessary (b) Prevents one team from starving others of resources (c) Only for security (d) Required by all clusters — **b**

### Homework
Build the complete Volume 4 mini project, add a ResourceQuota and LimitRange to its namespace, then intentionally attempt to violate each (exceed the quota, exceed the LimitRange max) and document the exact error messages Kubernetes returns for each violation.

### Summary
This capstone covered resource requests/limits, QoS classes, and namespace-wide ResourceQuotas/LimitRanges, then combined ConfigMaps, Secrets, and dynamically-provisioned storage into one complete, realistic application — exercising every major concept from this volume. You are now ready for Volume 5, which covers cluster administration and troubleshooting: RBAC, Service Accounts, node management, scheduling, taints/tolerations, affinity, cluster upgrades, etcd backup/restore, and production troubleshooting.

---

# Volume 4 — End of Volume Materials

## Volume Review

Volume 4 completed the picture of how applications get their data and configuration:

- **Lab 1–2:** ConfigMaps and Secrets — non-sensitive and sensitive configuration, separated from container images.
- **Lab 3:** Volumes (`emptyDir`, `hostPath`) — ephemeral, Pod- or node-scoped storage.
- **Lab 4–5:** PersistentVolumes and PersistentVolumeClaims — true, Pod-lifecycle-independent storage via static provisioning.
- **Lab 6–8:** StorageClasses, CSI drivers, and dynamic provisioning — the real-world-standard, automated storage workflow.
- **Lab 9:** Environment variables — consolidating all configuration sources, including the Downward API.
- **Lab 10:** Resource requests/limits, QoS classes, ResourceQuotas/LimitRanges, and a full mini project combining everything.

You should now be able to design complete application configurations combining ConfigMaps, Secrets, dynamically-provisioned storage, and properly-tuned resource limits. Volume 5 shifts to cluster administration and troubleshooting — RBAC, node management, scheduling controls, and production-grade operational skills.

## Practice Exam (25 Questions)

1. Why should configuration be separated from container images? — Allows the same image to be reused across environments without rebuilding.
2. What are the three ways to consume a ConfigMap? — Environment variables, command-line args (via env vars), mounted files.
3. Does updating a ConfigMap update already-injected environment variables? — No, requires a Pod restart.
4. How does a Secret differ from a ConfigMap? — Intended for sensitive data; otherwise structurally similar.
5. Does base64 encoding provide real security for Secrets? — No, it's trivially reversible.
6. What actually protects Secrets? — RBAC plus etcd encryption-at-rest.
7. What is emptyDir's lifetime? — Tied to the Pod; deleted when the Pod is deleted.
8. What's hostPath's key limitation? — Ties data to one specific node.
9. What is a PersistentVolume? — A cluster-level, Pod-lifecycle-independent storage resource.
10. Are PersistentVolumes namespaced? — No, cluster-scoped.
11. What are the three access modes? — ReadWriteOnce, ReadOnlyMany, ReadWriteMany.
12. What does a PersistentVolumeClaim represent? — A namespaced request for storage matching specific requirements.
13. Do Pods reference PVs or PVCs? — PVCs, never PVs directly.
14. What does a StorageClass reference? — A provisioner.
15. What does dynamic provisioning eliminate? — Manual PV pre-creation.
16. What is CSI? — Container Storage Interface, a standardized storage plugin API.
17. What are the two main CSI driver components? — Controller and node (DaemonSet) components.
18. What field enables volume expansion? — allowVolumeExpansion: true on the StorageClass.
19. Can PVCs be shrunk? — No.
20. What are the four environment variable sources? — Static values, ConfigMap/Secret refs, Downward API.
21. What does the Downward API expose? — Pod metadata and resource info as env vars or files.
22. What happens on CPU limit exceedance? — Throttling.
23. What happens on memory limit exceedance? — OOMKill.
24. What are the three QoS classes? — Guaranteed, Burstable, BestEffort.
25. What does a ResourceQuota control? — Total resource consumption across a namespace.

## Practical Assessment

Without referring back to the labs:
1. Create a ConfigMap and Secret, consuming both in one Pod (env vars and mounted files).
2. Create a PersistentVolume and PersistentVolumeClaim that bind successfully (static provisioning).
3. Create a dynamically-provisioned PVC using the cluster's default StorageClass.
4. Write data to a PVC-backed Pod, delete and recreate the Pod, and confirm data persistence.
5. Create a Pod with Guaranteed QoS and confirm via `jsonpath`.
6. Apply a ResourceQuota and LimitRange to a namespace and confirm both are enforced.
7. Build the full mini project (ConfigMap + Secret + dynamic PVC + Deployment) from Lab 10.
8. Clean up every resource created in this assessment.

**Self-grading:** You should complete every step from memory in under 35 minutes.

## Mini Project Reference
See Lab 10 for the full ConfigMap + Secret + dynamic PVC + tuned-resources mini project. Use it as your reference implementation for the Practical Assessment above.

## Troubleshooting Exercises

1. **Scenario:** A Pod shows `CreateContainerConfigError`. **Diagnosis:** Check `kubectl describe pod` for a missing ConfigMap/Secret reference (Labs 1–2).
2. **Scenario:** A PVC stays Pending indefinitely. **Diagnosis:** Check StorageClass existence and CSI driver health (Labs 6–8).
3. **Scenario:** A Pod is OOMKilled repeatedly. **Diagnosis:** Review actual memory usage versus configured limits; consider raising limits or optimizing the application (Lab 10).
4. **Scenario:** A new Pod is rejected at creation with a resource-related error. **Diagnosis:** Check ResourceQuota and LimitRange constraints in that namespace (Lab 10).
5. **Scenario:** Data is missing after a Pod restart. **Diagnosis:** Confirm the Pod was using a PVC-backed volume, not `emptyDir` (Labs 3–5).

## Volume 4 Cheat Sheet

| Task | Command |
|---|---|
| Create ConfigMap from literals | `kubectl create configmap <name> --from-literal=KEY=value` |
| Create ConfigMap from file | `kubectl create configmap <name> --from-file=<path>` |
| Create generic Secret | `kubectl create secret generic <name> --from-literal=KEY=value` |
| Create registry Secret | `kubectl create secret docker-registry <name> --docker-server=... --docker-username=... --docker-password=...` |
| List PersistentVolumes | `kubectl get pv` |
| List PersistentVolumeClaims | `kubectl get pvc` |
| List StorageClasses | `kubectl get storageclass` |
| List CSI drivers | `kubectl get csidrivers` |
| Expand a PVC | `kubectl patch pvc <name> -p '{"spec":{"resources":{"requests":{"storage":"<new-size>"}}}}'` |
| Check Pod QoS class | `kubectl get pod <name> -o jsonpath='{.status.qosClass}'` |
| Describe ResourceQuota usage | `kubectl describe resourcequota -n <namespace>` |

## kubectl Command Reference (Volume 4 subset)

`create configmap`, `create secret generic`, `create secret docker-registry`, `get pv`, `get pvc`, `get storageclass`, `get csidrivers`, `describe pv`, `describe pvc`, `describe resourcequota`, `patch` (for PVC expansion).

## YAML Reference (Volume 4 subset)

| Object | apiVersion | Key spec fields |
|---|---|---|
| ConfigMap | v1 | data |
| Secret | v1 | type, data / stringData |
| PersistentVolume | v1 | capacity, accessModes, persistentVolumeReclaimPolicy, storageClassName |
| PersistentVolumeClaim | v1 | accessModes, resources.requests.storage, storageClassName |
| StorageClass | storage.k8s.io/v1 | provisioner, parameters, reclaimPolicy, volumeBindingMode |
| ResourceQuota | v1 | hard (resource/object caps) |
| LimitRange | v1 | limits (default, defaultRequest, min, max) |

---

*End of Volume 4 — Storage & Configuration. Continue to Volume 5: Administration & Troubleshooting.*
