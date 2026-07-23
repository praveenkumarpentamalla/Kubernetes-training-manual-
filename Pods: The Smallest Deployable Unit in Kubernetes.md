# Lab 02 — Pods: The Smallest Deployable Unit in Kubernetes

**Kubernetes Version:** v1.31+
**Estimated Time:** 60–75 minutes
**Difficulty:** Beginner
**Lab Type:** Hands-on, real cluster required

---

## 1. Introduction

In Lab 01, you built a real cluster but never actually ran an application on it. In this lab, you'll run your first one — and to do that, you need to meet the **Pod**.

A **Pod** is the smallest unit Kubernetes lets you deploy. It's tempting to think "a Pod is just a container," but that's not quite right, and getting this distinction correct now will save you confusion for the rest of this course.

A Pod is a **wrapper** around one or more containers. Every container inside the same Pod:
- Shares the same network — they can talk to each other over `localhost`, and the whole Pod gets **one** IP address
- Can share storage, if you configure it to (covered in a later lab)
- Is always scheduled onto the same node, together, as one unit
- Lives and dies together — you can't have "half a Pod" running

Most Pods contain exactly **one** container. Multi-container Pods exist for specific advanced patterns (covered in a later lab) — but 90% of the time, one Pod means one container, and it's completely fine to think of them as nearly the same thing while you're learning.

```
                         A POD
        +--------------------------------------+
        |   One shared network (one Pod IP)       |
        |                                        |
        |     +----------------------+             |
        |     |  Container: nginx     |             |
        |     |  (your application)   |             |
        |     +----------------------+             |
        |                                        |
        +--------------------------------------+
              Scheduled onto exactly ONE node
```

---

## 2. Objectives

By the end of this lab, you will be able to:

1. Explain the difference between a Pod and a container in your own words.
2. Write a complete Pod manifest (YAML file) from scratch and explain every single line.
3. Create a Pod using `kubectl apply` and understand exactly what happens inside Kubernetes when you do.
4. Use `kubectl get`, `kubectl describe`, `kubectl logs`, and `kubectl exec` to inspect a running Pod.
5. Understand the Pod lifecycle: `Pending` → `Running` → `Succeeded`/`Failed`.
6. Delete a Pod correctly and understand what "correctly" even means here.
7. Diagnose the most common Pod failures: `ImagePullBackOff` and `CrashLoopBackOff`.

---

## 3. Scenario

Your manager at CloudNova Inc. is happy with your cluster from Lab 01. Now she wants proof you can actually run something on it. Your task: deploy a simple web server, confirm it's genuinely running and serving traffic, look at its logs like a real engineer would, and then safely tear it down.

---

## 4. Environment

This lab continues directly from Lab 01. You should have:
- The `my-first-cluster` Kind cluster still running (`kubectl get nodes` should show 2 `Ready` nodes)
- `kubectl` v1.31.0 installed and working

If your cluster from Lab 01 was deleted or your laptop restarted, recreate it now:
```bash
kind create cluster --name my-first-cluster --config kind-config.yaml
```

---

## 5. Step-by-Step Tasks

### Task 1 — Write your first Pod manifest

A **manifest** is just a YAML file describing a Kubernetes object you want to exist. Create a file named `nginx-pod.yaml`:

```bash
cat > nginx-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod
  labels:
    app: nginx-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
EOF
```

**Line-by-line YAML explanation:**

| Line | Meaning |
|---|---|
| `apiVersion: v1` | Every Kubernetes object belongs to an "API group" and version. Pods belong to the original, foundational API group, which is simply called `v1` (sometimes referred to as the "core" group). You will see other values like `apps/v1` in later labs, for different object types. |
| `kind: Pod` | Tells Kubernetes exactly what **type** of object this file describes. This is a real Kubernetes field, unrelated to the `kind` CLI tool from Lab 01 — same word, two unrelated meanings, which trips up almost everyone at first. |
| `metadata:` | The start of a section holding **identifying information** about this object — its name, labels, etc. — as opposed to `spec:` below, which describes what it should actually *do*. |
| `  name: my-first-pod` | The unique name for this Pod. Must be unique within its namespace (namespaces are covered in a later lab; for now everything lives in the `default` namespace). |
| `  labels:` | The start of a small set of key-value tags you attach to this object, used later for grouping and selecting Pods (you'll see this become critical in the Deployments lab). |
| `    app: nginx-demo` | One specific label: the key is `app`, the value is `nginx-demo`. This is just a tag we're choosing — the name `app` is a common convention, not a required keyword. |
| `spec:` | The start of the **specification** — what this Pod should actually contain and do. |
| `  containers:` | A **list** of containers this Pod should run. Even though we only have one, it's still written as a list (using `-`), because Pods *can* hold more than one. |
| `    - name: nginx` | The name of this specific container, within the Pod. Required so you can refer to it individually later (e.g., in a multi-container Pod). |
| `      image: nginx:1.27` | Which container image to run. `nginx` is the image name (a well-known, official web server image); `1.27` is the **tag**, pinning an exact version. Never using a tag (or using `latest`) is considered bad practice, because it makes your deployment unpredictable. |
| `      ports:` | A list of ports this container listens on. This is **informational/documentation only** — it does not, by itself, make the port reachable from outside the Pod (that requires a Service, covered in a later lab). |
| `        - containerPort: 80` | Declares that this container listens on port 80 inside its own network namespace. |

```
   nginx-pod.yaml describes THIS object:

   kind: Pod
   +--------------------------------+
   |  metadata.name: my-first-pod     |
   |  metadata.labels: app=nginx-demo  |
   |                                  |
   |  spec.containers:                 |
   |    - nginx:1.27, port 80           |
   +--------------------------------+
```

---

### Task 2 — Create the Pod

```bash
kubectl apply -f nginx-pod.yaml
```

**Word-by-word explanation:**
- `kubectl` — the program.
- `apply` — a subcommand meaning "make the cluster's actual state match what's described in this file." If the object doesn't exist, `apply` creates it. If it already exists, `apply` updates it to match. This is different from `kubectl create`, which only creates and fails loudly if the object already exists — `apply` is the one you'll use almost all the time in real work.
- `-f nginx-pod.yaml` — `-f` means "read the object definition from this file" (`-f` stands for "file").

**Realistic output:**
```
pod/my-first-pod created
```

**Output explanation:**
- `pod/` — confirms the type of object that was affected (a Pod).
- `my-first-pod` — the name of the object, matching what you wrote in the YAML.
- `created` — confirms this object did not exist before, and now it does.

### What actually happens internally right now

This single command triggers a real chain of events inside your cluster:

```
   1. kubectl reads nginx-pod.yaml and sends it as an HTTPS
      request to the API server (the "front door" of the cluster)
                        |
   2. The API server validates the YAML and saves this Pod's
      desired state into etcd (the cluster's database)
                        |
   3. The scheduler notices a new, unassigned Pod and picks
      the best node for it (here, your single worker node)
                        |
   4. The kubelet (an agent running on that node) notices it
      now owns this Pod and tells the container runtime
      (containerd) to actually download the image and start it
                        |
   5. Your Pod transitions: Pending -> Running
```

You don't need to memorize this flow perfectly yet (it's covered in full depth in Lab 03), but understanding that `kubectl apply` triggers a real, multi-step process — not just "some file gets saved somewhere" — is important.

---

### Task 3 — Watch the Pod come to life

```bash
kubectl get pods
```

**Word-by-word explanation:**
- `pods` — plural, listing all Pods (in the current, default namespace, since we didn't specify `-A` this time).

**Realistic output (immediately after creation):**
```
NAME           READY   STATUS    RESTARTS   AGE
my-first-pod   0/1     Pending   0          2s
```

Wait a few seconds and run it again:

```
NAME           READY   STATUS    RESTARTS   AGE
my-first-pod   1/1     Running   0          8s
```

**Output explanation, column by column:**
- `NAME` — the Pod's name, exactly as you defined it.
- `READY` — containers ready versus total containers in the Pod. `0/1` at first means "0 of 1 containers ready yet" (still starting up); `1/1` means fully ready.
- `STATUS` — the Pod's overall phase. `Pending` means Kubernetes has accepted the Pod but it isn't running yet (usually waiting on scheduling or image download). `Running` means at least one container is actively running.
- `RESTARTS` — how many times this Pod's container has crashed and restarted. `0` is healthy.
- `AGE` — time since creation.

To watch this transition live instead of re-typing the command:

```bash
kubectl get pods --watch
```

**Word-by-word explanation:**
- `--watch` — instead of printing once and exiting, keep the connection open and print a new line every time this Pod's status changes. Press `Ctrl+C` to stop watching.

---

### Task 4 — Inspect the Pod in detail

```bash
kubectl describe pod my-first-pod
```

**Word-by-word explanation:**
- `describe` — a subcommand that prints a detailed, human-readable report about one specific object — far more detail than `get` shows, including a chronological history of everything that happened to it.
- `pod my-first-pod` — the type and name of the specific object to describe.

**Realistic output (trimmed to the most important parts):**
```
Name:             my-first-pod
Namespace:        default
Priority:         0
Node:             my-first-cluster-worker/172.18.0.2
Start Time:       Wed, 23 Jul 2026 09:12:04 +0000
Labels:           app=nginx-demo
Status:           Running
IP:               10.244.1.5
IPs:
  IP:  10.244.1.5
Containers:
  nginx:
    Container ID:   containerd://a1b2c3d4e5f6...
    Image:          nginx:1.27
    Image ID:       docker.io/library/nginx@sha256:...
    Port:           80/TCP
    State:          Running
      Started:      Wed, 23 Jul 2026 09:12:07 +0000
    Ready:          True
    Restart Count:  0
Conditions:
  Type              Status
  PodReadyToStartContainers   True
  Initialized                True
  Ready                       True
  ContainersReady             True
  PodScheduled                True
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  30s   default-scheduler  Successfully assigned default/my-first-pod to my-first-cluster-worker
  Normal  Pulling    29s   kubelet            Pulling image "nginx:1.27"
  Normal  Pulled     26s   kubelet            Successfully pulled image "nginx:1.27" in 2.8s
  Normal  Created    26s   kubelet            Created container nginx
  Normal  Started    26s   kubelet            Started container nginx
```

**Output explanation, section by section:**
- `Node:` — confirms exactly which physical (well, container-simulated) node this Pod landed on — here, our worker node, exactly as the scheduler decided.
- `IP:` — the unique internal cluster IP address assigned to this Pod. Every Pod gets one, automatically.
- `Containers:` → `State: Running` — confirms this specific container, inside the Pod, is actively running.
- `Conditions:` — a checklist of internal readiness milestones. `Ready: True` means Kubernetes considers this Pod fully healthy and (once we add a Service in a later lab) eligible to receive traffic.
- `Events:` — **this is the single most useful section for troubleshooting**, and you'll rely on it in nearly every future lab. It's a chronological log of everything that's happened: the Pod was `Scheduled` to a node, its image started `Pulling`, finished being `Pulled`, the container was `Created`, and finally `Started`. When something goes wrong, this is always the first place to look.

---

### Task 5 — View the Pod's logs

```bash
kubectl logs my-first-pod
```

**Word-by-word explanation:**
- `logs` — a subcommand that prints whatever the container has written to its standard output (stdout) and standard error (stderr) streams — exactly what you'd see if you'd run the container directly and watched its terminal output.
- `my-first-pod` — which Pod to read logs from.

**Realistic output:**
```
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2026/07/23 09:12:07 [notice] 1#1: nginx/1.27.0
2026/07/23 09:12:07 [notice] 1#1: start worker processes
```

**Output explanation:**
This is nginx's own real startup log — the exact same output you'd see running this image anywhere else. `kubectl logs` doesn't add anything of its own; it's a direct window into the container's output. Lines like `Configuration complete; ready for start up` confirm the web server itself successfully initialized.

To follow logs live, as new lines are written (useful for a real running service):
```bash
kubectl logs my-first-pod --follow
```
**Word-by-word explanation:** `--follow` (often shortened to `-f`) keeps the connection open and streams new log lines as they happen, instead of printing what exists once and exiting. Press `Ctrl+C` to stop.

---

### Task 6 — Get a shell inside the running Pod

```bash
kubectl exec -it my-first-pod -- /bin/bash
```

**Word-by-word explanation:**
- `exec` — a subcommand that runs a command *inside* an already-running container, rather than on your own machine.
- `-it` — actually two combined single-letter flags: `-i` means "interactive" (keep standard input open so you can type), `-t` means "allocate a pseudo-terminal" (so it behaves like a normal interactive shell, with prompts and line editing). Together, `-it` is what makes this feel like a real terminal session instead of running one blind command.
- `my-first-pod` — which Pod to exec into.
- `--` — a separator meaning "everything after this belongs to the command being run inside the container, not to `kubectl` itself." This matters because without it, `kubectl` might try to interpret `/bin/bash`'s own flags as its own.
- `/bin/bash` — the actual command to run inside the container: start an interactive Bash shell.

**Realistic output:**
```
root@my-first-pod:/#
```

You are now, literally, inside the container. Try:
```bash
curl -s localhost:80 | head -5
```

**Realistic output:**
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
```

This proves nginx is genuinely serving real web traffic on port 80, **from inside its own Pod** — exactly what the `ports:` field in your YAML declared. Type `exit` to leave the shell and return to your own machine's terminal.

```bash
exit
```

---

### Task 7 — Delete the Pod correctly

```bash
kubectl delete pod my-first-pod
```

**Word-by-word explanation:**
- `delete` — a subcommand that removes an object from the cluster.
- `pod my-first-pod` — the type and name of the object to remove.

**Realistic output:**
```
pod "my-first-pod" deleted
```

### What happens internally when you delete a Pod

Kubernetes doesn't just instantly `kill -9` the container. It performs a **graceful shutdown**:

```
   1. kubectl sends a delete request to the API server
                        |
   2. The API server marks the Pod for deletion and sends
      a SIGTERM signal to the container (a polite "please
      shut yourself down" signal, not a forceful kill)
                        |
   3. The container has a grace period (30 seconds by
      default) to shut itself down cleanly
                        |
   4. If it hasn't exited by the end of the grace period,
      Kubernetes sends SIGKILL, forcefully terminating it
                        |
   5. Once fully stopped, the Pod object is removed from etcd
```

Confirm it's really gone:
```bash
kubectl get pods
```

**Realistic output:**
```
No resources found in default namespace.
```

---

## 6. Verification

Run this checklist yourself, from memory, without looking back at the earlier tasks:

```bash
kubectl apply -f nginx-pod.yaml
kubectl get pods
kubectl describe pod my-first-pod | tail -10
kubectl logs my-first-pod | tail -5
kubectl delete pod my-first-pod
```

If every command produced sensible output matching what you learned above (a `created` message, a `Running` status, real `Events`, real nginx logs, and a `deleted` confirmation), you've fully internalized the basic Pod lifecycle.

---

## 7. Common Errors

**Error 1 — `ImagePullBackOff`:**
```
NAME           READY   STATUS             RESTARTS   AGE
my-first-pod   0/1     ImagePullBackOff   0          45s
```
**Meaning:** Kubernetes cannot download the container image — usually a typo in the image name/tag, or no internet access from inside the Kind node.

**Error 2 — `CrashLoopBackOff`:**
```
NAME           READY   STATUS             RESTARTS   AGE
my-first-pod   0/1     CrashLoopBackOff   4          90s
```
**Meaning:** The container starts, then exits (crashes) almost immediately, repeatedly. Kubernetes keeps retrying with increasing delays between attempts (this "increasing delay" is exactly what "backoff" means in the name).

**Error 3:**
```
error: unable to upgrade connection: container not found ("nginx")
```
**Meaning:** You tried to `exec` or `logs` using a container name that doesn't match what's actually inside the Pod — usually a typo, or you're targeting the wrong Pod entirely.

---

## 8. Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `ImagePullBackOff` | Typo in image name or tag | Run `kubectl describe pod my-first-pod` and read the exact error in the Events section — it will state precisely what image string it tried and failed to pull |
| `ImagePullBackOff` on a correct image name | No internet access inside the Kind node | Run `docker exec -it my-first-cluster-worker ping -c 2 8.8.8.8` to test connectivity from inside the node itself |
| `CrashLoopBackOff` | The container's main process exits immediately | Run `kubectl logs my-first-pod --previous` (note the new flag) to see the log output from the **last, crashed** attempt, since a currently-crashed Pod has no "current" logs to show |
| Pod stuck at `Pending` for a long time | No node has enough resources, or scheduling is blocked | Run `kubectl describe pod my-first-pod` and read the Events section for a specific scheduling failure reason |
| `kubectl exec` returns `error: unable to upgrade connection` | Wrong container name, or Pod isn't actually Running yet | Run `kubectl get pods` first to confirm `STATUS: Running`, then re-check your container's name with `kubectl describe pod` |

---

## 9. Common Mistakes

- **Forgetting `apiVersion` or `kind` in a YAML file.** Kubernetes will reject the file outright with a clear error — these two fields are mandatory on every single object you'll ever write.
- **Using `latest` as an image tag.** It looks convenient but makes your deployment unpredictable — a different image could be pulled tomorrow with no warning. Always pin an explicit version, as we did with `nginx:1.27`.
- **Confusing `kubectl delete` with a "graceful" instant removal.** New learners are often surprised their Pod takes several seconds to actually disappear — that's the grace period working as intended, not a bug.
- **Trying to `kubectl exec` into a Pod that's still `Pending`.** There's no running container to connect to yet — always confirm `STATUS: Running` first.
- **Thinking `ports.containerPort` in the YAML makes your app reachable from your laptop's browser.** It does not, by itself — it's purely documentation of what the container listens on internally. Actually reaching a Pod from outside the cluster requires a Service, which is a topic for a later lab.

---

## 10. Challenge Exercises

1. Modify `nginx-pod.yaml` to run `httpd` (Apache) instead of `nginx`, using an explicit version tag, and confirm via `kubectl logs` that it started successfully.
2. Deliberately introduce a typo into the image tag (e.g., `nginx:1.9999`) and practice diagnosing the resulting `ImagePullBackOff` using only `kubectl describe`.
3. Create a Pod running the `busybox` image with this command instead of a web server: `command: ["sh", "-c", "echo hello; sleep 5; exit 1"]`. Watch it enter `CrashLoopBackOff` and use `kubectl logs --previous` to read its final output before it crashed.
4. Time how long `kubectl delete pod my-first-pod` actually takes to complete versus how long the Pod takes to fully vanish from `kubectl get pods` — relate this back to the 30-second default grace period explained in Task 7.
5. Without writing a new YAML file, use `kubectl run quick-pod --image=nginx:1.27 --restart=Never` (a new, imperative way to create a Pod, as opposed to `apply -f`) and compare the resulting Pod to the one you built declaratively — are they functionally identical?

---

## 11. Quiz

**1. What is the correct relationship between a Pod and a container?**
a) They are exactly the same thing, always
b) A Pod is a wrapper that can hold one or more containers, sharing network and optionally storage
c) A container can hold multiple Pods
d) Pods and containers are unrelated Kubernetes concepts

**2. What does `apply` do differently from `create`?**
a) `apply` is slower
b) `apply` creates or updates to match the file; `create` only creates and fails if it already exists
c) `apply` only works on Pods
d) They are identical commands

**3. In `kubectl get pods` output, what does `READY: 0/1` mean?**
a) The Pod has been deleted
b) 0 of 1 containers in this Pod are currently ready
c) The Pod has 0 restarts
d) The Pod is scheduled on 1 of 0 nodes

**4. Where should you look first when a Pod isn't behaving as expected?**
a) The Kubernetes source code
b) The `Events` section of `kubectl describe pod`
c) Your laptop's system logs
d) The image's Docker Hub page

**5. What does `kubectl logs my-first-pod --previous` show?**
a) Logs from a different Pod entirely
b) Logs from the last, already-crashed instance of the container, useful when the current one has restarted
c) Logs from before the Pod existed
d) A preview of logs that will appear in the future

**Answers:** 1-b, 2-b, 3-b, 4-b, 5-b

---

## 12. Summary

In this lab, you:

- Learned precisely what a Pod is — a wrapper around one or more containers, sharing network and identity — and how it differs from a plain container.
- Wrote a complete Pod manifest from scratch and understood every field: `apiVersion`, `kind`, `metadata`, `labels`, and the full `spec.containers` block.
- Used `kubectl apply` to create a real Pod, and traced exactly what happens internally, from the API server through the scheduler to the kubelet.
- Watched a Pod move through its lifecycle: `Pending` → `Running`.
- Used `kubectl describe` to read a Pod's full details and — critically — its `Events` history, your primary troubleshooting tool going forward.
- Used `kubectl logs` to view real application output, and `kubectl exec` to get an actual interactive shell inside a running container.
- Deleted a Pod and understood the graceful-shutdown process (SIGTERM, grace period, SIGKILL) happening underneath.
- Learned to recognize and begin diagnosing the two most common Pod failure states: `ImagePullBackOff` and `CrashLoopBackOff`.

You can now deploy, inspect, and remove real workloads on a real cluster. However, everything you did in this lab was **manual** — if this Pod crashed permanently, nothing would bring it back. That's exactly the gap the next lab closes.

**Next: Lab 03 — Understanding Kubernetes Architecture**, where you'll open up the `kube-system` namespace you glimpsed in Lab 01 and learn exactly what each control-plane component does and how they work together to make what you just did in this lab possible.
