# Kubernetes Training Manual

## Volume 1 — Kubernetes Fundamentals

**A Self-Study Handbook for Beginners to Intermediate Learners**

---

### Copyright Notice

© 2026. This training manual is an original work created for self-study purposes. All commands, YAML examples, diagrams, and explanations in this document have been written from scratch. Kubernetes®, Docker®, and related marks are trademarks of their respective owners (The Linux Foundation, Docker Inc.). This manual is not affiliated with or endorsed by any of these organizations.

---

### About This Volume

Volume 1 introduces the core ideas behind Kubernetes: what it is, why it exists, how it is built internally, and how to get a cluster running on your own laptop using two popular local tools — Kind and Minikube. By the end of this volume you will be comfortable with `kubectl`, understand the lifecycle of a Pod, and be able to organize resources using labels, selectors, annotations, and namespaces.

**Prerequisites for this volume:** Basic Linux command-line skills (navigating directories, editing files, running commands with `sudo`), a computer with at least 4 GB of free RAM, and a Ubuntu 22.04 (or similar) machine or virtual machine.

---

## Table of Contents — Volume 1

- Lab 1 — Introduction to Kubernetes
- Lab 2 — Kubernetes Architecture
- Lab 3 — Installing Kind
- Lab 4 — Installing Minikube
- Lab 5 — kubectl Basics
- Lab 6 — Creating Your First Pod
- Lab 7 — Understanding Pod Lifecycle
- Lab 8 — Labels, Selectors and Annotations
- Lab 9 — Namespaces
- Lab 10 — Mini Project
- Volume 1 Review, Exam, and Reference

---

# Lab 1 — Introduction to Kubernetes

**Estimated Duration:** 45 minutes
**Difficulty Level:** Beginner

### Learning Objectives
By the end of this lab you will be able to:
1. Explain what Kubernetes is and the problem it solves.
2. Describe the difference between a container, a container image, and an orchestrator.
3. List the main benefits Kubernetes provides in production environments.
4. Identify the major alternatives to Kubernetes and when they might be used instead.

### Prerequisites
- Basic familiarity with what Docker or containers are (not required to have Docker installed yet).
- No prior Kubernetes knowledge required.

### Theory

Before containers, most applications ran directly on physical or virtual servers. Every server needed its own operating system, its own set of installed dependencies, and its own manual configuration. Moving an application from a developer's laptop to a test server, and then to production, often broke because the environments were never perfectly identical. This became known informally as "it works on my machine" syndrome.

Containers solved the environment problem. A container packages an application together with everything it needs to run — libraries, binaries, configuration files — into a single, portable unit called a container image. Because the image contains its own filesystem, an application behaves the same way regardless of which machine runs it, as long as that machine has a compatible container runtime.

Docker made containers popular by giving developers a simple way to build, share, and run these images. But running a handful of containers by hand with `docker run` does not scale. Real systems need many more capabilities:

- **Scheduling** — deciding which physical machine should run which container, based on available CPU, memory, and other constraints.
- **Self-healing** — automatically restarting a container that crashes, or moving it to a healthy machine if the one it was running on fails.
- **Scaling** — increasing or decreasing the number of running copies of an application based on load.
- **Service discovery and load balancing** — letting containers find and talk to each other, and spreading traffic across multiple copies of the same application.
- **Rolling updates and rollbacks** — deploying new versions of an application without downtime, and reverting safely if something goes wrong.
- **Configuration and secret management** — supplying configuration data and sensitive values (passwords, API keys) to containers without baking them into the image.

A tool that provides all of these capabilities is called a **container orchestrator**. Kubernetes, originally designed at Google and now maintained by the Cloud Native Computing Foundation (CNCF), is the most widely used container orchestrator in the industry. The name comes from the Greek word for "helmsman" — the person who steers a ship — which reflects its role steering many containers across many machines.

At a very high level, you describe to Kubernetes the **desired state** of your application — for example, "I want three copies of this web server running at all times, each with 256 MB of memory." Kubernetes then continuously works to make the **actual state** of the cluster match that desired state. This declarative approach, rather than issuing individual imperative commands, is the foundational idea behind almost everything Kubernetes does.

**Kubernetes vs. alternatives:** Docker Swarm is simpler to learn but has a much smaller ecosystem and is rarely used in new production systems today. Nomad (by HashiCorp) is lightweight and flexible but has smaller community adoption than Kubernetes. Managed platforms like AWS ECS are simpler within their own cloud but lock you into that provider. Kubernetes is more complex to learn than these alternatives, but it is cloud-agnostic, has the largest ecosystem of tools and job postings, and is the de facto industry standard.

### Architecture Diagram

```
        Developer writes desired state (YAML)
                        |
                        v
              +-------------------+
              |     Kubernetes    |
              |      Cluster      |
              +-------------------+
                        |
        continuously reconciles actual state
                        |
                        v
        +---------------------------------+
        |  Running containers across many  |
        |  machines, self-healing, scaled  |
        |  and load-balanced automatically |
        +---------------------------------+
```

### Installation

No installation is required for this lab. In Lab 3 and Lab 4 you will install Kind and Minikube to create local clusters.

### Hands-on Practice

Since no cluster exists yet, this lab's hands-on component is conceptual. Open a terminal and check whether Docker is already installed, since both Kind and Minikube in later labs rely on a container runtime:

```bash
docker --version
```

**Purpose:** Confirms whether the Docker Engine is installed and prints its version.
**Syntax:** `docker --version` — no arguments required.
**Expected Output:** Something like `Docker version 24.0.7, build afdd53b`. If you see "command not found," Docker is not yet installed — you will install it in Lab 3.
**Common mistakes:** Confusing Docker Desktop (a GUI application for Mac/Windows) with the Docker Engine CLI needed on Linux servers.
**Real-world usage:** Every CI/CD pipeline and Kubernetes node needs a working container runtime; checking its version is a routine first troubleshooting step.

### YAML Files

Not applicable to this lab — you will write your first YAML manifest in Lab 6.

### Expected Output

N/A for this conceptual lab.

### Validation Steps

Confirm your own understanding by answering: "What is the difference between a container and an orchestrator?" You should be able to state that a container packages an application, while an orchestrator manages many containers across many machines.

### Common Errors

Not applicable — no cluster commands were run in this lab.

### Troubleshooting

Not applicable.

### Practice Tasks
1. Write, in your own words, a two-sentence definition of Kubernetes.
2. List three problems that existed before containers became popular.
3. Name two Kubernetes alternatives and one reason someone might choose each instead of Kubernetes.
4. Explain the difference between "desired state" and "actual state."
5. Research and write down which company originally created Kubernetes, and which organization maintains it today.

### Challenge Lab
Write a short (150-word) explanation, suitable for a non-technical manager, of why your company might want to adopt Kubernetes. Focus on business benefits (uptime, faster releases) rather than technical jargon.

### Best Practices
- Understand the "why" before the "how" — Kubernetes has a steep learning curve, and skipping fundamentals leads to confusion later.
- Do not assume Kubernetes is required for every project; small applications with light traffic may not need this level of orchestration.

### Interview Questions
1. **Q: What is Kubernetes?** A: An open-source container orchestration platform that automates deployment, scaling, and management of containerized applications.
2. **Q: What problem does Kubernetes solve?** A: It automates scheduling, scaling, self-healing, and networking of containers across a cluster of machines, replacing manual container management.
3. **Q: Who maintains Kubernetes today?** A: The Cloud Native Computing Foundation (CNCF).
4. **Q: What is the difference between a container image and a running container?** A: An image is a static, packaged filesystem and set of instructions; a container is a running instance of that image.
5. **Q: What does "declarative" mean in the context of Kubernetes?** A: You describe the desired end state, and the system figures out how to achieve and maintain it, rather than you issuing step-by-step commands.
6. **Q: Name two Kubernetes alternatives.** A: Docker Swarm and HashiCorp Nomad.
7. **Q: What does self-healing mean in Kubernetes?** A: The cluster automatically restarts or reschedules failed containers to maintain the desired state.
8. **Q: Is Kubernetes tied to a specific cloud provider?** A: No — it is cloud-agnostic and can run on-premises or on any major cloud provider.
9. **Q: What organization originally built Kubernetes?** A: Google, based on internal experience running a system called Borg.
10. **Q: Why might a small startup choose not to use Kubernetes?** A: Because of its operational complexity and learning curve, which may not be justified for a small, low-traffic application.

### Quiz
1. Kubernetes was originally developed by: (a) Microsoft (b) Google (c) Amazon (d) Red Hat — **Answer: b**
2. Which organization maintains Kubernetes today? (a) Apache (b) CNCF (c) ISO (d) IEEE — **Answer: b**
3. What is a container orchestrator responsible for? (a) Writing application code (b) Scheduling and managing containers across machines (c) Compiling source code (d) Designing databases — **Answer: b**
4. What does "desired state" mean? (a) The state a user wishes existed but cannot configure (b) The configuration you declare that Kubernetes works to achieve (c) A deprecated Kubernetes feature (d) The state of the underlying hardware — **Answer: b**
5. Which of the following is NOT a Kubernetes alternative? (a) Docker Swarm (b) Nomad (c) AWS ECS (d) Git — **Answer: d**
6. What does "self-healing" refer to? (a) Automatic OS patching (b) Automatically restarting/replacing failed containers (c) Automatic code review (d) Encrypting secrets — **Answer: b**
7. A container image contains: (a) Only source code (b) The application plus its dependencies and filesystem (c) Only configuration files (d) A virtual machine — **Answer: b**
8. Kubernetes is: (a) Tied only to Google Cloud (b) Cloud-agnostic (c) Only usable with AWS (d) A programming language — **Answer: b**
9. What was Kubernetes based on internally at Google? (a) Borg (b) Omega (c) Spanner (d) Colossus — **Answer: a**
10. Which of these is a core Kubernetes capability? (a) Rolling updates (b) Video editing (c) Spreadsheet calculation (d) Email routing — **Answer: a**

### Homework
Research one real company (search publicly available case studies) that migrated to Kubernetes. Write a half-page summary covering: what problem they had before, what changed after adopting Kubernetes, and one challenge they faced during migration.

### Summary
Kubernetes is a container orchestration platform that automates scheduling, scaling, healing, and networking of containerized applications across a cluster of machines. It follows a declarative model: you describe desired state, and Kubernetes continuously reconciles the cluster to match it. Understanding this mental model is essential before diving into Kubernetes' architecture, which is the subject of Lab 2.

---

# Lab 2 — Kubernetes Architecture

**Estimated Duration:** 60 minutes
**Difficulty Level:** Beginner

### Learning Objectives
1. Identify the components of the Kubernetes control plane.
2. Identify the components running on every worker node.
3. Explain how the components communicate to keep the cluster in its desired state.
4. Describe the role of etcd as the cluster's source of truth.

### Prerequisites
- Completion of Lab 1.

### Theory

A Kubernetes cluster is divided into two categories of machines: the **control plane** (sometimes still called the "master") and **worker nodes**. The control plane makes global decisions about the cluster — for example, deciding which node should run a new Pod — while worker nodes actually run your application containers.

**Control plane components:**

- **kube-apiserver** — The front door to the cluster. Every interaction, whether from `kubectl`, a controller, or another component, goes through the API server via REST calls over HTTPS. It validates requests and writes the resulting state to etcd.
- **etcd** — A distributed, consistent key-value store that holds the entire state of the cluster: every Pod, every Service, every Secret. If etcd is lost without a backup, the cluster's configuration is lost. This is why etcd backup (covered in Volume 5) is a critical production skill.
- **kube-scheduler** — Watches for newly created Pods that have no node assigned, and selects the best node for them based on resource requirements, constraints, and policies.
- **kube-controller-manager** — Runs controller loops that watch the cluster state and drive it toward the desired state. Examples include the Node controller (detects when nodes go down) and the ReplicaSet controller (ensures the right number of Pod replicas exist).
- **cloud-controller-manager** — Present only when running on a cloud provider; integrates the cluster with cloud-specific services like load balancers and storage volumes.

**Worker node components:**

- **kubelet** — An agent that runs on every node. It receives Pod specifications from the API server and ensures the described containers are actually running and healthy.
- **kube-proxy** — Maintains network rules on each node, allowing network communication to Pods from inside and outside the cluster.
- **Container runtime** — The software that actually runs containers, such as containerd or CRI-O. Modern Kubernetes uses the Container Runtime Interface (CRI) to support multiple runtimes.

The overall flow when you create a Pod: `kubectl` sends a request to the **API server** → the API server validates and stores it in **etcd** → the **scheduler** notices an unscheduled Pod and assigns it to a node → the **kubelet** on that node notices the assignment and instructs the **container runtime** to start the container → **kube-proxy** ensures the Pod is reachable over the network.

### Architecture Diagram

```
                         CONTROL PLANE
   +-----------------------------------------------------+
   |                                                       |
   |   kubectl  --->  kube-apiserver <---> etcd            |
   |                        ^  ^                           |
   |                        |  |                           |
   |             kube-scheduler  kube-controller-manager    |
   |                                                       |
   +-----------------------------------------------------+
                            |
              (assigns/updates Pod specs over API)
                            |
        +-------------------+-------------------+
        |                                       |
   WORKER NODE 1                          WORKER NODE 2
   +----------------+                     +----------------+
   | kubelet        |                     | kubelet        |
   | kube-proxy     |                     | kube-proxy     |
   | container      |                     | container      |
   | runtime        |                     | runtime        |
   |  [Pod] [Pod]   |                     |  [Pod]         |
   +----------------+                     +----------------+
```

### Installation
No installation in this lab; architecture is explored conceptually and, later, by inspecting a running cluster after Lab 3/4.

### Hands-on Practice

Once you complete Lab 3 or 4 and have a running cluster, return here and run:

```bash
kubectl get componentstatuses
```

**Purpose:** Lists the health of legacy control plane components (deprecated in newer versions but still instructive).
**Syntax:** No arguments needed.
**Expected Output:** A table showing `Healthy` next to `scheduler`, `controller-manager`, and `etcd-0` (output format varies by version).
**Common mistakes:** Running this before a cluster exists, resulting in a "connection refused" error.
**Real-world usage:** Quick sanity check that control plane components are up, though `kubectl get nodes` and `kubectl cluster-info` are more commonly used today.

```bash
kubectl get nodes -o wide
```
**Purpose:** Lists all nodes in the cluster along with their roles, versions, and internal IPs.
**Syntax:** `-o wide` adds extra columns such as OS image and container runtime.
**Expected Output:** A table with columns NAME, STATUS, ROLES, AGE, VERSION, INTERNAL-IP, EXTERNAL-IP, OS-IMAGE, KERNEL-VERSION, CONTAINER-RUNTIME.
**Common mistakes:** Forgetting `-o wide` and missing runtime/IP details needed for troubleshooting.
**Real-world usage:** The first command most engineers run when diagnosing "why isn't my Pod scheduling" issues.

### YAML Files
Not applicable — architecture is inspected via commands, not manifests, in this lab.

### Expected Output
```
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   10m   v1.29.0
kind-worker          Ready    <none>          10m   v1.29.0
```

### Validation Steps
Run `kubectl get nodes` and confirm at least one node shows `STATUS: Ready`. This confirms the kubelet on that node is successfully communicating with the API server.

### Common Errors
- **"The connection to the server ... was refused"** — no cluster is running, or your kubeconfig points to the wrong cluster.
- **Node stuck in `NotReady`** — usually a container runtime or networking (CNI) problem on that node.

### Troubleshooting
For a refused connection, verify a cluster exists (`kind get clusters` or `minikube status`) and that `~/.kube/config` points to it. For a `NotReady` node, run `kubectl describe node <name>` and inspect the `Conditions` section for clues such as `MemoryPressure` or `NetworkUnavailable`.

### Practice Tasks
1. List all five control plane components and one-line descriptions of each.
2. List the three components that run on every worker node.
3. Explain, step by step, what happens between running `kubectl apply` and a container actually starting.
4. Explain why losing etcd without a backup is catastrophic.
5. Draw (on paper) your own version of the architecture diagram from memory.

### Challenge Lab
Once you have a cluster running (Lab 3/4), run `kubectl get pods -n kube-system` and identify which Pods correspond to which control plane components you learned about in this lab.

### Best Practices
- In production, run etcd on dedicated, fast (SSD-backed) disks — it is highly sensitive to write latency.
- Always run an odd number of control plane replicas (typically 3 or 5) in production for etcd quorum.

### Interview Questions
1. **Q: What does the kube-apiserver do?** A: It is the single entry point for all cluster operations, validating and processing REST requests and persisting state to etcd.
2. **Q: What is etcd?** A: A distributed, consistent key-value store that holds all cluster state.
3. **Q: What does kube-scheduler do?** A: Assigns newly created, unscheduled Pods to suitable nodes based on resource requirements and constraints.
4. **Q: What is the kubelet's role?** A: An agent on every node that ensures containers described in Pod specs are running and healthy.
5. **Q: What does kube-proxy do?** A: Maintains network rules on nodes to enable communication to and from Pods.
6. **Q: What is the container runtime?** A: The underlying software (e.g., containerd, CRI-O) that actually runs containers via the Container Runtime Interface.
7. **Q: Why is etcd backup important?** A: Because etcd holds the entire cluster state; losing it without a backup means losing all configuration and requiring a full rebuild.
8. **Q: What does kube-controller-manager do?** A: Runs control loops (Node controller, ReplicaSet controller, etc.) that drive actual state toward desired state.
9. **Q: What is the cloud-controller-manager for?** A: Integrating the cluster with cloud-provider-specific resources like load balancers and storage.
10. **Q: How would you check if a node is healthy?** A: Run `kubectl get nodes` and look for `STATUS: Ready`, then `kubectl describe node <name>` for details.

### Quiz
1. Which component is the single entry point to the cluster? (a) etcd (b) kube-apiserver (c) kubelet (d) kube-proxy — **b**
2. Where is cluster state stored? (a) kube-scheduler (b) etcd (c) kubelet (d) kubectl config — **b**
3. Which component assigns Pods to nodes? (a) kube-proxy (b) kube-scheduler (c) kubelet (d) etcd — **b**
4. Which component runs on every worker node and manages containers directly? (a) kube-apiserver (b) kubelet (c) etcd (d) kube-controller-manager — **b**
5. Which component maintains network rules on nodes? (a) kube-proxy (b) kube-scheduler (c) etcd (d) kubectl — **a**
6. What type of data store is etcd? (a) Relational SQL database (b) Distributed key-value store (c) Flat file (d) In-memory cache only — **b**
7. What does `kubectl get nodes` show? (a) Pod logs (b) Node status and info (c) Secrets (d) Service endpoints — **b**
8. Which component would you check first if Pods aren't being scheduled? (a) kube-scheduler (b) kube-proxy (c) kubelet logs on a random node (d) etcd backup file — **a**
9. What replaces the term "master" in modern Kubernetes docs? (a) Control plane (b) Head node (c) Primary node (d) Root node — **a**
10. In production, how many control plane replicas is typical for etcd quorum? (a) Any even number (b) 2 (c) An odd number like 3 or 5 (d) Exactly 1 — **c**

### Homework
Draw a full architecture diagram of a 3-node Kubernetes cluster (1 control plane, 2 workers) by hand or in a diagramming tool, labeling every component covered in this lab and the direction of communication between them.

### Summary
A Kubernetes cluster consists of a control plane (API server, etcd, scheduler, controller manager) that makes decisions, and worker nodes (kubelet, kube-proxy, container runtime) that run workloads. Understanding this separation is essential for troubleshooting and for the installation labs that follow.

---

# Lab 3 — Installing Kind

**Estimated Duration:** 45 minutes
**Difficulty Level:** Beginner

### Learning Objectives
1. Install Docker as a prerequisite container runtime.
2. Install the Kind (Kubernetes IN Docker) CLI.
3. Create a local multi-node Kubernetes cluster using Kind.
4. Verify the cluster is healthy.

### Prerequisites
- Ubuntu 22.04 (or similar Linux distribution) with `sudo` access.
- Completion of Labs 1–2.

### Theory

Kind ("Kubernetes IN Docker") runs Kubernetes nodes as Docker containers rather than as full virtual machines. This makes it extremely fast to create and destroy clusters, and it is the tool most commonly used in CI pipelines to test Kubernetes manifests before deploying to real infrastructure. Because every "node" is really just a container, Kind is lightweight, but it is not intended for production use — it exists purely for local development and testing.

### Architecture Diagram

```
   Host Machine (your laptop/VM)
   +----------------------------------------------------+
   |  Docker Engine                                       |
   |   +----------------------+   +----------------------+ |
   |   | Container:            |   | Container:            | |
   |   | kind-control-plane    |   | kind-worker           | |
   |   | (acts as a full node) |   | (acts as a full node) | |
   |   +----------------------+   +----------------------+ |
   +----------------------------------------------------+
```

### Installation

**Step 1 — Install Docker on Ubuntu:**
```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
```
Log out and log back in (or run `newgrp docker`) for the group change to apply.

**Step 2 — Install kubectl:**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

**Step 3 — Install Kind:**
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

**Step 4 — Create a two-node cluster:**
```bash
cat <<EOF | kind create cluster --name my-cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
EOF
```

### Hands-on Practice

```bash
kind get clusters
```
**Purpose:** Lists all Kind clusters currently running on your machine.
**Syntax:** No arguments required.
**Expected Output:** `my-cluster`
**Common mistakes:** Forgetting the cluster name and assuming Kind clusters share a namespace with Minikube — they don't; they are entirely separate tools.
**Real-world usage:** Useful when several test clusters exist simultaneously during development.

```bash
kubectl cluster-info --context kind-my-cluster
```
**Purpose:** Confirms `kubectl` can reach the API server and shows the endpoint URLs for the control plane and DNS.
**Syntax:** `--context` selects which cluster/config entry to use, since your kubeconfig may contain several.
**Expected Output:** Lines showing `Kubernetes control plane is running at https://127.0.0.1:PORT`.
**Common mistakes:** Omitting `--context` when multiple clusters exist, accidentally querying the wrong one.
**Real-world usage:** The standard first command to run against any new or unfamiliar cluster.

### YAML Files

The cluster configuration YAML used above:
```yaml
kind: Cluster                          # Tells Kind this file describes a cluster (not a Pod)
apiVersion: kind.x-k8s.io/v1alpha4     # Kind's own config API version, unrelated to Kubernetes core APIs
nodes:
- role: control-plane                  # This node will run the control plane components
- role: worker                         # This node will run only workloads
```

### Expected Output
```
Creating cluster "my-cluster" ...
  Ensuring node image (kindest/node:v1.29.0)
  Preparing nodes
  Writing configuration
  Starting control-plane
  Installing CNI
  Installing StorageClass
  Joining worker nodes
Set kubectl context to "kind-my-cluster"
```

### Validation Steps
```bash
kubectl get nodes
```
Both `my-cluster-control-plane` and `my-cluster-worker` should show `STATUS: Ready`.

### Common Errors
- **"Cannot connect to the Docker daemon"** — Docker is not running or your user isn't in the `docker` group.
- **"port is already allocated"** — another cluster or service is using the same host port; delete unused clusters with `kind delete cluster --name <name>`.
- **Nodes stuck `NotReady` after creation** — CNI plugin still initializing; wait 30–60 seconds and re-check.

### Troubleshooting
Run `sudo systemctl status docker` to confirm the Docker daemon is active. If your user was just added to the `docker` group, you must start a new shell session. For port conflicts, run `docker ps` to see what is using the port, or delete and recreate the cluster with a different config.

### Practice Tasks
1. Create a Kind cluster with three worker nodes instead of one.
2. Delete the cluster and recreate it, timing how long each step takes.
3. Run `docker ps` while the cluster is running and identify which containers correspond to which Kind nodes.
4. Change the cluster name and confirm `kubectl config get-contexts` reflects the new name.
5. Export the kubeconfig for your cluster to a separate file using `kind get kubeconfig --name my-cluster > my-cluster.yaml`.

### Challenge Lab
Create a Kind cluster with one control-plane node and three worker nodes, label each worker node differently (`kubectl label node <name> role=web`, `role=db`, `role=cache`), and confirm the labels with `kubectl get nodes --show-labels`.

### Best Practices
- Delete unused Kind clusters (`kind delete cluster --name <name>`) to free up system resources.
- Never use Kind clusters for anything resembling production traffic — they are for local development and CI only.
- Pin a specific Kind node image version in your config file for reproducible test environments.

### Interview Questions
1. **Q: What is Kind?** A: A tool that runs Kubernetes clusters using Docker containers as nodes, mainly for local testing and CI.
2. **Q: Is Kind suitable for production?** A: No, it is designed exclusively for local development and testing.
3. **Q: What underlying technology does Kind depend on?** A: Docker (or a compatible container runtime).
4. **Q: How do you create a multi-node Kind cluster?** A: By supplying a YAML config file with multiple `nodes` entries to `kind create cluster --config`.
5. **Q: How do you list existing Kind clusters?** A: `kind get clusters`.
6. **Q: How do you delete a Kind cluster?** A: `kind delete cluster --name <name>`.
7. **Q: Why might a "port already allocated" error occur?** A: Another running cluster or Docker container is already using that host port.
8. **Q: What command confirms kubectl can reach a cluster?** A: `kubectl cluster-info`.
9. **Q: What role does the `role: control-plane` field play in a Kind config?** A: It designates that specific node to run control plane components.
10. **Q: Why is Kind popular in CI pipelines?** A: It spins up and tears down full Kubernetes clusters very quickly, entirely inside Docker, without needing real VMs.

### Quiz
1. Kind stands for: (a) Kubernetes Is Not Docker (b) Kubernetes IN Docker (c) Kernel Independent Node Daemon (d) None of the above — **b**
2. Kind nodes are implemented as: (a) Virtual machines (b) Docker containers (c) Bare metal servers (d) Cloud instances — **b**
3. Is Kind recommended for production? (a) Yes (b) No — **b**
4. Which command lists running Kind clusters? (a) `kind ls` (b) `kind get clusters` (c) `kind list` (d) `kubectl get clusters` — **b**
5. What must be installed before Kind? (a) Minikube (b) Docker (c) Helm (d) Terraform — **b**
6. Which file format is used for Kind cluster configuration? (a) JSON only (b) YAML (c) TOML (d) INI — **b**
7. What command deletes a Kind cluster? (a) `kind remove` (b) `kind delete cluster --name <name>` (c) `kind stop` (d) `kubectl delete cluster` — **b**
8. A "port already allocated" error usually means: (a) Out of disk space (b) A port conflict with another running container (c) DNS failure (d) Insufficient RAM — **b**
9. What does `kubectl cluster-info` confirm? (a) Node CPU usage (b) That kubectl can reach the API server (c) Pod logs (d) Docker version — **b**
10. Kind is most commonly used for: (a) Production workloads (b) Local development and CI testing (c) Mobile app deployment (d) Database hosting — **b**

### Homework
Install Kind on a fresh Ubuntu VM (or container) from scratch, following only the commands in this lab, and write down any deviations or errors you encountered along with how you resolved them.

### Summary
Kind provides fast, disposable Kubernetes clusters running as Docker containers, ideal for local development and CI testing. You installed Docker, kubectl, and Kind, then created and validated a multi-node cluster. Lab 4 introduces Minikube, an alternative local cluster tool with a different underlying approach (VM or container-based single-node clusters).

---

# Lab 4 — Installing Minikube

**Estimated Duration:** 45 minutes
**Difficulty Level:** Beginner

### Learning Objectives
1. Install Minikube and a compatible driver.
2. Start a local single-node (or multi-node) Kubernetes cluster.
3. Use the Minikube dashboard and addon system.
4. Compare Minikube with Kind to choose the right tool for a given task.

### Prerequisites
- Completion of Lab 3 (Docker and kubectl already installed).

### Theory

Minikube is another local Kubernetes tool, but it takes a different approach than Kind: it can run the cluster inside a virtual machine, inside Docker, or using other drivers such as Podman or VirtualBox. Minikube has been around longer than Kind and includes extra conveniences such as a built-in web dashboard and an addon system (for example, enabling an Ingress controller with one command). Many learners use Minikube for hands-on learning and demos, while teams increasingly use Kind for automated testing due to its speed and container-native design. Knowing both tools broadens your options.

### Architecture Diagram

```
   Host Machine
   +----------------------------------------------+
   |  Minikube VM or Docker container ("node")      |
   |   +------------------------------------------+ |
   |   |  Control plane + worker components         | |
   |   |  (single-node cluster by default)           | |
   |   +------------------------------------------+ |
   +----------------------------------------------+
```

### Installation

**Step 1 — Install Minikube:**
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```

**Step 2 — Start a cluster using the Docker driver:**
```bash
minikube start --driver=docker --nodes=2 --cpus=2 --memory=4096
```

**Step 3 — Enable the dashboard addon:**
```bash
minikube addons enable dashboard
minikube addons enable metrics-server
```

### Hands-on Practice

```bash
minikube status
```
**Purpose:** Reports whether the Minikube VM/container, kubelet, and API server are running.
**Syntax:** No arguments needed.
**Expected Output:**
```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```
**Common mistakes:** Assuming `minikube start` failing silently means the cluster is fine — always check `minikube status` afterward.
**Real-world usage:** First diagnostic command whenever `kubectl` commands against Minikube start failing.

```bash
minikube dashboard --url
```
**Purpose:** Prints (without auto-opening a browser) the local URL for the Kubernetes web dashboard.
**Syntax:** `--url` avoids trying to launch a GUI browser, useful on headless servers.
**Expected Output:** `http://127.0.0.1:PORT/api/v1/namespaces/kubernetes-dashboard/...`
**Common mistakes:** Running plain `minikube dashboard` on a server with no GUI, which hangs waiting to open a browser.
**Real-world usage:** Quick visual inspection of cluster resources without memorizing kubectl syntax, useful for demos.

### YAML Files

Minikube itself is configured via CLI flags rather than YAML, but once running, you use ordinary Kubernetes YAML manifests exactly as with any other cluster — Minikube's value is providing you a compliant cluster to apply them to, not a different manifest format.

### Expected Output
```
  minikube v1.33.0 on Ubuntu 22.04
  Using the docker driver based on user configuration
  Starting control plane node minikube in cluster minikube
  Pulling base image ...
  Creating docker container (CPUs=2, Memory=4096MB) ...
  Preparing Kubernetes v1.29.0 on Docker 24.0.7 ...
  Verifying Kubernetes components...
  Enabled addons: storage-provisioner, default-storageclass
  Done! kubectl is now configured to use "minikube" cluster
```

### Validation Steps
Run `kubectl get nodes` and `minikube status`; both should report a healthy, `Ready` cluster.

### Common Errors
- **"driver not found"** — the chosen driver (e.g., `docker`, `virtualbox`) isn't installed.
- **Insufficient resources** — `minikube start` fails if `--memory` or `--cpus` exceed what's available on the host.
- **Stale cluster after host reboot** — run `minikube start` again; Minikube can usually resume the existing cluster.

### Troubleshooting
For driver errors, run `minikube start --driver=docker` explicitly, or install the missing driver. For resource errors, lower `--memory`/`--cpus` or free up resources with `docker system prune`. If the cluster seems broken beyond repair, `minikube delete` followed by `minikube start` gives a clean slate.

### Practice Tasks
1. Start Minikube with the `docker` driver and confirm `kubectl get nodes` shows it as `Ready`.
2. Enable the `metrics-server` addon and run `kubectl top nodes` once data is available.
3. Stop the cluster with `minikube stop` and then resume it with `minikube start`.
4. Compare `minikube start` time versus `kind create cluster` time on your machine.
5. List all available addons with `minikube addons list`.

### Challenge Lab
Start a 3-node Minikube cluster, enable the dashboard and ingress addons, and use `kubectl get pods -A` to identify every addon Pod that was created as a result.

### Best Practices
- Match `--cpus`/`--memory` flags to your actual workload needs; over-provisioning wastes local resources.
- Use `minikube delete --all` periodically to clean up old, unused clusters and profiles.
- Prefer the `docker` driver for speed and consistency unless you specifically need VM-level isolation.

### Interview Questions
1. **Q: What is Minikube?** A: A tool for running a local, single- or multi-node Kubernetes cluster for development and learning.
2. **Q: What drivers can Minikube use?** A: Docker, VirtualBox, Podman, HyperKit, and others depending on the host OS.
3. **Q: How do you check Minikube's status?** A: `minikube status`.
4. **Q: How do you enable an addon?** A: `minikube addons enable <addon-name>`.
5. **Q: What is the Kubernetes dashboard?** A: A web-based UI for viewing and managing cluster resources.
6. **Q: How does Minikube differ from Kind architecturally?** A: Minikube typically runs a full VM or container acting as one node with flexible driver options; Kind runs each "node" as a lightweight Docker container by design, optimized for speed.
7. **Q: How do you stop a Minikube cluster without deleting it?** A: `minikube stop`.
8. **Q: How do you completely remove a Minikube cluster?** A: `minikube delete`.
9. **Q: Why might `minikube start` fail with a resources error?** A: The requested `--memory`/`--cpus` exceed what is available on the host.
10. **Q: Is Minikube meant for production use?** A: No — like Kind, it is intended for local development and learning only.

### Quiz
1. Minikube can use which of the following as a driver? (a) Docker (b) VirtualBox (c) Podman (d) All of the above — **d**
2. Which command checks Minikube's health? (a) `minikube health` (b) `minikube status` (c) `minikube check` (d) `kubectl minikube` — **b**
3. Addons are enabled with: (a) `minikube enable` (b) `minikube addons enable <name>` (c) `kubectl enable addon` (d) `minikube plugin add` — **b**
4. What does `minikube dashboard --url` do? (a) Deletes the dashboard (b) Prints the dashboard URL without opening a browser (c) Installs Minikube (d) Lists nodes — **b**
5. Which command stops (without deleting) a cluster? (a) `minikube pause` (b) `minikube stop` (c) `minikube halt` (d) `minikube suspend` — **b**
6. Which command fully removes a Minikube cluster? (a) `minikube delete` (b) `minikube remove` (c) `minikube destroy` (d) `minikube uninstall` — **a**
7. Minikube configuration for starting a cluster is primarily done via: (a) YAML files only (b) CLI flags (c) A GUI installer (d) Environment variables only — **b**
8. `minikube addons list` shows: (a) Installed Pods (b) Available and enabled addons (c) Node IP addresses (d) etcd backups — **b**
9. Is Minikube production-ready? (a) Yes (b) No — **b**
10. Compared to Kind, Minikube: (a) Cannot run multiple nodes (b) Offers more built-in conveniences like a dashboard and addons (c) Doesn't support kubectl (d) Only works on Windows — **b**

### Homework
Install Minikube, start a cluster, enable at least two addons, and write a short comparison (half a page) of your experience with Minikube versus Kind from Lab 3 — including startup time, ease of use, and which you'd choose for CI versus local learning.

### Summary
Minikube offers a flexible, feature-rich way to run local Kubernetes clusters, complete with a dashboard and addon system. Together with Kind, you now have two ways to spin up local clusters for the rest of this manual. Lab 5 moves on to mastering `kubectl`, the command-line tool you'll use for everything from here forward.

---

# Lab 5 — kubectl Basics

**Estimated Duration:** 60 minutes
**Difficulty Level:** Beginner

### Learning Objectives
1. Understand the structure of a `kubectl` command.
2. Use `get`, `describe`, `logs`, and `exec` to inspect a cluster.
3. Understand `kubectl apply` versus `kubectl create`.
4. Configure `kubectl` autocompletion and aliases for productivity.

### Prerequisites
- A running Kind or Minikube cluster from Lab 3/4.

### Theory

`kubectl` is the primary command-line tool for interacting with a Kubernetes cluster. Every `kubectl` command follows a similar pattern:

```
kubectl <verb> <resource-type> <resource-name> [flags]
```

For example, `kubectl get pods my-pod -n default` reads as: **verb** = get, **resource type** = pods, **resource name** = my-pod, **flag** = `-n default` (namespace).

Common verbs include `get` (list/read), `describe` (detailed human-readable info, including recent events), `create` (imperatively create a resource), `apply` (declaratively create or update a resource from a YAML file), `delete`, `logs` (view container logs), and `exec` (run a command inside a running container).

A crucial distinction: `kubectl create -f file.yaml` will fail if the resource already exists, while `kubectl apply -f file.yaml` will create it if missing or update it if it already exists, using a three-way diff. For this reason, `apply` is almost always preferred in real workflows, especially when manifests are stored in Git and applied repeatedly (a pattern called GitOps).

`kubectl` reads its connection details from a **kubeconfig** file, usually at `~/.kube/config`. This file can contain multiple clusters, users, and contexts, letting one `kubectl` installation talk to many different clusters by switching context.

### Architecture Diagram

```
   You type:  kubectl get pods -n default
                       |
                       v
       kubectl reads ~/.kube/config to find
       the right cluster + credentials
                       |
                       v
          HTTPS request to kube-apiserver
                       |
                       v
       API server reads current state from etcd
                       |
                       v
        Human-readable table printed to terminal
```

### Installation
kubectl was already installed in Lab 3. Optionally enable autocompletion:
```bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

### Hands-on Practice

```bash
kubectl get pods -A
```
**Purpose:** Lists Pods in every namespace (`-A` = `--all-namespaces`).
**Syntax:** `-A` overrides the default namespace scope.
**Expected Output:** A table listing system Pods such as `coredns-...`, `kube-proxy-...`, and `etcd-...`.
**Common mistakes:** Forgetting `-A` and assuming an empty result means the cluster is broken, when really you're just looking at the (empty) `default` namespace.
**Real-world usage:** The most common first command to see everything running cluster-wide.

```bash
kubectl describe node <node-name>
```
**Purpose:** Shows detailed information about a node, including capacity, allocatable resources, conditions, and recent events.
**Syntax:** Requires a specific node name, obtained from `kubectl get nodes`.
**Expected Output:** A multi-section text block ending with an `Events:` table.
**Common mistakes:** Skipping straight to logs when a Pod won't schedule — `describe node` often reveals resource pressure that `describe pod` alone won't show as clearly.
**Real-world usage:** Diagnosing "why won't my Pod schedule onto this node" issues.

```bash
kubectl config get-contexts
kubectl config use-context <context-name>
```
**Purpose:** Lists and switches between clusters/users defined in your kubeconfig.
**Syntax:** `use-context` takes the exact context name shown by `get-contexts`.
**Expected Output:** A table with a `*` marking the current context; after `use-context`, a confirmation message.
**Common mistakes:** Running commands against the wrong cluster because the wrong context is active — always double check with `kubectl config current-context` before destructive operations.
**Real-world usage:** Critical safety habit for engineers who manage multiple clusters (dev, staging, production).

### YAML Files

`kubectl` itself doesn't require YAML for read-only commands, but `apply` uses standard manifests, structured as covered in Lab 6. A minimal example for reference:
```yaml
apiVersion: v1        # Which version of the Kubernetes API this object uses
kind: Namespace       # The type of object being described
metadata:
  name: demo          # The object's unique name
```

### Expected Output
```
NAMESPACE     NAME                            READY   STATUS    RESTARTS   AGE
kube-system   coredns-76f75df574-abcde        1/1     Running   0          15m
kube-system   etcd-my-cluster-control-plane   1/1     Running   0          15m
kube-system   kube-proxy-xyz12                1/1     Running   0          15m
```

### Validation Steps
Confirm `kubectl get pods -A` returns system Pods all showing `STATUS: Running` and `READY: 1/1` (or similar) before proceeding to later labs.

### Common Errors
- **"error: You must be logged in to the server (Unauthorized)"** — kubeconfig credentials expired or misconfigured.
- **"the server doesn't have a resource type ..."** — typo in resource name, or using an API resource not installed in this cluster.
- **Empty output with no error** — you're likely looking in the wrong namespace.

### Troubleshooting
For authorization errors on Kind/Minikube, regenerating the cluster resolves local credential issues quickly. For "doesn't have a resource type," run `kubectl api-resources` to see valid resource names. For empty output, add `-A` or the correct `-n <namespace>` flag.

### Practice Tasks
1. List all namespaces with `kubectl get ns`.
2. Describe the `kube-system` namespace's Pods and read through the `Events` section of one.
3. Run `kubectl api-resources` and identify five resource types you haven't heard of yet.
4. Set up the `k` alias and autocompletion, then use it for the rest of this manual.
5. Practice switching contexts between your Kind and Minikube clusters (if both are installed).

### Challenge Lab
Without looking anything up, from memory, write the full command to: list all Pods in the `kube-system` namespace sorted by restart count, describe one of them, and fetch its logs — then verify your answer against the documentation.

### Best Practices
- Always confirm `kubectl config current-context` before running destructive commands (`delete`, `apply` with force).
- Prefer `apply` over `create` for anything you intend to manage as code long-term.
- Use `-o yaml` or `-o json` on `get` commands to inspect the full underlying object definition, not just the summary table.

### Interview Questions
1. **Q: What is the basic structure of a kubectl command?** A: `kubectl <verb> <resource> <name> [flags]`.
2. **Q: What is the difference between `create` and `apply`?** A: `create` fails if the resource exists; `apply` creates or updates it declaratively via a diff.
3. **Q: Where does kubectl store cluster connection info?** A: In a kubeconfig file, usually at `~/.kube/config`.
4. **Q: What does `kubectl describe` show that `kubectl get` doesn't?** A: Detailed fields and a chronological list of recent Events for that object.
5. **Q: How do you list resources across all namespaces?** A: Add the `-A` (or `--all-namespaces`) flag.
6. **Q: How do you view a Pod's logs?** A: `kubectl logs <pod-name>`.
7. **Q: How do you run a shell inside a running container?** A: `kubectl exec -it <pod-name> -- /bin/sh` (or `/bin/bash`).
8. **Q: What does a Kubernetes "context" represent?** A: A combination of cluster, user, and default namespace that kubectl uses for a given session.
9. **Q: How do you discover all resource types a cluster supports?** A: `kubectl api-resources`.
10. **Q: Why is `apply` generally preferred for production workflows?** A: It supports idempotent, declarative updates suitable for version-controlled, repeatable deployments (GitOps).

### Quiz
1. What verb lists resources? (a) show (b) get (c) list (d) view — **b**
2. What flag lists resources in every namespace? (a) `--all` (b) `-A` (c) `--everywhere` (d) `-N` — **b**
3. Which command fails if the resource already exists? (a) apply (b) create (c) patch (d) edit — **b**
4. Where is the default kubeconfig location? (a) `/etc/kube/config` (b) `~/.kube/config` (c) `/root/config.yaml` (d) `/var/lib/kubectl` — **b**
5. Which command shows recent Events for an object? (a) get (b) describe (c) logs (d) exec — **b**
6. Which command opens a shell in a running container? (a) kubectl shell (b) kubectl exec -it ... -- /bin/sh (c) kubectl ssh (d) kubectl attach --shell — **b**
7. Which command lists all supported resource types? (a) kubectl api-resources (b) kubectl list-types (c) kubectl resources (d) kubectl help types — **a**
8. A "context" in kubectl combines: (a) Only a namespace (b) Cluster, user, and namespace (c) Only credentials (d) Only cluster name — **b**
9. Which is the preferred command for GitOps-style, repeatable deployments? (a) create (b) apply (c) run (d) patch — **b**
10. `kubectl config current-context` shows: (a) All contexts (b) The active context (c) All clusters (d) The kubeconfig file path — **b**

### Homework
Write a personal "kubectl cheat sheet" of the 15 commands you expect to use most often based on this lab, with a one-line description of each. You will expand this cheat sheet as you progress through the manual.

### Summary
`kubectl` is the primary interface to any Kubernetes cluster, following a consistent `verb resource name` pattern. You learned to inspect nodes and Pods, switch contexts safely, and understand the difference between imperative (`create`) and declarative (`apply`) workflows. Lab 6 puts this into practice by creating your first Pod.

---

# Lab 6 — Creating Your First Pod

**Estimated Duration:** 60 minutes
**Difficulty Level:** Beginner

### Learning Objectives
1. Understand what a Pod is and why it is the smallest deployable unit in Kubernetes.
2. Write a complete Pod YAML manifest from scratch.
3. Create, inspect, and delete a Pod using `kubectl`.
4. Understand the purpose of each core field in a Pod spec.

### Prerequisites
- Completion of Lab 5.

### Theory

A **Pod** is the smallest deployable unit in Kubernetes. Importantly, a Pod is not the same thing as a container — a Pod is a wrapper that can hold **one or more** containers that are always scheduled together, on the same node, sharing the same network namespace (so they can reach each other via `localhost`) and, optionally, storage volumes.

Most Pods run a single container, but multi-container Pods (covered in Volume 2) are used for tightly coupled helper patterns, such as a logging sidecar sitting alongside a main application container.

You almost never create Pods directly in production — instead you use higher-level objects like Deployments (Volume 2) that create and manage Pods for you, adding self-healing and scaling. However, understanding raw Pods first is essential, since Deployments are ultimately just a wrapper that manages Pods.

### Architecture Diagram

```
                     Pod
   +--------------------------------------------+
   |  Shared network namespace (same IP)          |
   |  Shared storage volumes (optional)            |
   |                                              |
   |   +----------------+                         |
   |   |  Container:     |                         |
   |   |  nginx           |                         |
   |   +----------------+                         |
   +--------------------------------------------+
              scheduled onto exactly one node
```

### Installation
No new installation — uses your existing Kind/Minikube cluster.

### Hands-on Practice

First, create the manifest file (see YAML section below), then:

```bash
kubectl apply -f my-first-pod.yaml
```
**Purpose:** Creates the Pod described in the YAML file.
**Syntax:** `-f` points to a local file (or URL).
**Expected Output:** `pod/my-first-pod created`
**Common mistakes:** Indentation errors in YAML (Kubernetes YAML is whitespace-sensitive) causing parsing failures.
**Real-world usage:** The standard way any Kubernetes object is created in real workflows.

```bash
kubectl get pods -o wide
```
**Purpose:** Confirms the Pod exists and shows its assigned node and internal IP.
**Expected Output:**
```
NAME            READY   STATUS    RESTARTS   AGE   IP           NODE
my-first-pod    1/1     Running   0          10s   10.244.1.5   my-cluster-worker
```

```bash
kubectl logs my-first-pod
kubectl exec -it my-first-pod -- /bin/sh
```
**Purpose:** `logs` shows container stdout/stderr; `exec` opens an interactive shell inside the running container.
**Common mistakes:** Trying `exec` on a Pod that has multiple containers without specifying `-c <container-name>`, causing ambiguity errors.
**Real-world usage:** The two most common commands for live debugging a running Pod.

### YAML Files

`my-first-pod.yaml`:
```yaml
apiVersion: v1                 # Core API version; Pods live in the "v1" (core) API group
kind: Pod                      # The type of Kubernetes object being defined
metadata:
  name: my-first-pod           # Unique name for this Pod within its namespace
  labels:
    app: demo                  # Arbitrary key-value tag, used later for selectors
spec:
  containers:
    - name: nginx              # Name of the container within the Pod
      image: nginx:1.25        # Container image and tag to pull and run
      ports:
        - containerPort: 80    # Documents which port the container listens on (informational, doesn't open it externally)
      resources:
        requests:
          cpu: "100m"          # Minimum CPU guaranteed (100 millicores = 0.1 CPU core)
          memory: "64Mi"       # Minimum memory guaranteed
        limits:
          cpu: "250m"          # Maximum CPU this container may use
          memory: "128Mi"      # Maximum memory; exceeding this can cause the container to be killed (OOMKilled)
      livenessProbe:
        httpGet:
          path: /              # Kubernetes periodically GETs this path...
          port: 80              # ...on this port to check the container is still alive
        initialDelaySeconds: 5
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /              # Similar check, but determines if the Pod should receive traffic
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
```

### Expected Output
After `kubectl describe pod my-first-pod`, the `Events` section should end with:
```
Normal  Scheduled  10s   default-scheduler  Successfully assigned default/my-first-pod to my-cluster-worker
Normal  Pulled     9s    kubelet            Successfully pulled image "nginx:1.25"
Normal  Created    9s    kubelet            Created container nginx
Normal  Started    8s    kubelet            Started container nginx
```

### Validation Steps
1. `kubectl get pods` shows `STATUS: Running` and `READY: 1/1`.
2. `kubectl exec -it my-first-pod -- curl -s localhost:80` returns the default nginx welcome page HTML.

### Common Errors
- **ImagePullBackOff** — the image name/tag is wrong, or the registry is unreachable.
- **CrashLoopBackOff** — the container starts and then exits repeatedly, often due to an app crash or wrong command.
- **Pending** — no node has enough resources, or a scheduling constraint can't be satisfied.

### Troubleshooting
For `ImagePullBackOff`, run `kubectl describe pod <name>` and check the `Events` section for the exact pull error, then verify the image name/tag spelling. For `CrashLoopBackOff`, check `kubectl logs <name> --previous` to see why the last attempt exited. For `Pending`, run `kubectl describe pod <name>` and look for scheduling-related events, then check node resource availability with `kubectl describe nodes`.

### Practice Tasks
1. Create a Pod running the `httpd` image instead of `nginx`.
2. Delete your Pod and recreate it using `kubectl apply -f` a second time — confirm it doesn't error like `create` would.
3. Deliberately mistype the image tag and observe the resulting `ImagePullBackOff`.
4. Add a `command` field that overrides the container's default entrypoint to run `sleep 3600` instead, and observe how the container behaves differently.
5. Use `kubectl exec` to inspect environment variables inside your running Pod with `env`.

### Challenge Lab
Create a Pod manifest for a simple `redis:7` container with appropriate resource requests/limits and a `tcpSocket` liveness probe on port 6379. Apply it, verify it's Running, and connect to it via `kubectl exec` using the `redis-cli` tool inside the container.

### Best Practices
- Always set resource `requests` and `limits` — unconstrained Pods can starve other workloads on a node.
- Always define liveness and readiness probes for real applications; without them, Kubernetes cannot detect a hung (but still running) process.
- Avoid the `latest` image tag in production manifests; pin explicit versions for reproducibility.

### Interview Questions
1. **Q: What is a Pod?** A: The smallest deployable unit in Kubernetes, wrapping one or more containers that share network and optionally storage.
2. **Q: Can a Pod contain multiple containers?** A: Yes, though most Pods run a single container; multi-container Pods share network namespace and can share volumes.
3. **Q: Do containers in the same Pod share an IP address?** A: Yes, they share the Pod's network namespace and can reach each other via `localhost`.
4. **Q: What's the difference between a liveness and a readiness probe?** A: Liveness determines if the container should be restarted; readiness determines if it should receive traffic.
5. **Q: What happens if a container exceeds its memory limit?** A: It can be OOMKilled (terminated due to out-of-memory) by the kernel/kubelet.
6. **Q: How do you view logs from a Pod?** A: `kubectl logs <pod-name>`.
7. **Q: What causes ImagePullBackOff?** A: The container image cannot be pulled — wrong name/tag, private registry auth issues, or network problems.
8. **Q: What causes CrashLoopBackOff?** A: The container repeatedly starts and then exits/crashes.
9. **Q: Why are Pods rarely created directly in production?** A: They lack self-healing and scaling on their own; higher-level controllers like Deployments manage them instead.
10. **Q: What field specifies which image a container runs?** A: `spec.containers[].image`.

### Quiz
1. What is the smallest deployable unit in Kubernetes? (a) Container (b) Pod (c) Node (d) Namespace — **b**
2. Containers in the same Pod share: (a) Nothing (b) Network namespace (c) A separate IP each (d) Different nodes — **b**
3. Which probe determines if traffic should be routed to a Pod? (a) Liveness (b) Readiness (c) Startup (d) Health — **b**
4. `ImagePullBackOff` typically indicates: (a) Out of memory (b) Wrong image name/tag or registry issue (c) DNS misconfiguration (d) Disk full — **b**
5. Which field sets the maximum CPU a container may use? (a) requests.cpu (b) limits.cpu (c) max.cpu (d) cpu.ceiling — **b**
6. `kubectl apply -f` versus `kubectl create -f`: apply is preferred because: (a) It's faster (b) It's declarative and idempotent (c) It doesn't need YAML (d) It skips validation — **b**
7. What does CrashLoopBackOff mean? (a) The Pod is Pending forever (b) The container keeps crashing and restarting (c) The node is down (d) DNS failed — **b**
8. Which command opens an interactive shell in a container? (a) kubectl logs -it (b) kubectl exec -it ... -- /bin/sh (c) kubectl shell (d) kubectl attach --shell — **b**
9. Resource `requests` represent: (a) Hard maximum (b) Guaranteed minimum used for scheduling (c) A soft suggestion ignored by the scheduler (d) Node capacity — **b**
10. Multi-container Pods are typically used for: (a) Running unrelated apps together (b) Tightly coupled helper/sidecar patterns (c) Load balancing between clusters (d) Storing Secrets — **b**

### Homework
Create three different Pods (using three different public images of your choice), apply all of them, and write a short report describing each Pod's status, IP, and any issues you had to troubleshoot.

### Summary
Pods are the fundamental building block of Kubernetes workloads, wrapping one or more containers with shared networking. You wrote a complete Pod manifest including resource limits and health probes, and practiced the create/inspect/debug cycle you'll repeat throughout this manual. Lab 7 goes deeper into what happens across a Pod's full lifecycle.

---

# Lab 7 — Understanding Pod Lifecycle

**Estimated Duration:** 50 minutes
**Difficulty Level:** Beginner–Intermediate

### Learning Objectives
1. Identify each Pod phase: `Pending`, `Running`, `Succeeded`, `Failed`, `Unknown`.
2. Understand container states: `Waiting`, `Running`, `Terminated`.
3. Explain restart policies and how Kubernetes decides to restart containers.
4. Use `kubectl describe` and Events to trace a Pod through its lifecycle.

### Prerequisites
- Completion of Lab 6.

### Theory

Every Pod moves through a **phase**, visible in `kubectl get pods` as the `STATUS` column:

- **Pending** — the Pod has been accepted by the cluster but one or more containers haven't started yet (commonly: still being scheduled, or pulling images).
- **Running** — the Pod has been bound to a node and at least one container is running.
- **Succeeded** — all containers terminated successfully and will not be restarted (typical for batch Jobs, covered in Volume 2).
- **Failed** — all containers terminated, and at least one exited with a failure.
- **Unknown** — the Pod's state couldn't be determined, usually due to a communication problem with the node.

Independently, each **container** within a Pod has its own state: `Waiting` (not yet running, e.g., pulling image), `Running`, or `Terminated` (with an exit code and reason, such as `Completed` or `Error`).

The **restartPolicy** field on a Pod controls what happens when a container exits: `Always` (default — restart no matter the exit code, used for long-running services), `OnFailure` (restart only on non-zero exit code, used for Jobs), or `Never` (do not restart, useful for one-off diagnostic Pods).

When a container repeatedly crashes, Kubernetes applies **exponential backoff** before each restart attempt (10s, 20s, 40s... up to a cap of 5 minutes), which is what produces the `CrashLoopBackOff` status you may have seen in Lab 6.

### Architecture Diagram

```
   Pod created
       |
       v
   [Pending] --(scheduled + image pulled)--> [Running]
       |                                          |
       |                                (container exits)
       |                                          |
       |                                          v
       |                              exit code 0? --- yes --> [Succeeded] (if restartPolicy != Always)
       |                                          |
       |                                          no
       |                                          v
       |                                     [Failed] or restarted per restartPolicy
       v
   [Unknown]  (node unreachable)
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl run crash-demo --image=busybox --restart=Never -- sh -c "echo hello; exit 1"
kubectl get pod crash-demo -w
```
**Purpose:** Creates a Pod that intentionally exits with a failure code, then watches (`-w`) its status change live.
**Syntax:** `--restart=Never` tells `kubectl run` to create a bare Pod (rather than a Deployment); the `--` separates kubectl flags from the container's command.
**Expected Output:** The Pod transitions from `Pending` to `Running` briefly, then to `Error` or `Failed`, with `RESTARTS: 0` (since `restartPolicy=Never`).
**Common mistakes:** Forgetting `-w` exits automatically once you Ctrl+C; some learners think it hung.
**Real-world usage:** Simulating and observing failure states is a core debugging skill, especially for diagnosing Jobs.

```bash
kubectl get pod crash-demo -o jsonpath='{.status.containerStatuses[0].state}'
```
**Purpose:** Extracts just the detailed container state as JSON, bypassing the summarized table view.
**Syntax:** `jsonpath` expressions navigate the object's JSON structure directly.
**Expected Output:** `{"terminated":{"containerID":"...","exitCode":1,"reason":"Error",...}}`
**Common mistakes:** Getting the JSONPath syntax wrong (missing braces or dots) and receiving an empty result.
**Real-world usage:** Used heavily in scripts and automation that need to programmatically check Pod state.

### YAML Files
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restart-demo
spec:
  restartPolicy: OnFailure   # Only restart the container if it exits with non-zero status
  containers:
    - name: demo
      image: busybox
      command: ["sh", "-c", "echo starting; sleep 5; exit 1"]  # Deliberately fails after 5s
```

### Expected Output
```
NAME            READY   STATUS      RESTARTS      AGE
restart-demo    0/1     Error       0             5s
restart-demo    0/1     CrashLoopBackOff   1        20s
```

### Validation Steps
Run `kubectl describe pod restart-demo` and confirm the `Events` section shows a sequence of `Pulled`, `Created`, `Started`, and `BackOff` events, demonstrating the restart cycle described in the theory section.

### Common Errors
- **Pod stuck in `Pending` indefinitely** — insufficient cluster resources or an unsatisfiable scheduling constraint.
- **`CreateContainerConfigError`** — usually a missing ConfigMap or Secret referenced by the Pod (covered fully in Volume 4).
- **`Terminating` for a long time** — a container isn't responding to `SIGTERM`; Kubernetes will force-kill after the grace period (default 30s).

### Troubleshooting
For stuck `Pending` Pods, run `kubectl describe pod <name>` and read the last Event for a scheduling failure reason (e.g., `Insufficient cpu`). For `CreateContainerConfigError`, check that every referenced ConfigMap/Secret actually exists in the same namespace. For long `Terminating` Pods, check if the app handles `SIGTERM` gracefully, or force delete as a last resort with `kubectl delete pod <name> --grace-period=0 --force` (understanding this can leave orphaned resources).

### Practice Tasks
1. Create a Pod with `restartPolicy: Never` that exits successfully (`exit 0`) and confirm it reaches `Succeeded`, not `Running`.
2. Create a Pod that requests more CPU than any node has available, and observe it stay `Pending`.
3. Use `kubectl get pod <name> -o jsonpath='{.status.phase}'` to script a status check.
4. Trigger and observe a full `CrashLoopBackOff` cycle, timing the backoff intervals.
5. Force-delete a stuck Pod and explain, in your own words, the risk of doing so.

### Challenge Lab
Write a small bash script that creates a Pod, polls its phase every 2 seconds using `jsonpath`, and prints a message the moment the Pod reaches `Running` or `Failed`.

### Best Practices
- Never routinely force-delete Pods in production without understanding why they're stuck — it can leave orphaned processes or volume attachments.
- Design applications to handle `SIGTERM` gracefully so they shut down cleanly within the default 30-second grace period.
- Use `restartPolicy: OnFailure` or `Never` for batch-style, run-to-completion workloads, and leave `Always` (the default) for long-running services.

### Interview Questions
1. **Q: What are the five Pod phases?** A: Pending, Running, Succeeded, Failed, Unknown.
2. **Q: What's the difference between Pod phase and container state?** A: Pod phase is a high-level summary; container state (Waiting/Running/Terminated) is per-container detail.
3. **Q: What does restartPolicy: Always do?** A: Restarts the container regardless of exit code — the default, used for long-running services.
4. **Q: What causes CrashLoopBackOff?** A: A container repeatedly exits/crashes shortly after starting, triggering exponential backoff between restart attempts.
5. **Q: What is the default termination grace period?** A: 30 seconds.
6. **Q: How do you extract a specific field from Pod status via CLI?** A: Using `kubectl get pod <name> -o jsonpath='{...}'`.
7. **Q: When would you use restartPolicy: Never?** A: For one-off diagnostic or run-once Pods where you don't want automatic restarts.
8. **Q: Why might a Pod stay in Pending forever?** A: Insufficient cluster resources or unsatisfiable scheduling constraints (e.g., node selectors, taints).
9. **Q: What signal does Kubernetes send to gracefully stop a container?** A: SIGTERM, followed by SIGKILL if the grace period expires.
10. **Q: What does force-deleting a Pod risk?** A: Skipping graceful shutdown, potentially orphaning resources or leaving in-flight work incomplete.

### Quiz
1. Which phase means all containers exited successfully and won't restart? (a) Running (b) Succeeded (c) Failed (d) Unknown — **b**
2. Default restartPolicy is: (a) Never (b) OnFailure (c) Always (d) None — **c**
3. CrashLoopBackOff results from: (a) Node failure (b) Repeated container crashes with exponential backoff (c) DNS failure (d) Missing image only — **b**
4. Default termination grace period is: (a) 5s (b) 30s (c) 60s (d) 120s — **b**
5. Which restartPolicy is typical for batch Jobs? (a) Always (b) OnFailure (c) Never only (d) Automatic — **b**
6. `Unknown` phase usually indicates: (a) A successful run (b) Communication failure with the node (c) A scheduling error (d) A YAML syntax error — **b**
7. Container state `Waiting` most often means: (a) It crashed (b) It hasn't started yet, e.g., pulling image (c) It finished successfully (d) It's been deleted — **b**
8. `jsonpath` output format is useful for: (a) Pretty printing only (b) Extracting specific fields programmatically (c) Deleting resources (d) Creating YAML — **b**
9. Force-deleting a Pod uses: (a) `--force` only (b) `--grace-period=0 --force` (c) `--now` (d) `--hard-delete` — **b**
10. Which signal is sent first during graceful shutdown? (a) SIGKILL (b) SIGTERM (c) SIGSTOP (d) SIGHUP — **b**

### Homework
Create three Pods, each with a different `restartPolicy` (`Always`, `OnFailure`, `Never`) all running a command that exits with code 1 after 5 seconds. Observe and document how each one behaves differently over the next two minutes.

### Summary
Understanding Pod phases, container states, and restart policies is essential for diagnosing real-world issues. You practiced simulating failures and reading Events to trace the exact lifecycle of a Pod. Lab 8 introduces labels, selectors, and annotations — the metadata system used to organize and select groups of Pods.

---

# Lab 8 — Labels, Selectors and Annotations

**Estimated Duration:** 45 minutes
**Difficulty Level:** Beginner

### Learning Objectives
1. Apply labels to Kubernetes objects and explain their purpose.
2. Use label selectors to filter and target groups of resources.
3. Distinguish labels from annotations and choose the right one for a given use case.
4. Use `kubectl label` and `kubectl get --selector` in practice.

### Prerequisites
- Completion of Lab 7.

### Theory

**Labels** are key-value pairs attached to Kubernetes objects, intended to be used for identifying and grouping resources — for example, `app: frontend`, `env: production`, `tier: backend`. Crucially, labels are not just descriptive metadata; they are the mechanism many core Kubernetes features rely on. Services find the Pods they should route traffic to using label selectors; Deployments know which Pods they own using label selectors; and you as an operator use label selectors to run bulk `kubectl` operations across a group of related resources.

**Selectors** come in two forms: **equality-based** (`app=frontend`, `env!=production`) and **set-based** (`environment in (production, staging)`, `tier notin (frontend)`). Equality-based selectors are more common for simple cases; set-based selectors are useful for more complex matching logic.

**Annotations**, unlike labels, are also key-value pairs, but they are **not** used for selection. Annotations exist to attach arbitrary, often larger, non-identifying metadata to objects — for example, a build timestamp, a Git commit hash, a contact email, or configuration for a specific controller (many Ingress controllers read behavior configuration from annotations). A simple rule of thumb: **if you need to select or group by it, use a label; if you just need to attach information to it, use an annotation.**

### Architecture Diagram

```
   Pod A                Pod B               Pod C
   labels:               labels:             labels:
     app=web               app=web             app=db
     env=prod               env=staging         env=prod
      \                      |                    /
       \                     |                   /
        \                    |                  /
     Selector: app=web   -->  matches Pod A, Pod B (not C)
     Selector: env=prod  -->  matches Pod A, Pod C (not B)
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl label pod my-first-pod env=production tier=frontend
```
**Purpose:** Adds two labels to an already-existing Pod.
**Syntax:** `kubectl label <resource> <name> key=value [key2=value2 ...]`.
**Expected Output:** `pod/my-first-pod labeled`
**Common mistakes:** Trying to overwrite an existing label key without `--overwrite`, which returns an error by design (to prevent accidental changes).
**Real-world usage:** Common when retroactively tagging resources for cost tracking, ownership, or environment classification.

```bash
kubectl get pods --selector app=demo
kubectl get pods -l 'env in (production, staging)'
```
**Purpose:** Filters Pods using equality-based and set-based selectors respectively.
**Syntax:** `--selector` and its shorthand `-l` both accept selector expressions.
**Expected Output:** Only Pods matching the given labels are listed.
**Common mistakes:** Quoting issues in the shell when using set-based selectors with parentheses and commas.
**Real-world usage:** Bulk operations, e.g., `kubectl delete pods -l env=staging` to clean up an entire environment at once.

```bash
kubectl annotate pod my-first-pod owner="platform-team@example.com" build-commit="a1b2c3d"
```
**Purpose:** Attaches non-identifying metadata to the Pod.
**Expected Output:** `pod/my-first-pod annotated`
**Common mistakes:** Using annotations where a label was actually needed (e.g., trying to `--selector` on an annotation, which is not possible).
**Real-world usage:** Recording deployment metadata such as CI build numbers or contact information for the owning team.

### YAML Files
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: labeled-pod
  labels:
    app: demo             # Used for selection/grouping
    tier: frontend
    env: staging
  annotations:
    owner: "platform-team@example.com"   # Informational only — not selectable
    build-commit: "a1b2c3d"
spec:
  containers:
    - name: nginx
      image: nginx:1.25
```

### Expected Output
```bash
$ kubectl get pods -l app=demo --show-labels
NAME          READY   STATUS    RESTARTS   AGE   LABELS
labeled-pod   1/1     Running   0          5s    app=demo,env=staging,tier=frontend
```

### Validation Steps
Run `kubectl get pods --show-labels` and confirm your applied labels appear exactly as expected; run `kubectl get pod labeled-pod -o jsonpath='{.metadata.annotations}'` to confirm annotations are present but do not show up under `--show-labels`.

### Common Errors
- **"overwrite is false" error** — attempting to change an existing label's value without `--overwrite`.
- **Empty selector results** — typo in label key/value, or case sensitivity mismatch (labels are case-sensitive).
- **Confusing labels and annotations** — writing a Service selector that references an annotation key, which will never match anything.

### Troubleshooting
Add `--overwrite` to `kubectl label` when intentionally replacing a value. For empty selector results, run `kubectl get pods --show-labels` to see the exact labels present and compare character-by-character. Remember: Services and Deployments only ever match on **labels**, never annotations.

### Practice Tasks
1. Label three different Pods with the same `app=` value and select them all in a single `kubectl get pods -l` command.
2. Try changing an existing label's value without `--overwrite` and observe the error, then retry with `--overwrite`.
3. Add at least three annotations to a Pod describing its owner, purpose, and last-updated date.
4. Use a set-based selector to select Pods where `tier notin (backend)`.
5. Remove a label using `kubectl label pod <name> <key>-` (note the trailing dash).

### Challenge Lab
Create five Pods representing a fictional application with labels for `app`, `tier` (frontend/backend), and `env` (dev/staging/prod). Then write a single `kubectl get pods` command using a set-based selector that returns only backend Pods in staging or production.

### Best Practices
- Adopt a consistent labeling convention across your organization early (e.g., `app.kubernetes.io/name`, `app.kubernetes.io/instance`, following the official recommended labels).
- Never store sensitive information in labels or annotations — they are visible to anyone with read access to the object and are not encrypted.
- Use annotations for tooling metadata (e.g., Ingress controller configuration) exactly as that tool's documentation specifies.

### Interview Questions
1. **Q: What is the primary purpose of labels?** A: To identify and group Kubernetes objects for selection by controllers, Services, and operators.
2. **Q: How do Services find the Pods they route to?** A: Via a label selector defined in the Service spec matching Pod labels.
3. **Q: What's the difference between labels and annotations?** A: Labels are used for selection/identification; annotations hold non-identifying metadata and cannot be selected on.
4. **Q: What are the two types of label selectors?** A: Equality-based (`key=value`) and set-based (`key in (v1,v2)`).
5. **Q: How do you remove a label from a resource?** A: `kubectl label <resource> <name> <key>-` (trailing dash).
6. **Q: Why does kubectl require `--overwrite` to change a label?** A: To prevent accidental, unintended changes to identifying metadata that other resources may depend on.
7. **Q: Are labels case-sensitive?** A: Yes.
8. **Q: Can annotations be used in a Service selector?** A: No — Service selectors only match labels.
9. **Q: What is a common real-world use for set-based selectors?** A: Matching multiple environments or excluding a specific tier in a single query, e.g., `env in (staging, prod)`.
10. **Q: Should sensitive data ever be stored in labels or annotations?** A: No, they are plaintext and visible to anyone with read access; use Secrets instead.

### Quiz
1. Labels are primarily used for: (a) Encryption (b) Selection and grouping (c) Logging (d) Networking — **b**
2. Annotations are: (a) Selectable metadata (b) Non-selectable, informational metadata (c) Only for Secrets (d) Required on every object — **b**
3. Which selector type uses `in (a, b)` syntax? (a) Equality-based (b) Set-based (c) Regex-based (d) Range-based — **b**
4. To overwrite an existing label value, you must add: (a) `--force` (b) `--overwrite` (c) `--replace` (d) Nothing extra — **b**
5. Removing a label uses syntax: (a) `key=` (b) `key-` (c) `key!` (d) `--remove key` — **b**
6. Are labels case-sensitive? (a) Yes (b) No — **a**
7. Services select Pods using: (a) Annotations (b) Labels (c) Pod names only (d) Namespaces only — **b**
8. Sensitive data should be stored in: (a) Labels (b) Annotations (c) Secrets (d) Pod names — **c**
9. `kubectl get pods -l app=web` filters using: (a) Field selector (b) Label selector (c) Annotation selector (d) Namespace selector — **b**
10. A recommended practice is to: (a) Avoid labeling anything (b) Adopt a consistent labeling convention org-wide (c) Only label Services (d) Use labels for secrets — **b**

### Homework
Design a labeling scheme (on paper) for a fictional e-commerce platform with a frontend, backend API, database, and cache, across dev/staging/prod environments. List the label keys and possible values you'd use, and explain your reasoning.

### Summary
Labels are the selection mechanism underlying Services, Deployments, and most `kubectl` bulk operations, while annotations hold non-selectable metadata. Establishing a consistent labeling scheme early pays off enormously as a cluster grows. Lab 9 covers Namespaces, which provide another layer of organization at the cluster level.

---

# Lab 9 — Namespaces

**Estimated Duration:** 40 minutes
**Difficulty Level:** Beginner

### Learning Objectives
1. Explain the purpose of Namespaces and when to use them.
2. Create and delete Namespaces.
3. Understand which resources are namespaced versus cluster-scoped.
4. Set a default namespace for a kubectl context.

### Prerequisites
- Completion of Lab 8.

### Theory

A **Namespace** provides a way to divide a single Kubernetes cluster into multiple virtual clusters. Namespaces are commonly used to separate environments (e.g., `dev`, `staging`, `prod`), separate teams sharing one cluster, or separate applications, allowing each to have its own set of similarly-named resources without collision — you could have a Pod named `web` in both the `team-a` and `team-b` namespaces without conflict.

Every cluster starts with four built-in namespaces: `default` (used when no namespace is specified), `kube-system` (control plane and system components), `kube-public` (readable by all users, rarely used), and `kube-node-lease` (used internally for node heartbeat tracking).

Not every Kubernetes object lives inside a namespace. **Namespaced resources** include Pods, Deployments, Services, ConfigMaps, and Secrets. **Cluster-scoped resources** include Nodes, PersistentVolumes, Namespaces themselves, and ClusterRoles. You can check any resource type's scope with `kubectl api-resources --namespaced=true` (or `false`).

Namespaces alone do not provide network isolation or hard resource limits between teams — those require additional configuration (NetworkPolicies in Volume 3, and ResourceQuotas/LimitRanges) layered on top of namespace boundaries.

### Architecture Diagram

```
                    Kubernetes Cluster
   +----------------------------------------------------+
   |  Namespace: dev        Namespace: staging            |
   |   Pod: web               Pod: web                    |
   |   Service: web-svc        Service: web-svc            |
   |                                                       |
   |  Namespace: prod        Namespace: kube-system         |
   |   Pod: web                CoreDNS, etcd, kube-proxy...  |
   |   Service: web-svc                                     |
   +----------------------------------------------------+
   Cluster-scoped: Nodes, PersistentVolumes, Namespaces themselves
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl create namespace staging
kubectl get namespaces
```
**Purpose:** Creates a new namespace and lists all namespaces in the cluster.
**Syntax:** `create namespace <name>` — imperative form; a YAML equivalent is shown below.
**Expected Output:** `namespace/staging created`, followed by a table including `default`, `kube-system`, `kube-public`, `kube-node-lease`, and `staging`.
**Common mistakes:** Assuming namespaces provide strong security isolation by default — they don't, without RBAC and NetworkPolicies layered on top.
**Real-world usage:** Standard step when onboarding a new team or environment onto a shared cluster.

```bash
kubectl config set-context --current --namespace=staging
kubectl config view --minify | grep namespace
```
**Purpose:** Changes the default namespace for your current context so you don't need `-n staging` on every command.
**Syntax:** `--current` modifies the active context in place.
**Expected Output:** Confirms `namespace: staging` is now set for the current context.
**Common mistakes:** Forgetting you changed the default namespace and being confused when `kubectl get pods` returns nothing (because you're now looking in `staging`, not `default`).
**Real-world usage:** Common productivity habit so engineers don't accidentally deploy to the wrong namespace by forgetting a flag.

```bash
kubectl api-resources --namespaced=false
```
**Purpose:** Lists all cluster-scoped (non-namespaced) resource types.
**Expected Output:** A table including `Node`, `Namespace`, `PersistentVolume`, `ClusterRole`, `ClusterRoleBinding`.
**Real-world usage:** Useful reference when writing RBAC rules or scripting cluster-wide resource management.

### YAML Files
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging          # No 'spec' fields are required for a basic Namespace
  labels:
    team: platform         # Namespaces can also carry labels, useful for policy targeting
```

To create a resource inside a specific namespace via YAML:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: staging      # Explicitly places this Pod inside the "staging" namespace
spec:
  containers:
    - name: nginx
      image: nginx:1.25
```

### Expected Output
```
NAME              STATUS   AGE
default           Active   2d
kube-node-lease   Active   2d
kube-public       Active   2d
kube-system       Active   2d
staging           Active   5s
```

### Validation Steps
Run `kubectl get pods -n staging` (or without `-n` if you set it as your default context namespace) and confirm your Pod appears only in `staging`, not `default`.

### Common Errors
- **"namespaces 'staging' not found"** — the namespace wasn't actually created, or was deleted; check with `kubectl get ns`.
- **Resources "disappearing"** — actually just created in the wrong namespace due to a stale default context.
- **Deleting a namespace hangs in `Terminating`** — usually a finalizer on a resource inside it is blocking deletion.

### Troubleshooting
For "not found" errors, re-run `kubectl get ns` to confirm exact spelling. For resources that seem to have vanished, check `kubectl config view --minify | grep namespace` to see your current default. For a namespace stuck in `Terminating`, inspect `kubectl get namespace <name> -o json` for a non-empty `spec.finalizers` list, which is an advanced topic usually requiring careful manual intervention.

### Practice Tasks
1. Create three namespaces: `dev`, `staging`, `prod`.
2. Create identically-named Pods in each of the three namespaces and confirm there's no naming conflict.
3. Set your default context namespace to `dev`, then confirm `kubectl get pods` no longer shows `default` namespace resources.
4. Run `kubectl api-resources --namespaced=true` and count how many resource types are namespaced.
5. Delete the `staging` namespace and observe that all Pods inside it are also deleted (namespace deletion cascades).

### Challenge Lab
Create a `team-a` and `team-b` namespace, each with a Pod named `app` and a ConfigMap named `settings`, and confirm through `kubectl get all -n team-a` and `kubectl get all -n team-b` that the two are fully isolated by name.

### Best Practices
- Never deploy real workloads directly into `kube-system` — that namespace is reserved for cluster-critical system components.
- Combine namespaces with ResourceQuotas (Volume 4) to prevent one team from consuming all cluster resources.
- Set your kubectl context's default namespace explicitly for each project to avoid accidental cross-environment mistakes.

### Interview Questions
1. **Q: What is a Namespace used for?** A: Dividing a cluster into virtual sub-clusters, typically by team, environment, or application.
2. **Q: What are the four default namespaces?** A: default, kube-system, kube-public, kube-node-lease.
3. **Q: Do Namespaces provide network isolation by default?** A: No — that requires NetworkPolicies in addition to namespace boundaries.
4. **Q: Are Nodes namespaced?** A: No, Nodes are cluster-scoped resources.
5. **Q: How do you check if a resource type is namespaced?** A: `kubectl api-resources --namespaced=true/false`.
6. **Q: What happens to Pods when their namespace is deleted?** A: They are also deleted — namespace deletion cascades to everything inside it.
7. **Q: How do you set a default namespace for your current kubectl context?** A: `kubectl config set-context --current --namespace=<name>`.
8. **Q: Can two Pods with the same name exist in different namespaces?** A: Yes, since names only need to be unique within a namespace.
9. **Q: Should you deploy application workloads into kube-system?** A: No, that namespace is reserved for cluster system components.
10. **Q: What besides Namespaces is typically used to enforce resource fairness between teams?** A: ResourceQuotas and LimitRanges.

### Quiz
1. Which namespace is used when none is specified? (a) kube-system (b) default (c) kube-public (d) none — **b**
2. Which namespace holds control plane components? (a) default (b) kube-system (c) kube-public (d) kube-node-lease — **b**
3. Are Nodes namespaced? (a) Yes (b) No — **b**
4. Deleting a namespace: (a) Leaves its resources intact (b) Cascades and deletes everything inside it (c) Requires deleting resources first manually always (d) Is reversible — **b**
5. To check if a resource type is cluster-scoped: (a) kubectl get scope (b) kubectl api-resources --namespaced=false (c) kubectl describe scope (d) kubectl cluster-info — **b**
6. Namespaces alone provide: (a) Full network isolation (b) Naming/organizational separation only (c) Encryption (d) Backup — **b**
7. Which command sets a default namespace for the current context? (a) kubectl namespace set (b) kubectl config set-context --current --namespace=<name> (c) kubectl use namespace (d) kubectl set default-ns — **b**
8. Can identical Pod names exist across different namespaces? (a) No (b) Yes — **b**
9. kube-node-lease is used for: (a) Secrets storage (b) Node heartbeat tracking (c) DNS (d) Ingress — **b**
10. Best practice for enforcing resource fairness across teams sharing a cluster: (a) Namespaces alone (b) ResourceQuotas and LimitRanges alongside namespaces (c) Nothing needed (d) Deleting unused Pods manually — **b**

### Homework
Set up a `dev`, `staging`, and `prod` namespace structure on your local cluster, deploy a simple Pod into each, and write a short explanation of what additional controls (referencing concepts from later volumes if needed) you'd add before trusting this structure in a real multi-team production environment.

### Summary
Namespaces let you logically divide a cluster for organizational purposes, but they are not a security boundary by themselves. You practiced creating namespaces, placing resources inside them, and understanding the namespaced vs. cluster-scoped resource distinction. Lab 10 brings together everything from this volume into a single mini project.

---

# Lab 10 — Mini Project

**Estimated Duration:** 90 minutes
**Difficulty Level:** Intermediate (capstone for Volume 1)

### Learning Objectives
1. Combine everything learned in Labs 1–9 into a single working exercise.
2. Practice organizing resources by namespace and label.
3. Practice full-cycle debugging using `describe`, `logs`, and Events.
4. Build confidence before advancing to Volume 2 (Workloads).

### Prerequisites
- Completion of Labs 1–9.

### Theory

This mini project simulates a small but realistic scenario: standing up an isolated environment for a fictional two-tier "hello-app" consisting of a web-facing Pod and a backend Pod, correctly namespaced, labeled, and annotated, with health checks in place. Although you have not yet learned Services or Deployments (Volume 2 and 3), this project intentionally uses raw Pods so you fully internalize what those higher-level objects will later automate for you.

### Architecture Diagram

```
   Namespace: hello-app
   +-------------------------------------------+
   |  Pod: web        labels: app=hello,tier=web |
   |   image: nginx:1.25                          |
   |                                              |
   |  Pod: api        labels: app=hello,tier=api  |
   |   image: hashicorp/http-echo                 |
   +-------------------------------------------+
```

### Installation
No new installation — uses your existing Kind or Minikube cluster.

### Hands-on Practice / Project Steps

**Step 1 — Create the namespace:**
```bash
kubectl create namespace hello-app
```

**Step 2 — Apply the manifests below** (save as `hello-app.yaml`):
```bash
kubectl apply -f hello-app.yaml
```

**Step 3 — Verify both Pods are Running:**
```bash
kubectl get pods -n hello-app -o wide
```

**Step 4 — Check logs and connectivity:**
```bash
kubectl logs -n hello-app web
kubectl exec -n hello-app -it web -- curl -s localhost:80 | head -5
```

**Step 5 — Practice a selector-based bulk operation:**
```bash
kubectl get pods -n hello-app -l app=hello
```

### YAML Files

`hello-app.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: hello-app
  labels:
    app: hello
    tier: web
  annotations:
    owner: "you@example.com"
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
---
apiVersion: v1
kind: Pod
metadata:
  name: api
  namespace: hello-app
  labels:
    app: hello
    tier: api
  annotations:
    owner: "you@example.com"
spec:
  containers:
    - name: http-echo
      image: hashicorp/http-echo
      args: ["-text=hello from the api tier"]
      ports:
        - containerPort: 5678
      resources:
        requests: { cpu: "50m", memory: "32Mi" }
        limits: { cpu: "100m", memory: "64Mi" }
```

### Expected Output
```
NAME   READY   STATUS    RESTARTS   AGE   IP            NODE
api    1/1     Running   0          20s   10.244.1.7    my-cluster-worker
web    1/1     Running   0          20s   10.244.1.6    my-cluster-worker
```

### Validation Steps
1. Both Pods show `STATUS: Running` and `READY: 1/1`.
2. `kubectl get pods -n hello-app -l tier=web` returns only `web`.
3. `kubectl exec -n hello-app -it web -- curl -s localhost:80` returns nginx's default HTML page.

### Common Errors
- Forgetting `-n hello-app` on every command and wondering why Pods "don't exist" (they're in the wrong namespace context).
- Label typos causing selector queries to return nothing.
- Requesting more resources in `requests` than your local cluster has available, leaving a Pod `Pending`.

### Troubleshooting
Set your namespace as the context default (`kubectl config set-context --current --namespace=hello-app`) to avoid repeatedly typing `-n`. Double-check label spelling with `--show-labels`. If a Pod is `Pending` due to resources, lower the `requests` values or increase your Kind/Minikube node resources.

### Practice Tasks
1. Add a third Pod (`tier: cache`, any lightweight image such as `redis:7`) to the same namespace and same `app=hello` label group.
2. Delete only the `api` Pod using a label selector that targets just `tier=api`.
3. Annotate all three Pods with a `project=mini-project-1` annotation.
4. Practice recovering from a deliberately broken manifest (e.g., wrong image name) using `describe` and `logs`.
5. Tear down the entire namespace with one command and confirm all Pods are gone.

### Challenge Lab
Extend this mini project into three separate namespaces (`hello-app-dev`, `hello-app-staging`, `hello-app-prod`), each with the same two Pods but different `env` labels, and write the exact `kubectl` commands you'd use to list every "web" tier Pod across all three environments in one command each.

### Best Practices
- Even in a learning project, always set resource `requests`/`limits` and at least a readiness probe — build the habit now.
- Keep manifests in a single file per logical application (as done here) to make review and version control easier.
- Clean up (`kubectl delete namespace hello-app`) after finishing to free local resources.

### Interview Questions
1. **Q: Why use a dedicated namespace for this mini project instead of `default`?** A: To isolate and organize related resources, avoid naming collisions, and practice realistic namespace usage.
2. **Q: Why were readiness probes added even for a simple demo?** A: To build the habit of always defining health checks, since production apps depend on them for correct traffic routing.
3. **Q: What would happen if the `web` and `api` Pods were deployed without labels?** A: You would lose the ability to select or bulk-manage them together, and later, Services/Deployments in Volume 2/3 wouldn't be able to target them either.
4. **Q: How would you quickly delete just the `api`-tier Pod?** A: `kubectl delete pod -n hello-app -l tier=api`.
5. **Q: What's the fastest way to confirm a Pod is actually serving traffic internally?** A: `kubectl exec` into it (or another Pod) and `curl` its port directly.
6. **Q: Why annotate the Pods with an owner email?** A: To record non-selectable operational metadata useful for accountability without affecting selection logic.
7. **Q: What single command deletes the entire mini project cleanly?** A: `kubectl delete namespace hello-app`.
8. **Q: What's a risk of not setting resource requests/limits, even in a demo?** A: Pods can consume unbounded resources and starve other workloads, or fail to schedule predictably.
9. **Q: How does this exercise foreshadow Volume 2's Deployments?** A: Deployments will automate exactly what you did manually here — creating, labeling, and health-checking Pods — while adding self-healing and scaling.
10. **Q: Why is it useful to intentionally break something during practice?** A: It builds real debugging skill using `describe`/`logs`/Events rather than only ever seeing the "happy path."

### Quiz
1. This mini project uses how many Pods? (a) 1 (b) 2 (c) 3 (d) 4 — **b**
2. Which field ties both Pods together for group selection? (a) metadata.name (b) labels.app=hello (c) namespace only (d) annotations — **b**
3. Which command deletes only the api-tier Pod? (a) kubectl delete pod api (b) kubectl delete pod -n hello-app -l tier=api (c) kubectl delete namespace (d) kubectl delete all — **b**
4. What does the readiness probe on `web` check? (a) CPU usage (b) HTTP response on port 80 (c) Disk space (d) Node health — **b**
5. What command removes the entire mini project cleanly? (a) kubectl delete pod all (b) kubectl delete namespace hello-app (c) kubectl stop hello-app (d) kubectl clean — **b**
6. Why isolate this exercise in its own namespace? (a) Required by Kubernetes (b) Organizational best practice, avoids collisions (c) Improves performance (d) Required for readiness probes — **b**
7. What is the purpose of the `owner` annotation here? (a) Selection (b) Non-selectable operational metadata (c) Security enforcement (d) Networking — **b**
8. If `web`'s image name were misspelled, what status would you expect? (a) Running (b) ImagePullBackOff (c) Succeeded (d) Terminating — **b**
9. Which Volume 2 concept will later automate what Lab 10 did manually? (a) Namespaces (b) Deployments (c) Secrets (d) Ingress — **b**
10. What is the primary goal of this mini project? (a) Learn Ingress (b) Consolidate Volume 1 concepts into one hands-on exercise (c) Learn RBAC (d) Learn etcd backup — **b**

### Homework
Extend the mini project with a third `cache` tier Pod, apply appropriate labels/annotations/resource limits/probes matching the patterns in this lab, tear the whole namespace down, then recreate it entirely from your own memory (not copy-pasting) to reinforce muscle memory before moving to Volume 2.

### Summary
This mini project combined Pods, labels, annotations, namespaces, resource limits, probes, and debugging practices from every prior lab in Volume 1 into one realistic exercise. You are now ready for Volume 2, which introduces Deployments and other workload controllers that automate much of what you just did by hand.

---

# Volume 1 — End of Volume Materials

## Volume Review

Volume 1 covered the foundational knowledge required for everything that follows:

- **Lab 1–2:** What Kubernetes is and how its control plane and worker node components fit together.
- **Lab 3–4:** Standing up local clusters using Kind and Minikube.
- **Lab 5:** Mastering `kubectl` as your primary interface to any cluster.
- **Lab 6–7:** Creating Pods and understanding their full lifecycle, states, and restart behavior.
- **Lab 8–9:** Organizing resources with labels, selectors, annotations, and namespaces.
- **Lab 10:** Combining everything into a realistic mini project.

You should now be comfortable creating a local cluster from scratch, writing basic Pod manifests, debugging a failing Pod using `describe`/`logs`/Events, and organizing resources logically. Volume 2 builds directly on this foundation by introducing controllers — Deployments, ReplicaSets, DaemonSets, StatefulSets, Jobs, and CronJobs — that manage Pods on your behalf.

## Practice Exam (25 Questions)

1. What does Kubernetes primarily automate? — Container orchestration (scheduling, scaling, healing, networking).
2. Name the five control plane components. — kube-apiserver, etcd, kube-scheduler, kube-controller-manager, cloud-controller-manager.
3. Name the three worker node components. — kubelet, kube-proxy, container runtime.
4. What does etcd store? — The entire cluster state.
5. What tool runs Kubernetes nodes as Docker containers? — Kind.
6. What tool can run Kubernetes in a VM, container, or other driver, with a built-in dashboard? — Minikube.
7. What is the basic kubectl command structure? — `kubectl <verb> <resource> <name> [flags]`.
8. What's the difference between `create` and `apply`? — `create` fails if it exists; `apply` creates or updates declaratively.
9. What is the smallest deployable unit in Kubernetes? — Pod.
10. Can a Pod contain multiple containers? — Yes.
11. What do containers in the same Pod share? — Network namespace (and optionally volumes).
12. Name the five Pod phases. — Pending, Running, Succeeded, Failed, Unknown.
13. What's the default restartPolicy? — Always.
14. What causes CrashLoopBackOff? — Repeated container crashes with exponential backoff.
15. What's the purpose of a liveness probe? — Determines if a container should be restarted.
16. What's the purpose of a readiness probe? — Determines if a Pod should receive traffic.
17. What are labels used for? — Identification and selection/grouping of resources.
18. What are annotations used for? — Non-selectable, informational metadata.
19. What are the two types of label selectors? — Equality-based and set-based.
20. What is a Namespace used for? — Dividing a cluster into virtual sub-clusters for organization.
21. Name the four default namespaces. — default, kube-system, kube-public, kube-node-lease.
22. Are Nodes namespaced? — No, they are cluster-scoped.
23. What command lists cluster-scoped resource types? — `kubectl api-resources --namespaced=false`.
24. What happens to Pods when their namespace is deleted? — They are cascaded/deleted along with it.
25. What is the recommended way to manage Kubernetes resources long-term (declarative, Git-friendly)? — `kubectl apply -f` with version-controlled YAML manifests.

## Practical Assessment

Complete the following without referring back to the labs:
1. Create a new Kind cluster named `assessment` with one control-plane and two worker nodes.
2. Create a namespace called `assess-ns`.
3. Write and apply a Pod manifest in that namespace running `nginx:1.25`, with resource requests/limits and a readiness probe.
4. Label the Pod `app=assess` and `tier=web`.
5. Annotate the Pod with your name as `owner`.
6. Retrieve the Pod using a label selector.
7. View its logs and exec into it to confirm it's serving traffic on port 80.
8. Delete the Pod using its label selector, then delete the namespace.
9. Delete the Kind cluster entirely.

**Self-grading:** You should be able to complete every step using only commands and YAML you can recall from memory, in under 20 minutes.

## Mini Project Reference
See Lab 10 for the full mini project (`hello-app` namespace with `web` and `api` tier Pods). Use it as your reference implementation when tackling the Practical Assessment above.

## Troubleshooting Exercises

1. **Scenario:** A Pod shows `ImagePullBackOff`. **Diagnosis steps:** `kubectl describe pod <name>`, check the exact image string in `Events`, verify spelling and registry access.
2. **Scenario:** A Pod shows `Pending` for over 5 minutes. **Diagnosis steps:** `kubectl describe pod <name>`, check `Events` for scheduling failures; `kubectl describe nodes` for resource pressure.
3. **Scenario:** A Pod is `Running` but not responding to requests. **Diagnosis steps:** Check readiness probe configuration; `kubectl exec` in and test locally with `curl localhost:<port>`.
4. **Scenario:** `kubectl get pods` returns nothing, but you're sure you created Pods. **Diagnosis steps:** Check current namespace with `kubectl config view --minify | grep namespace`; try `-A`.
5. **Scenario:** A namespace is stuck in `Terminating`. **Diagnosis steps:** Check for finalizers with `kubectl get namespace <name> -o json`.

## Volume 1 Cheat Sheet

| Task | Command |
|---|---|
| List nodes | `kubectl get nodes -o wide` |
| List all Pods, all namespaces | `kubectl get pods -A` |
| Describe a Pod | `kubectl describe pod <name>` |
| View logs | `kubectl logs <pod>` |
| Exec into a Pod | `kubectl exec -it <pod> -- sh` |
| Apply a manifest | `kubectl apply -f file.yaml` |
| Delete by label | `kubectl delete pods -l key=value` |
| Create namespace | `kubectl create namespace <name>` |
| Set default namespace | `kubectl config set-context --current --namespace=<ns>` |
| Label a resource | `kubectl label pod <name> key=value` |
| Annotate a resource | `kubectl annotate pod <name> key=value` |
| Watch Pod status | `kubectl get pod <name> -w` |
| Get raw JSON field | `kubectl get pod <name> -o jsonpath='{.status.phase}'` |
| List cluster-scoped resources | `kubectl api-resources --namespaced=false` |
| Create Kind cluster | `kind create cluster --name <name> --config=<file>` |
| Start Minikube | `minikube start --driver=docker` |

## kubectl Command Reference (Volume 1 subset)

`get`, `describe`, `apply`, `create`, `delete`, `logs`, `exec`, `label`, `annotate`, `config get-contexts`, `config use-context`, `config set-context`, `config current-context`, `api-resources`, `run`, `explain` (try `kubectl explain pod.spec.containers` to view built-in field documentation directly from your terminal).

## YAML Reference (Volume 1 subset)

Minimum required fields for any object: `apiVersion`, `kind`, `metadata.name`. For a Pod, `spec.containers` (a list, each requiring at minimum `name` and `image`) is also required. Optional but strongly recommended fields covered in this volume: `metadata.labels`, `metadata.annotations`, `metadata.namespace`, `spec.containers[].resources.requests/limits`, `spec.containers[].livenessProbe`, `spec.containers[].readinessProbe`, `spec.restartPolicy`.

---

*End of Volume 1 — Kubernetes Fundamentals. Continue to Volume 2: Pods & Workloads.*
