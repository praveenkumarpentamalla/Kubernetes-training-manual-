# Lab 01 — Understanding Kubernetes and Building Your First Cluster

**Kubernetes Version:** v1.31+
**Estimated Time:** 45–60 minutes
**Difficulty:** Beginner
**Lab Type:** Hands-on, real cluster required

---

## 1. Introduction

Kubernetes is a system that runs your applications for you across many machines, restarts them when they crash, and lets you describe *what you want* instead of typing out every step by hand.

Before Kubernetes, if you wanted to run a web application reliably, you had to:
- SSH into a server
- Copy your application onto it
- Start it manually (or write a script to start it)
- Watch it, and if it crashed, restart it yourself
- If you needed more capacity, repeat all of this on a second server
- Somehow keep track of which server is running what

Kubernetes automates every one of those steps. You write down, in a text file, "I want 3 copies of my app running, each with this much memory," and Kubernetes makes that true — and keeps it true, even if a server dies at 3 AM.

In this lab, you will not read about Kubernetes — you will **build a real, working Kubernetes cluster** on your own machine and run your first commands against it. Every command you type will hit a real Kubernetes API server, not a simulation.

### 1.1 What exactly is a "cluster"?

A **cluster** is a group of machines (called **nodes**) working together as one system. In a real company, a cluster might have 50 physical servers. In this lab, we'll build a cluster using **containers pretending to be servers** — this is exactly what a tool called **Kind** (Kubernetes IN Docker) does, and it's one of the most common ways professionals test Kubernetes on a laptop before touching a real, expensive cloud cluster.

```
                    YOUR CLUSTER (after this lab)
        +-------------------------------------------------+
        |                                                   |
        |   +----------------------+   +------------------+ |
        |   |  CONTROL PLANE NODE   |   |  WORKER NODE      | |
        |   |  (the "brain")        |   |  (runs your apps)  | |
        |   +----------------------+   +------------------+ |
        |                                                   |
        +-------------------------------------------------+
              Both "nodes" above are actually Docker
              containers running on YOUR single laptop.
```

---

## 2. Objectives

By the end of this lab, you will be able to:

1. Explain, in plain language, what a Kubernetes cluster is and why it exists.
2. Install the two required tools: `kubectl` (the command-line tool you use to talk to Kubernetes) and `kind` (the tool that creates a local cluster).
3. Write a cluster configuration file and understand every line in it.
4. Create a real, 2-node Kubernetes cluster running Kubernetes v1.31.
5. Run your first `kubectl` commands and understand exactly what every word in each command means.
6. Read and understand real `kubectl` output, column by column.
7. Diagnose and fix the most common setup mistakes.

---

## 3. Scenario

You have just joined **CloudNova Inc.** as a junior DevOps engineer. Your first task, before touching any real company infrastructure, is to build a **personal Kubernetes sandbox** on your workstation. Your manager wants to see that you can:

- Stand up a working cluster from nothing
- Confirm it's healthy using the correct commands
- Explain to a non-technical teammate what just happened

This lab simulates exactly that first-day task.

---

## 4. Environment

This lab assumes:

- A Linux machine (Ubuntu 22.04 or similar) or a Linux-compatible shell (WSL2 on Windows also works)
- `sudo` access
- At least 4 GB of free RAM and 10 GB of free disk space
- Basic Linux comfort: you know how to run a command, and you know what `sudo` does. **No Kubernetes knowledge is assumed at all.**

We will install everything else together, from scratch, in this lab.

---

## 5. Step-by-Step Tasks

### Task 1 — Verify Docker is installed

Kind needs Docker (or a compatible container engine) to create its fake "nodes." Let's check first.

```bash
docker --version
```

**Word-by-word explanation:**
- `docker` — the name of the program we are running (the Docker command-line client).
- `--version` — a **flag** (an option that changes a command's behavior). This specific flag tells Docker: "Don't do anything except print your own version number and exit."

**Realistic output:**
```
Docker version 24.0.7, build afdd53b
```

**Output explanation:**
- `Docker version 24.0.7` — the installed Docker Engine's version number.
- `build afdd53b` — a short hash identifying the exact build; not important for this lab, but useful when reporting bugs.

If instead you see:
```
docker: command not found
```

Docker isn't installed. Install it first:

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
```

**Word-by-word explanation of the last line:**
- `sudo` — run the following command as the administrator (root) user, since modifying user permissions requires elevated rights.
- `usermod` — the Linux program that modifies an existing user account.
- `-aG docker` — `-a` means "append" (add to existing groups, don't replace them), `-G docker` means "add this user to the group named `docker`." Being in the `docker` group lets you run Docker commands without typing `sudo` every time.
- `$USER` — a Linux environment variable that automatically holds your current username.

After running this, **log out and log back in** (or run `newgrp docker`) for the group change to take effect. Then re-run `docker --version` to confirm.

---

### Task 2 — Install `kubectl` (the Kubernetes command-line tool)

`kubectl` (pronounced "koob-cuttle" or "koob-control-el," people argue about this constantly) is the tool you use to send instructions to any Kubernetes cluster, anywhere — local or in the cloud.

```bash
curl -LO "https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl"
```

**Word-by-word explanation:**
- `curl` — a program that downloads content from a URL.
- `-L` — a flag meaning "follow redirects" (if the URL points somewhere else, follow it automatically instead of failing).
- `-O` — a flag meaning "save the downloaded file using its original filename" (here, that will be a file literally named `kubectl`).
- `"https://dl.k8s.io/..."` — the exact URL of the official Kubernetes v1.31.0 binary for 64-bit Linux (`amd64`).

```bash
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
```

**Word-by-word explanation:**
- `chmod` — "change mode," the Linux command that changes a file's permissions.
- `+x` — add ("+") the execute ("x") permission, meaning "allow this file to be run as a program."
- `mv` — "move" the file from its current location to a new one.
- `/usr/local/bin/kubectl` — a standard system directory that's already on every user's `PATH` (the list of folders Linux searches when you type a command name), so typing `kubectl` from anywhere will now find this file.

**Verify:**
```bash
kubectl version --client
```

**Word-by-word explanation:**
- `kubectl` — the program.
- `version` — a **subcommand** (kubectl has many; this one prints version information).
- `--client` — a flag telling `kubectl` "only show me your own (client) version — don't try to contact a cluster yet," since we don't have one running yet.

**Realistic output:**
```
Client Version: v1.31.0
Kustomize Version: v5.4.2
```

**Output explanation:**
- `Client Version: v1.31.0` — confirms `kubectl` itself is installed and reports the exact version, matching what we downloaded.
- `Kustomize Version: v5.4.2` — `kubectl` ships with a bundled tool called Kustomize (for advanced YAML customization, not covered in this lab); this line just confirms it's present.

---

### Task 3 — Install `kind`

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

**Word-by-word explanation (new parts only):**
- `-Lo ./kind` — `-L` again means "follow redirects." `-o ./kind` (lowercase) means "save the output under this **specific** filename," `./kind`, rather than the URL's original name — this is different from Task 2's `-O`, which kept the original name.

**Verify:**
```bash
kind version
```

**Realistic output:**
```
kind v0.24.0 go1.22.6 linux/amd64
```

**Output explanation:**
- `kind v0.24.0` — the installed version of the `kind` tool itself.
- `go1.22.6` — the version of the Go programming language `kind` was built with (irrelevant to using it, just informational).
- `linux/amd64` — the operating system and CPU architecture this specific binary was built for.

---

### Task 4 — Write the cluster configuration file

We want a cluster with **two nodes**: one control plane (the "brain") and one worker (where our applications will actually run). We describe this in a YAML file.

Create a file named `kind-config.yaml`:

```bash
cat > kind-config.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.31.0
  - role: worker
    image: kindest/node:v1.31.0
EOF
```

**Line-by-line YAML explanation:**

| Line | Meaning |
|---|---|
| `kind: Cluster` | This tells the `kind` tool: "this file describes an entire cluster," not a single application. (Note: this `kind:` field is part of *kind the tool's own config format* — it is unrelated to a Kubernetes `Pod`'s `kind` field, which you'll meet in Lab 02. They just happen to share the word "kind.") |
| `apiVersion: kind.x-k8s.io/v1alpha4` | Every structured config file in the Kubernetes ecosystem declares which version of its own format it's using. This line says "I'm using version `v1alpha4` of kind's own configuration schema." |
| `nodes:` | The start of a **list** of nodes this cluster should contain. Everything indented under it, starting with a `-`, is one list item. |
| `  - role: control-plane` | The first node in our list. `role: control-plane` means this node will run the Kubernetes "brain" components (the API server, the scheduler, and more — covered in Lab 03). |
| `    image: kindest/node:v1.31.0` | Which container image to use to simulate this node. `kindest/node` is a special image maintained by the `kind` project that contains a full, working Kubernetes installation inside a container. The `:v1.31.0` tag pins the **exact** Kubernetes version this node will run. |
| `  - role: worker` | The second node in the list. `role: worker` means this node will run your actual applications, not the cluster's brain. |
| `    image: kindest/node:v1.31.0` | Same image, same Kubernetes version, so both nodes are consistent. |

```
   kind-config.yaml describes THIS:

   +-------------------------+     +-------------------------+
   |  role: control-plane     |     |  role: worker             |
   |  image: kindest/node      |     |  image: kindest/node      |
   |         :v1.31.0            |     |         :v1.31.0            |
   +-------------------------+     +-------------------------+
```

---

### Task 5 — Create the cluster

```bash
kind create cluster --name my-first-cluster --config kind-config.yaml
```

**Word-by-word explanation:**
- `kind` — the program.
- `create` — a subcommand meaning "make a new thing."
- `cluster` — the type of thing to create (as opposed to, say, deleting one).
- `--name my-first-cluster` — a flag giving this specific cluster a name, `my-first-cluster`, so you can have several different clusters at once and refer to each by name.
- `--config kind-config.yaml` — a flag pointing `kind` at the file we just wrote, telling it exactly what nodes to build, instead of using kind's own single-node default.

**Realistic output:**
```
Creating cluster "my-first-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.31.0)
 ✓ Preparing nodes
 ✓ Writing configuration
 ✓ Starting control-plane
 ✓ Installing CNI
 ✓ Installing StorageClass
 ✓ Joining worker nodes
Set kubectl context to "kind-my-first-cluster"
You can now use your cluster with:

kubectl cluster-info --context kind-my-first-cluster
```

**Output explanation, line by line:**
- `Ensuring node image` — kind is downloading the `kindest/node:v1.31.0` container image if you don't already have it locally.
- `Preparing nodes` — kind is creating the actual Docker containers that will act as your two nodes.
- `Writing configuration` — kind is generating the internal Kubernetes configuration files (certificates, cluster settings) needed to bring the cluster to life.
- `Starting control-plane` — the "brain" components (covered fully in Lab 03) are being started on the control-plane node.
- `Installing CNI` — CNI stands for **Container Network Interface**. This installs the plugin responsible for giving every Pod (Lab 02) its own IP address and letting Pods talk to each other. Without this, nothing could network at all.
- `Installing StorageClass` — sets up a default way for the cluster to provide storage to applications later (relevant starting in Lab 06+).
- `Joining worker nodes` — the worker node is told "you belong to this cluster now," and starts accepting instructions from the control plane.
- `Set kubectl context to "kind-my-first-cluster"` — **this is important.** `kubectl` can talk to many different clusters. This line means `kubectl` has automatically been configured to point at this brand-new cluster by default, so your very next command will "just work" against it.

This step usually takes 30–90 seconds on a typical laptop.

---

### Task 6 — Verify the cluster is alive

```bash
kubectl cluster-info
```

**Word-by-word explanation:**
- `kubectl` — the program.
- `cluster-info` — a subcommand that asks the cluster to report its own basic addresses and status.

**Realistic output:**
```
Kubernetes control plane is running at https://127.0.0.1:41231
CoreDNS is running at https://127.0.0.1:41231/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

**Output explanation:**
- `Kubernetes control plane is running at https://127.0.0.1:41231` — confirms the API server (the front door to your entire cluster — you'll learn this term properly in Lab 03) is up and reachable at this local address. The port number will differ on your machine; that's normal and expected.
- `CoreDNS is running at ...` — confirms the cluster's internal DNS system (which lets applications find each other by name — covered in a later networking lab) is also up.

Now let's look at the nodes themselves:

```bash
kubectl get nodes
```

**Word-by-word explanation:**
- `kubectl` — the program.
- `get` — a subcommand meaning "list/read" — one of the most common subcommands you'll ever type.
- `nodes` — the type of thing to list. Note this is **plural** — `kubectl get` always takes the plural (or sometimes shortened) form of the resource type.

**Realistic output:**
```
NAME                              STATUS   ROLES           AGE   VERSION
my-first-cluster-control-plane    Ready    control-plane   90s   v1.31.0
my-first-cluster-worker           Ready    <none>          75s   v1.31.0
```

**Output explanation, column by column:**
- `NAME` — the unique name Kubernetes gave each node. Notice it's built from your cluster's name (`my-first-cluster`) plus the role.
- `STATUS` — whether the node is healthy and able to run workloads. `Ready` is what you want to see. If it says `NotReady`, something is still starting up or broken.
- `ROLES` — what job this node performs. `control-plane` runs the cluster's brain; `<none>` (literally the text `<none>`) means this is an ordinary worker node with no special role.
- `AGE` — how long ago this node joined the cluster, in a human-readable format (`90s` = 90 seconds, later you'll see `5m`, `3h`, `10d`, etc.).
- `VERSION` — the exact Kubernetes version running on that node. Both should read `v1.31.0`, matching what we specified in `kind-config.yaml`.

For a more detailed view:

```bash
kubectl get nodes -o wide
```

**Word-by-word explanation (new part):**
- `-o wide` — `-o` means "output format." `wide` is one specific format that adds extra columns beyond the default.

**Realistic output:**
```
NAME                              STATUS   ROLES           AGE   VERSION   INTERNAL-IP   OS-IMAGE                       KERNEL-VERSION      CONTAINER-RUNTIME
my-first-cluster-control-plane    Ready    control-plane   2m    v1.31.0   172.18.0.3    Debian GNU/Linux 12 (bookworm) 5.15.0-119-generic  containerd://1.7.18
my-first-cluster-worker           Ready    <none>          2m    v1.31.0   172.18.0.2    Debian GNU/Linux 12 (bookworm) 5.15.0-119-generic  containerd://1.7.18
```

**New column explanations:**
- `INTERNAL-IP` — the private IP address this node uses to communicate with other nodes inside the cluster.
- `OS-IMAGE` — the operating system running inside this node's container.
- `KERNEL-VERSION` — the Linux kernel version underneath (inherited from your actual laptop, since these are containers, not full VMs).
- `CONTAINER-RUNTIME` — the software actually responsible for running containers on this node. Here it's `containerd`, one of the most common container runtimes used in real production Kubernetes clusters.

Finally, check what's already running inside your brand-new cluster, even though you haven't deployed anything yourself yet:

```bash
kubectl get pods -A
```

**Word-by-word explanation:**
- `pods` — the smallest deployable unit in Kubernetes (you'll create your own in Lab 02 — for now, just know the cluster itself uses Pods internally too).
- `-A` — short for `--all-namespaces`. A **namespace** is a way of organizing resources into separate groups (covered fully in a later lab). This flag says "don't just show me my own stuff — show me everything, everywhere, including the cluster's own internal system components."

**Realistic output:**
```
NAMESPACE            NAME                                                      READY   STATUS    RESTARTS   AGE
kube-system          coredns-7db6d8ff4d-8f2xk                                  1/1     Running   0          3m
kube-system          coredns-7db6d8ff4d-vqz9p                                  1/1     Running   0          3m
kube-system          etcd-my-first-cluster-control-plane                       1/1     Running   0          3m
kube-system          kindnet-4bxsk                                             1/1     Running   0          3m
kube-system          kindnet-p7wjq                                             1/1     Running   0          3m
kube-system          kube-apiserver-my-first-cluster-control-plane             1/1     Running   0          3m
kube-system          kube-controller-manager-my-first-cluster-control-plane    1/1     Running   0          3m
kube-system          kube-proxy-8x2mn                                          1/1     Running   0          3m
kube-system          kube-proxy-znxxr                                         1/1     Running   0          3m
kube-system          kube-scheduler-my-first-cluster-control-plane             1/1     Running   0          3m
local-path-provisioner   local-path-provisioner-6795b5f9d8-4wq2x               1/1     Running   0          3m
```

**Output explanation, column by column:**
- `NAMESPACE` — which group this Pod belongs to. `kube-system` is a special namespace reserved for the cluster's own internal machinery — you will never put your own applications here.
- `NAME` — the unique name of each Pod. The random-looking suffix (like `-8f2xk`) is auto-generated so multiple copies can coexist without name collisions.
- `READY` — how many containers inside the Pod are up and healthy, out of how many total. `1/1` means "1 out of 1 containers ready" — fully healthy.
- `STATUS` — the Pod's current lifecycle state. `Running` is healthy and normal.
- `RESTARTS` — how many times a container in this Pod has crashed and been automatically restarted. `0` is what you want to see on a freshly-created cluster.
- `AGE` — same meaning as before, how long this Pod has existed.

You don't need to understand every one of these system Pods yet (that's what Lab 03 is for) — the important takeaway right now is: **even an "empty" cluster is already running real software to keep itself alive**, and you can watch all of it with one command.

---

## 6. Verification

Run this single command as your final check for this lab:

```bash
kubectl get nodes && kubectl get pods -A | grep -c Running
```

**Word-by-word explanation:**
- `&&` — a shell operator meaning "run the next command only if the first one succeeded."
- `| grep -c Running` — `|` (pipe) sends the output of `kubectl get pods -A` into the `grep` program instead of printing it directly. `grep -c Running` counts how many lines contain the word `Running`.

**Expected result:** You should see your two `Ready` nodes printed, followed by a number (typically `10` or `11`, depending on your exact kind version) representing how many system Pods are healthily `Running`. If that number is `0`, something is wrong — proceed to the Troubleshooting section below.

---

## 7. Common Errors

**Error 1:**
```
ERROR: failed to create cluster: node(s) already exist for a cluster with the name "my-first-cluster"
```
**Meaning:** You already have a cluster with this exact name (perhaps from a previous, earlier attempt).

**Error 2:**
```
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```
**Meaning:** Docker itself isn't running, or your user isn't in the `docker` group yet.

**Error 3:**
```
The connection to the server 127.0.0.1:41231 was refused - did you specify the right host or port?
```
**Meaning:** `kubectl` can't reach the cluster — either the cluster isn't fully started yet, or `kubectl`'s context is pointing at the wrong (or a deleted) cluster.

---

## 8. Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `node(s) already exist` | A cluster with that name already exists | Run `kind delete cluster --name my-first-cluster`, then retry Task 5 |
| `Cannot connect to the Docker daemon` | Docker isn't running | Run `sudo systemctl start docker`, confirm with `docker ps` |
| `Cannot connect to the Docker daemon` (after starting Docker) | Your user isn't in the `docker` group yet | Run `newgrp docker` or fully log out and back in |
| `connection ... was refused` | Wrong kubectl context | Run `kubectl config get-contexts` to see all known clusters, then `kubectl config use-context kind-my-first-cluster` |
| Node stuck at `NotReady` for more than 2 minutes | CNI still initializing, or genuinely stuck | Wait 60 more seconds and re-check; if still stuck, run `kubectl describe node <node-name>` and read the `Conditions` section at the bottom for a specific reason |
| `kind: command not found` after install | `/usr/local/bin` isn't on your `PATH` | Run `echo $PATH` to check, or simply run kind using its full path: `/usr/local/bin/kind version` |

**General debugging habit to build starting now:** whenever something in Kubernetes looks broken, `kubectl describe <resource-type> <resource-name>` is almost always your first move — it shows a detailed, chronological "Events" section at the bottom explaining exactly what Kubernetes has tried to do and what went wrong. You'll use this constantly starting in Lab 02.

---

## 9. Common Mistakes

- **Forgetting to log out/in after adding yourself to the `docker` group.** The group change doesn't apply to your *already-open* terminal session.
- **Running `kind create cluster` without `--config`.** This silently creates a single-node cluster instead of the two-node one this lab intends — always double-check you passed the config file if you meant to.
- **Assuming node `AGE` resets if you restart your laptop.** Kind clusters are containers; if your laptop reboots, Docker (and therefore your cluster) needs to be started again, but the cluster's internal `AGE` clock keeps counting from original creation, not from your laptop's last boot.
- **Confusing `kind` (the tool) with `kind:` (the YAML field).** They're spelled the same but are two completely unrelated things — one is a CLI program, the other is a YAML key you'll see constantly in real Kubernetes manifests starting in Lab 02.

---

## 10. Challenge Exercises

1. Delete this cluster (`kind delete cluster --name my-first-cluster`) and recreate it with **three** worker nodes instead of one, by editing `kind-config.yaml` yourself.
2. Create a **second**, completely separate cluster named `second-cluster` while your first one is still running, then use `kubectl config get-contexts` to see both listed, and practice switching between them with `kubectl config use-context`.
3. Without looking back at this lab, write out from memory the exact three-command sequence to check: (a) that Docker is running, (b) that your cluster's nodes are healthy, and (c) how many system Pods exist.
4. Run `kubectl get nodes -o yaml` (a new flag you haven't used yet) and see if you can find the `Ready` status buried inside the much larger raw output — this previews what "real" Kubernetes data looks like underneath the friendly table view.

---

## 11. Quiz

**1. What does the `-A` flag mean in `kubectl get pods -A`?**
a) Show only active Pods
b) Show Pods from all namespaces
c) Show Pods in alphabetical order
d) Auto-restart failed Pods

**2. In `kind-config.yaml`, what does `role: control-plane` mean for that node?**
a) It's the only node that can be deleted
b) It runs the cluster's core "brain" components
c) It runs your applications only
d) It's a backup node used only during failures

**3. What does `STATUS: Ready` in `kubectl get nodes` output tell you?**
a) The node is for sale
b) The node is healthy and able to run workloads
c) The node has finished downloading its OS
d) The node is currently idle with zero Pods

**4. What is CNI responsible for, based on this lab?**
a) Storing cluster secrets
b) Giving Pods networking and IP addresses
c) Scheduling Pods onto nodes
d) Backing up the cluster

**5. Why did `kubectl cluster-info` immediately work after `kind create cluster` with no extra configuration from you?**
a) Kubernetes has no authentication by default
b) `kind` automatically set your kubectl context to point at the new cluster
c) `cluster-info` doesn't actually need a real cluster
d) It didn't — the example output was fabricated

**Answers:** 1-b, 2-b, 3-b, 4-b, 5-b

---

## 12. Summary

In this lab, you:

- Learned what a Kubernetes cluster fundamentally is: a group of nodes working together, managed declaratively.
- Installed `kubectl` (the client you use to talk to any cluster) and `kind` (a tool that builds real, local Kubernetes clusters using Docker containers as nodes).
- Wrote and understood, line by line, a cluster configuration file describing a 2-node cluster running Kubernetes v1.31.0.
- Created that real cluster and watched `kind` bring up networking (CNI), storage defaults, and the control plane.
- Ran your first `kubectl` commands — `cluster-info`, `get nodes`, `get nodes -o wide`, and `get pods -A` — and learned to read every single column in their output.
- Practiced diagnosing the most common setup failures.

You now have a real, disposable Kubernetes playground running on your own machine, and the foundational vocabulary (node, control plane, worker, CNI, namespace, Pod) to keep building on it.

**Next: Lab 02 — Pods: The Smallest Deployable Unit in Kubernetes**, where you'll create, inspect, and destroy your very first application running inside this cluster.
