# Kubernetes Training Manual

## Volume 3 — Networking & Services

**A Self-Study Handbook for Beginners to Intermediate Learners**

---

### Copyright Notice

© 2026. This training manual is an original work created for self-study purposes. All commands, YAML examples, diagrams, and explanations in this document have been written from scratch. Kubernetes®, Docker®, and related marks are trademarks of their respective owners (The Linux Foundation, Docker Inc.). This manual is not affiliated with or endorsed by any of these organizations.

---

### About This Volume

Volume 2 gave you every major workload controller, but none of those workloads could reliably talk to each other or to the outside world — Pod IPs are ephemeral and change every time a Pod is replaced. Volume 3 solves that problem completely. You will learn Services (ClusterIP, NodePort, LoadBalancer), Ingress and Ingress Controllers for HTTP routing, how cluster DNS and CoreDNS work, Network Policies for traffic security, and the service discovery patterns that tie it all together.

**Prerequisites for this volume:** Completion of Volumes 1 and 2, including comfort with Deployments, DaemonSets, and label selectors.

---

## Table of Contents — Volume 3

- Lab 1 — Services & ClusterIP
- Lab 2 — NodePort
- Lab 3 — LoadBalancer
- Lab 4 — Ingress Basics
- Lab 5 — Ingress Controllers
- Lab 6 — DNS Fundamentals
- Lab 7 — CoreDNS Deep Dive
- Lab 8 — Network Policies
- Lab 9 — Service Discovery Patterns
- Lab 10 — Mini Project
- Volume 3 Review, Exam, and Reference

---

# Lab 1 — Services & ClusterIP

**Estimated Duration:** 55 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain why Pods need a stable networking abstraction in front of them.
2. Create and use a ClusterIP Service, the default Service type.
3. Understand how Services use label selectors to find their target Pods.
4. Inspect Endpoints/EndpointSlices to see the underlying Pod IPs a Service routes to.

### Prerequisites
- Completion of Volume 2, especially Lab 4 (Deployments).

### Theory

Every Pod gets its own IP address, but that IP is not stable — when a Pod is replaced (crash, rolling update, rescheduling), the new Pod gets a **different** IP. If other components hard-coded that IP, they would break constantly. A **Service** solves this by providing a single, stable virtual IP address and DNS name that automatically routes to whichever healthy Pods currently match its label selector — the exact same selector mechanism from Volume 1.

The default Service type is **ClusterIP**: it allocates a stable, internal-only IP address reachable only from inside the cluster. This is the right choice for internal communication between components — for example, a frontend Deployment reaching a backend Deployment — and is what you'll use for the vast majority of Services in a real cluster (NodePort and LoadBalancer, covered in Labs 2–3, exist specifically to expose a Service externally).

Under the hood, a Service doesn't proxy traffic itself — `kube-proxy` (from Volume 1's architecture lab) watches Services and their matching Pods, and programs networking rules (iptables or IPVS, depending on cluster configuration) on every node so that traffic sent to the Service's virtual IP is transparently routed to one of the matching, healthy Pod IPs. The Kubernetes API also maintains an **Endpoints** (or, in modern clusters, **EndpointSlice**) object per Service, listing the current set of matching Pod IPs — a very useful thing to inspect when debugging.

### Architecture Diagram

```
              Service: web-svc (ClusterIP: 10.96.10.5)
                          |
              selector: app=web  (watches matching Pods)
                          |
        +-----------------+-----------------+
        |                 |                 |
   Pod: web-1         Pod: web-2         Pod: web-3
   IP: 10.244.1.5     IP: 10.244.2.7     IP: 10.244.1.9

   Traffic to 10.96.10.5 is load-balanced across whichever
   Pods currently match app=web and are Ready.
```

### Installation
No new installation — uses your existing cluster and the `web` Deployment pattern from Volume 2.

### Hands-on Practice

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl expose deployment web --port=80 --target-port=80 --name=web-svc
kubectl get svc web-svc
```
**Purpose:** Creates a Deployment, then exposes it via a ClusterIP Service in one imperative step.
**Syntax:** `expose` automatically copies the Deployment's Pod template labels as the Service's selector.
**Expected Output:**
```
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
web-svc   ClusterIP   10.96.10.5     <none>        80/TCP    5s
```
**Common mistakes:** Confusing `--port` (the Service's own port) with `--target-port` (the container's port) — they're often the same value but conceptually distinct.
**Real-world usage:** The standard way to give any Deployment a stable internal address.

```bash
kubectl get endpoints web-svc
```
**Purpose:** Shows the actual Pod IPs currently backing the Service.
**Expected Output:** `web-svc   10.244.1.5:80,10.244.2.7:80,10.244.1.9:80   10s`
**Real-world usage:** The first thing to check when a Service "isn't working" — an empty Endpoints list almost always means the selector doesn't match any Ready Pods.

```bash
kubectl run debug --image=busybox --rm -it --restart=Never -- wget -qO- web-svc
```
**Purpose:** Confirms the Service is reachable and load-balancing by name from inside the cluster.
**Common mistakes:** Running this from outside the cluster and expecting it to work — ClusterIP is internal-only by design.
**Real-world usage:** Standard smoke test for internal service-to-service connectivity.

### YAML Files
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  type: ClusterIP           # Default type; can be omitted entirely
  selector:
    app: web                # Must match the target Pods' labels, exactly like ReplicaSets/Deployments
  ports:
    - port: 80               # Port the Service itself listens on
      targetPort: 80          # Port on the Pod/container to forward traffic to
      protocol: TCP
```

### Expected Output
```
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
web-svc   ClusterIP   10.96.10.5     <none>        80/TCP    30s
```

### Validation Steps
1. `kubectl get endpoints web-svc` lists exactly as many IPs as healthy, Ready Pods matching the selector.
2. A `wget`/`curl` from a debug Pod to `web-svc` (or `web-svc.default.svc.cluster.local`) succeeds and returns nginx's default page.

### Common Errors
- **Empty Endpoints list** — selector doesn't match any Pods, or matching Pods aren't Ready (failing readiness probes).
- **"Connection refused"** — `targetPort` doesn't match the actual container port.
- **Service works for some Pods but not others** — inconsistent labels across Pods that should all be matched.

### Troubleshooting
Always start with `kubectl get endpoints <service-name>` — an empty list points straight at a selector or readiness problem, not a networking bug. Confirm `targetPort` matches `containerPort` in the Pod spec. Use `kubectl get pods --show-labels` to confirm every intended Pod actually carries the expected label.

### Practice Tasks
1. Create a ClusterIP Service for a Deployment using YAML instead of `kubectl expose`.
2. Scale the Deployment and watch `kubectl get endpoints` update automatically.
3. Deliberately mismatch the selector and observe the resulting empty Endpoints list.
4. Expose two different ports from the same Pod via one Service using multiple `ports` entries with distinct `name` fields (required when a Service has more than one port).
5. Delete the Service (not the Deployment) and confirm the Pods keep running unaffected — Services and workloads are decoupled.

### Challenge Lab
Create two separate Deployments (`frontend`, `backend`) each with their own ClusterIP Service, and from a debug Pod, `wget` the `backend` Service using its short name and its full DNS name (`backend.default.svc.cluster.local` — proper DNS mechanics are covered fully in Lab 6), confirming both work identically from within the same namespace.

### Best Practices
- Always define a readiness probe on your Deployment's Pods — Services only route to Pods that are both selector-matched **and** Ready.
- Give Services clear, descriptive names, since they become part of internal DNS names other components will reference.
- Prefer `kubectl apply -f` with YAML manifests over `kubectl expose` for anything beyond quick local testing, so the Service is properly version-controlled.

### Interview Questions
1. **Q: Why do Services exist?** A: To provide a stable, unchanging virtual IP and DNS name in front of Pods whose individual IPs change constantly.
2. **Q: What is the default Service type?** A: ClusterIP.
3. **Q: Is a ClusterIP Service reachable from outside the cluster?** A: No, it is internal-only by design.
4. **Q: How does a Service know which Pods to route to?** A: Via a label selector, the same mechanism used by ReplicaSets and Deployments.
5. **Q: What component actually implements Service traffic routing on each node?** A: kube-proxy, by programming iptables/IPVS rules.
6. **Q: What object shows the actual Pod IPs backing a Service?** A: Endpoints (or EndpointSlice in modern clusters).
7. **Q: What's the difference between `port` and `targetPort` in a Service spec?** A: `port` is the Service's own listening port; `targetPort` is the port on the backing Pods/containers.
8. **Q: Does deleting a Service affect the underlying Pods?** A: No, Services and workloads are fully decoupled.
9. **Q: Why might a Service have an empty Endpoints list?** A: The selector doesn't match any Pods, or matching Pods are failing readiness checks.
10. **Q: What command quickly creates a Service from an existing Deployment?** A: `kubectl expose deployment <name> --port=<p> --target-port=<tp>`.

### Quiz
1. Default Service type: (a) NodePort (b) ClusterIP (c) LoadBalancer (d) ExternalName — **b**
2. ClusterIP Services are reachable: (a) From anywhere on the internet (b) Only from inside the cluster (c) Only from the same node (d) Never — **b**
3. Services find target Pods using: (a) Pod names (b) Label selectors (c) Node names (d) IP ranges — **b**
4. Which component programs the actual traffic rules? (a) kubelet (b) kube-proxy (c) etcd (d) kube-scheduler — **b**
5. Which object lists the actual backing Pod IPs? (a) Service (b) Endpoints/EndpointSlice (c) ReplicaSet (d) ConfigMap — **b**
6. `targetPort` refers to: (a) The Service's own port (b) The Pod/container's port (c) The node's port (d) An external port — **b**
7. Empty Endpoints usually means: (a) Cluster is down (b) Selector mismatch or unready Pods (c) DNS failure (d) Node failure — **b**
8. Deleting a Service: (a) Deletes its Pods (b) Leaves Pods running, unaffected (c) Deletes the Deployment (d) Crashes the cluster — **b**
9. Command to quickly expose a Deployment: (a) kubectl publish (b) kubectl expose deployment ... (c) kubectl service create (d) kubectl network expose — **b**
10. Services solve the problem of: (a) Node scheduling (b) Unstable, changing Pod IPs (c) Image pulling (d) Namespace creation — **b**

### Homework
Create three Deployments each exposed via its own ClusterIP Service, then from a debug Pod, `wget` each by name and document the response, alongside the output of `kubectl get endpoints` for each Service, to build intuition for how selectors map to live Pod IPs.

### Summary
Services provide the stable networking abstraction that dynamic, ever-changing Pod IPs need, using the same label selector mechanism from Volume 1. ClusterIP, the default type, is for internal-only communication. Lab 2 covers NodePort, the first mechanism for exposing a Service outside the cluster.

---

# Lab 2 — NodePort

**Estimated Duration:** 40 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain how NodePort exposes a Service outside the cluster.
2. Create a NodePort Service and access it via a node's IP.
3. Understand the NodePort port range and its limitations.
4. Recognize when NodePort is (and isn't) the right choice.

### Prerequisites
- Completion of Lab 1.

### Theory

A **NodePort** Service builds directly on top of ClusterIP — it still gets an internal ClusterIP, but additionally opens a specific port (from a default range of 30000–32767) on **every node** in the cluster. Any traffic arriving at `<any-node-IP>:<node-port>` is routed to the Service, and from there to a matching, Ready Pod, regardless of which node that Pod actually lives on.

NodePort is simple and requires no external infrastructure, which makes it useful for local development, demos, and bare-metal clusters without a cloud load balancer. However, it has real limitations for production use: the high, unmemorable port range, no built-in health-aware external load balancing across nodes, and the operational burden of clients needing to know a specific node's IP (which may change if that node is replaced). In most production environments, NodePort is used as an implementation detail underneath a LoadBalancer Service (Lab 3) or an Ingress Controller (Lab 5), rather than being exposed to end users directly.

### Architecture Diagram

```
   External Client
        |
        v
   Node A:31500  --or--  Node B:31500  --or--  Node C:31500   (same port on every node)
        |                     |                     |
        +---------------------+---------------------+
                              |
                    Service: web-nodeport
                              |
              routes to any Ready Pod matching selector,
              regardless of which node it's actually running on
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl expose deployment web --type=NodePort --port=80 --target-port=80 --name=web-nodeport
kubectl get svc web-nodeport
```
**Purpose:** Creates a NodePort Service, automatically assigning a port from the default range if not specified.
**Expected Output:**
```
NAME            TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
web-nodeport    NodePort   10.96.20.8    <none>        80:31500/TCP   5s
```
**Common mistakes:** Trying to set `nodePort` to an arbitrary value outside 30000–32767 — rejected unless the API server's range has been reconfigured.
**Real-world usage:** Quick local exposure for demos; rarely the final production access method.

```bash
# For a Kind cluster, get the container "node" IP directly:
docker inspect my-cluster-worker --format '{{ .NetworkSettings.Networks.kind.IPAddress }}'
curl http://<node-ip>:31500
```
**Purpose:** Confirms external reachability by hitting a node's IP directly on the NodePort.
**Common mistakes:** Forgetting that in Kind, "nodes" are Docker containers on an internal Docker network, requiring port-mapping tricks (`extraPortMappings` in the Kind cluster config) to reach from the host machine.
**Real-world usage:** For Minikube, `minikube service web-nodeport --url` conveniently prints a directly-usable URL, handling this complexity for you.

```bash
minikube service web-nodeport --url
```
**Purpose:** (Minikube only) Prints a ready-to-use URL for a NodePort Service, tunneling through Minikube's networking automatically.
**Real-world usage:** The simplest way to manually test a NodePort Service when using Minikube specifically.

### YAML Files
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80              # ClusterIP-internal port, still allocated
      targetPort: 80         # Container port
      nodePort: 31500        # Optional: pin a specific port; omit to let Kubernetes auto-assign
```

### Expected Output
```
NAME           TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
web-nodeport   NodePort   10.96.20.8    <none>        80:31500/TCP   10s
```

### Validation Steps
`curl http://<any-node-ip>:31500` (or `minikube service web-nodeport --url` output) returns nginx's default page, and this works identically regardless of which specific node you target.

### Common Errors
- **"provided port is not in the valid range"** — a manually specified `nodePort` was outside 30000–32767.
- **Connection refused from outside the cluster** — for Kind specifically, host-to-container port mapping wasn't configured when the cluster was created.
- **Works on one node's IP but not another** — extremely rare, but would indicate a kube-proxy issue on the non-working node; typically NodePort works identically on every node by design.

### Troubleshooting
For invalid port range errors, either omit `nodePort` (letting Kubernetes auto-assign) or choose a value within 30000–32767. For Kind connectivity issues, recreate the cluster with an `extraPortMappings` block in its config mapping a host port to the desired NodePort. For Minikube, always prefer `minikube service <name> --url` over manually guessing the node IP.

### Practice Tasks
1. Create a NodePort Service and access it successfully from your host machine (using Minikube's `--url` helper or Kind's port mapping).
2. Manually pin `nodePort: 30080` instead of letting it auto-assign.
3. Try setting `nodePort: 25000` (outside the valid range) and read the resulting validation error.
4. Scale the backing Deployment and confirm the same NodePort continues working without any changes.
5. Compare `kubectl get svc web-svc web-nodeport` side by side, noting the `PORT(S)` column difference between ClusterIP and NodePort types.

### Challenge Lab
Reconfigure a Kind cluster (recreate it) with an `extraPortMappings` entry mapping host port 8080 to a NodePort Service's port 31500, and confirm `curl http://localhost:8080` on your host machine reaches the Service without needing to know any node's internal IP.

### Best Practices
- Avoid exposing NodePort Services directly to end users in production; prefer LoadBalancer (Lab 3) or Ingress (Labs 4–5) instead.
- Let Kubernetes auto-assign the `nodePort` unless you have a specific operational reason to pin one, to avoid port conflicts across multiple Services.
- Remember every node opens the same NodePort, even nodes not currently running a matching Pod — traffic is still correctly routed cluster-wide.

### Interview Questions
1. **Q: What does a NodePort Service do beyond ClusterIP?** A: Opens a specific port (30000–32767 by default) on every node, routing external traffic into the cluster.
2. **Q: What is the default NodePort range?** A: 30000–32767.
3. **Q: Does a NodePort Service still have a ClusterIP?** A: Yes, NodePort builds on top of ClusterIP.
4. **Q: If a Pod isn't running on a particular node, can that node's NodePort still route to it?** A: Yes — traffic reaching any node's NodePort is routed cluster-wide to a matching Pod, wherever it runs.
5. **Q: Why is NodePort rarely used directly in production?** A: Unmemorable high ports, no health-aware external load balancing, and dependency on potentially-changing node IPs.
6. **Q: What Minikube command simplifies accessing a NodePort Service?** A: `minikube service <name> --url`.
7. **Q: What happens if you specify a `nodePort` outside the valid range?** A: The API server rejects the Service with a validation error.
8. **Q: What is NodePort commonly used as an implementation detail for?** A: LoadBalancer Services and some Ingress Controller setups.
9. **Q: How do you let Kubernetes auto-assign the NodePort?** A: Simply omit the `nodePort` field.
10. **Q: What's a key operational risk of NodePort in production?** A: If clients hard-code a specific node's IP, losing that node breaks access.

### Quiz
1. Default NodePort range: (a) 8000-9000 (b) 30000-32767 (c) 80-443 (d) 1000-2000 — **b**
2. NodePort builds on top of: (a) LoadBalancer (b) ClusterIP (c) Ingress (d) DNS — **b**
3. Does every node open the same NodePort? (a) Yes (b) No — **a**
4. NodePort Services are best suited for: (a) Production end-user traffic (b) Local dev/demo/bare-metal access (c) Only internal traffic (d) DNS resolution — **b**
5. Minikube helper command for NodePort access: (a) minikube expose (b) minikube service <name> --url (c) minikube nodeport (d) minikube tunnel-service — **b**
6. Invalid nodePort values are: (a) Silently ignored (b) Rejected with a validation error (c) Auto-corrected (d) Allowed anyway — **b**
7. If nodePort is omitted: (a) Service fails (b) Kubernetes auto-assigns one (c) Defaults to 80 (d) Defaults to ClusterIP only — **b**
8. NodePort is commonly used underneath: (a) DNS (b) LoadBalancer Services / some Ingress setups (c) ConfigMaps (d) Namespaces — **b**
9. A key production risk of NodePort: (a) None (b) Dependency on a specific, possibly-changing node IP (c) Too secure (d) Too fast — **b**
10. NodePort still allocates: (a) Nothing else (b) A ClusterIP as well (c) A LoadBalancer IP (d) A DNS zone — **b**

### Homework
Set up a NodePort Service, deliberately delete and recreate the node it was primarily being accessed through (if using a multi-node Kind cluster), and document whether/how access continued to work through the other node's IP — reinforcing NodePort's cluster-wide (not node-specific) routing behavior.

### Summary
NodePort exposes a Service on a fixed port across every node, useful for local development but rarely ideal for production end-user traffic. Lab 3 covers LoadBalancer, which builds on NodePort to provide a single, cloud-provisioned external IP.

---

# Lab 3 — LoadBalancer

**Estimated Duration:** 45 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain how a LoadBalancer Service differs from NodePort and ClusterIP.
2. Create a LoadBalancer Service and understand its cloud-provider dependency.
3. Use `metallb` or Kind/Minikube workarounds to simulate LoadBalancer behavior locally.
4. Understand `externalTrafficPolicy` and its tradeoffs.

### Prerequisites
- Completion of Lab 2.

### Theory

A **LoadBalancer** Service is the highest-level of the three core Service types. It requests the cluster's underlying cloud provider (AWS, GCP, Azure, etc., via the `cloud-controller-manager` from Volume 1's architecture lab) to provision a real, external load balancer with its own dedicated public IP address, automatically pointing that load balancer at the Service's NodePort across all nodes. This is the standard way to expose a production HTTP/TCP service directly to the internet on a cloud-managed Kubernetes cluster.

On local tools like Kind or Minikube, there is no real cloud provider to fulfill this request, so a LoadBalancer Service's `EXTERNAL-IP` normally stays `<pending>` forever. Minikube works around this with `minikube tunnel`, which simulates a LoadBalancer by routing traffic from your host machine. For Kind, or for learning how real production clusters solve this, `MetalLB` is a popular open-source load-balancer implementation that can be installed into any cluster (cloud or bare-metal) to fulfill LoadBalancer Service requests using a pool of IPs you configure.

`spec.externalTrafficPolicy` controls an important tradeoff: `Cluster` (default) routes traffic to any node and then potentially forwards it again to reach the actual Pod (simpler, but can obscure the client's real source IP and adds an extra network hop); `Local` only routes to Pods on the node that received the traffic directly (preserves source IP, avoids the extra hop, but can cause uneven load if Pods aren't evenly spread across nodes).

### Architecture Diagram

```
        Internet
            |
   Cloud Load Balancer (public IP, provisioned automatically)
            |
     (routes to NodePort on any/all nodes)
            |
   +--------+--------+--------+
   | Node A  | Node B  | Node C |
   +--------+--------+--------+
            |
      Service routes to matching, Ready Pods
```

### Installation

**Minikube tunnel approach:**
```bash
minikube tunnel   # run in a separate terminal, keep it running
```

**MetalLB approach (works on Kind too):**
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
kubectl wait --namespace metallb-system --for=condition=ready pod --selector=app=metallb --timeout=90s
```
Then apply an IP address pool matching your local Docker network range (specifics vary by setup; consult MetalLB's documentation for the exact CIDR to use with Kind).

### Hands-on Practice

```bash
kubectl expose deployment web --type=LoadBalancer --port=80 --target-port=80 --name=web-lb
kubectl get svc web-lb -w
```
**Purpose:** Creates a LoadBalancer Service and watches for an `EXTERNAL-IP` to be assigned.
**Expected Output (with MetalLB or on a real cloud):**
```
NAME     TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
web-lb   LoadBalancer   10.96.30.2    172.18.255.1    80:31900/TCP   30s
```
**Common mistakes:** On plain Kind/Minikube without `tunnel` or MetalLB, expecting `EXTERNAL-IP` to ever leave `<pending>` — it won't, since there's no cloud provider or load-balancer implementation to fulfill the request.
**Real-world usage:** The standard way to expose production HTTP/TCP services directly on any real cloud-managed cluster (EKS, GKE, AKS).

```bash
curl http://<external-ip>
```
**Purpose:** Confirms the load balancer is correctly forwarding external traffic into the cluster.
**Real-world usage:** Basic smoke test after provisioning a new production Service.

### YAML Files
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
  externalTrafficPolicy: Local   # Preserves client source IP; traffic only goes to Pods on the receiving node
```

### Expected Output
```
NAME     TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
web-lb   LoadBalancer   10.96.30.2    172.18.255.1    80:31900/TCP   45s
```

### Validation Steps
`EXTERNAL-IP` transitions from `<pending>` to an actual IP address (given MetalLB, `minikube tunnel`, or a real cloud provider), and `curl`ing that IP returns a successful response from the backing application.

### Common Errors
- **`EXTERNAL-IP` stuck at `<pending>` forever** — no cloud provider or load-balancer implementation (like MetalLB) is available to fulfill the request; expected on plain local clusters.
- **Client source IP always shows as a node's internal IP, not the real client** — `externalTrafficPolicy` is `Cluster` (default); switch to `Local` if source IP preservation matters.
- **Uneven traffic distribution with `Local` policy** — Pods aren't evenly spread across nodes; consider pod anti-affinity (an advanced scheduling topic) to spread them out.

### Troubleshooting
For a stuck `<pending>` state locally, either run `minikube tunnel` (Minikube) or install MetalLB (Kind or any cluster). For source IP issues, switch `externalTrafficPolicy` to `Local`, understanding the tradeoff around uneven distribution. Always confirm with `curl -v` to actually observe the connecting behavior end-to-end.

### Practice Tasks
1. Create a LoadBalancer Service on Minikube and access it using `minikube tunnel`.
2. Compare the resulting `EXTERNAL-IP` behavior with and without `tunnel` running.
3. Switch `externalTrafficPolicy` between `Cluster` and `Local` and, if possible, observe the difference in preserved source IP using a simple logging backend.
4. Explain, in writing, why LoadBalancer Services are cloud-provider-dependent while ClusterIP and NodePort are not.
5. Estimate the cost implication of using one LoadBalancer Service per application versus consolidating many applications behind a single Ingress (a preview of Labs 4–5).

### Challenge Lab
Install MetalLB into your Kind cluster, configure an IP address pool matching your Docker network, create a LoadBalancer Service, and successfully `curl` its assigned external IP from your host machine — fully replicating cloud LoadBalancer behavior locally.

### Best Practices
- In real cloud environments, avoid creating one LoadBalancer Service per microservice — each one typically provisions a real (billable) cloud load balancer; prefer a single Ingress Controller (Labs 4–5) fronting many services instead.
- Choose `externalTrafficPolicy: Local` when preserving true client IPs matters (e.g., for IP-based access logging or rate limiting), but ensure Pods are well-distributed across nodes first.
- Treat local LoadBalancer simulation tools (MetalLB, `minikube tunnel`) purely as learning/testing aids — they don't need to mirror your exact production setup precisely.

### Interview Questions
1. **Q: What does a LoadBalancer Service provision?** A: A real, external, cloud-provider load balancer with a dedicated public IP, pointing at the Service's NodePort.
2. **Q: Why does `EXTERNAL-IP` stay `<pending>` on plain Kind/Minikube?** A: There's no cloud provider or load-balancer implementation available to fulfill the request.
3. **Q: What tool provisions LoadBalancer IPs on bare-metal or local clusters?** A: MetalLB.
4. **Q: What Minikube command simulates LoadBalancer behavior?** A: `minikube tunnel`.
5. **Q: What does `externalTrafficPolicy: Cluster` do?** A: Routes traffic to any node, potentially forwarding again to reach the Pod; simpler but can obscure client source IP.
6. **Q: What does `externalTrafficPolicy: Local` do?** A: Only routes to Pods on the node that received the traffic, preserving source IP but risking uneven load.
7. **Q: Why avoid one LoadBalancer per microservice in production?** A: Each typically provisions a real, billable cloud load balancer; an Ingress Controller can consolidate many services behind one.
8. **Q: Does LoadBalancer replace NodePort, or build on it?** A: It builds on NodePort under the hood.
9. **Q: What determines whether a cloud provider fulfills a LoadBalancer request?** A: The cluster's `cloud-controller-manager` integration with that specific cloud.
10. **Q: What's a tradeoff of `Local` traffic policy?** A: Uneven load distribution if Pods aren't spread evenly across nodes.

### Quiz
1. LoadBalancer Services provision: (a) A ClusterIP only (b) A real external cloud load balancer (c) An Ingress (d) A DNS record only — **b**
2. On plain Kind/Minikube, EXTERNAL-IP typically: (a) Assigns instantly (b) Stays pending without extra tooling (c) Errors out (d) Defaults to localhost — **b**
3. Which tool provisions LoadBalancer IPs on bare-metal clusters? (a) kube-proxy (b) MetalLB (c) CoreDNS (d) etcd — **b**
4. Minikube's LoadBalancer simulation command: (a) minikube lb (b) minikube tunnel (c) minikube expose (d) minikube bridge — **b**
5. externalTrafficPolicy: Local preserves: (a) Nothing (b) Client source IP (c) Node IP only (d) Pod IP only — **b**
6. Tradeoff of Local traffic policy: (a) None (b) Potential uneven load distribution (c) Always faster (d) Breaks Services — **b**
7. LoadBalancer builds on top of: (a) Ingress (b) NodePort (c) ClusterIP only, skipping NodePort (d) DNS — **b**
8. Best practice for many microservices in production: (a) One LoadBalancer each (b) Consolidate behind an Ingress Controller (c) Use only ClusterIP (d) Avoid Services entirely — **b**
9. What component integrates with the cloud to fulfill LoadBalancer requests? (a) kubelet (b) cloud-controller-manager (c) kube-scheduler (d) CoreDNS — **b**
10. externalTrafficPolicy: Cluster: (a) Always preserves source IP (b) May obscure source IP due to extra hop (c) Disables load balancing (d) Is invalid — **b**

### Homework
Install MetalLB on your local Kind cluster, provision a LoadBalancer Service successfully, and write a short comparison of the setup effort versus simply using NodePort or `minikube tunnel`, including when you'd realistically choose each approach.

### Summary
LoadBalancer Services provide the standard cloud-native way to expose a Service to the internet with a dedicated external IP, building on NodePort underneath. Local clusters need extra tooling (MetalLB, `minikube tunnel`) to simulate this. Lab 4 introduces Ingress, which solves the "one load balancer per service" cost and complexity problem for HTTP traffic specifically.

---

# Lab 4 — Ingress Basics

**Estimated Duration:** 55 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain what problem Ingress solves that Services alone cannot.
2. Write an Ingress resource routing by hostname and path.
3. Understand the relationship between an Ingress resource and an Ingress Controller.
4. Configure TLS termination at the Ingress layer conceptually.

### Prerequisites
- Completion of Lab 3.
- An Ingress Controller must be installed for Ingress resources to actually work (installation covered fully in Lab 5) — this lab focuses on writing correct Ingress YAML.

### Theory

Imagine running ten different HTTP microservices, each needing external access. Giving each one its own LoadBalancer Service means ten separate cloud load balancers — expensive and hard to manage. **Ingress** solves this by providing HTTP/HTTPS-aware routing rules — hostname-based and path-based — that let a **single** entry point (usually one LoadBalancer Service in front of an Ingress Controller) route incoming traffic to many different backend Services based on the request's URL.

An Ingress **resource** is just a set of routing rules — it does nothing by itself. It requires an **Ingress Controller** (Lab 5) — a separate piece of software (NGINX Ingress Controller, Traefik, and others are common choices) running in the cluster that watches Ingress resources and actually implements the routing, typically by configuring an embedded reverse proxy.

A typical Ingress rule structure: match a `host` (e.g., `shop.example.com`), then match a `path` (e.g., `/api`), and route matching requests to a specific backend `service.name` and `service.port`. Ingress also supports TLS termination — you reference a Secret (covered fully in Volume 4) containing a certificate and key, and the Ingress Controller handles HTTPS for you, so individual backend Services don't each need to implement TLS themselves.

### Architecture Diagram

```
                     Internet
                        |
              One LoadBalancer / NodePort
                        |
               Ingress Controller (e.g., NGINX)
                        |
        reads Ingress resources for routing rules
                        |
        +---------------+----------------+
        |                                |
  host: shop.example.com          host: api.example.com
  path: /                         path: /v1
        |                                |
        v                                v
  Service: shop-svc                Service: api-svc
```

### Installation
Covered fully in Lab 5. For this lab, assume an Ingress Controller is already running (or use `minikube addons enable ingress` from Volume 1 Lab 4 as a quick starting point).

### Hands-on Practice

```bash
kubectl apply -f shop-ingress.yaml
kubectl get ingress
```
**Purpose:** Creates an Ingress resource and shows its assigned address (once a controller is running).
**Expected Output:**
```
NAME            CLASS   HOSTS               ADDRESS        PORTS   AGE
shop-ingress    nginx   shop.example.com    172.18.255.2   80      10s
```
**Common mistakes:** Forgetting `ingressClassName`, especially in clusters with more than one Ingress Controller installed, leading to no controller picking up the resource at all.
**Real-world usage:** The standard way to expose many HTTP services behind one entry point.

```bash
curl -H "Host: shop.example.com" http://<ingress-address>/
```
**Purpose:** Tests hostname-based routing without needing real DNS configured, by manually setting the `Host` header.
**Common mistakes:** Forgetting the `Host` header entirely and being confused when the request doesn't match any rule.
**Real-world usage:** Standard local/staging testing technique before real DNS records point at the Ingress.

```bash
kubectl describe ingress shop-ingress
```
**Purpose:** Shows detailed routing rules and any recent Events related to the Ingress Controller processing this resource.
**Real-world usage:** First troubleshooting step when a route "isn't working" as expected.

### YAML Files

`shop-ingress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /    # Controller-specific annotation (NGINX example)
spec:
  ingressClassName: nginx        # Which Ingress Controller should handle this resource
  rules:
    - host: shop.example.com     # Route based on this hostname
      http:
        paths:
          - path: /
            pathType: Prefix     # Prefix, Exact, or ImplementationSpecific
            backend:
              service:
                name: shop-svc
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
  tls:
    - hosts: ["shop.example.com"]
      secretName: shop-tls-cert   # References a TLS Secret (Volume 4) for HTTPS termination
```

### Expected Output
```
NAME           CLASS   HOSTS                              ADDRESS        PORTS     AGE
shop-ingress   nginx   shop.example.com,api.example.com    172.18.255.2  80, 443   20s
```

### Validation Steps
`curl -H "Host: shop.example.com" http://<address>/` returns the `shop-svc` backend's response, and `curl -H "Host: api.example.com" http://<address>/v1` returns the `api-svc` backend's response, confirming host-based routing works correctly.

### Common Errors
- **`ADDRESS` stays empty** — no Ingress Controller is running or watching the specified `ingressClassName`.
- **404 from the Ingress Controller itself (not your backend)** — no rule matched the request's Host header or path.
- **502/503 Bad Gateway** — the rule matched, but the backend Service has no healthy Endpoints (same root cause as Lab 1's empty-Endpoints scenario).

### Troubleshooting
For an empty `ADDRESS`, confirm an Ingress Controller matching `ingressClassName` is actually deployed (Lab 5 covers this fully). For 404s, double check `host`/`path` spelling matches exactly what you're sending. For 502/503s, check the backend Service's Endpoints directly — the Ingress layer is working correctly; the problem is one layer down.

### Practice Tasks
1. Write an Ingress routing two different hostnames to two different backend Services.
2. Add a second path under one host routing to a different Service (e.g., `/` to `shop-svc`, `/admin` to `admin-svc`).
3. Test routing using `curl -H "Host: ..."` without configuring any real DNS.
4. Deliberately reference a nonexistent backend Service and observe the resulting 502/503 behavior.
5. Add a TLS block referencing a (for now, nonexistent) Secret and read the resulting Ingress Controller Events describing the missing certificate.

### Challenge Lab
Design an Ingress for a fictional multi-tenant SaaS platform routing `tenant-a.example.com` and `tenant-b.example.com` to two entirely separate backend Services, plus a shared `api.example.com/v1` path routed to a third Service — all in one Ingress resource.

### Best Practices
- Always set `ingressClassName` explicitly once more than one Ingress Controller might ever exist in a cluster, to avoid ambiguity.
- Prefer a small number of well-organized Ingress resources over many overlapping ones, to keep routing rules easy to audit.
- Never terminate TLS at the backend Pod level when an Ingress Controller can do it centrally — it's simpler to manage certificates in one place.

### Interview Questions
1. **Q: What problem does Ingress solve?** A: Letting a single entry point route HTTP/HTTPS traffic to many backend Services based on hostname/path, avoiding one load balancer per service.
2. **Q: Does an Ingress resource do anything by itself?** A: No — it requires an Ingress Controller to actually implement the routing rules.
3. **Q: What does `ingressClassName` specify?** A: Which Ingress Controller should handle a given Ingress resource.
4. **Q: What are the three `pathType` options?** A: Prefix, Exact, ImplementationSpecific.
5. **Q: How can you test Ingress routing without real DNS?** A: Send requests with a manually set `Host` header via `curl -H "Host: ..."`.
6. **Q: What does the `tls` block in an Ingress spec reference?** A: A Secret containing a TLS certificate and key for HTTPS termination.
7. **Q: What does a 502/503 from an Ingress Controller usually indicate, assuming the rule matched?** A: The backend Service has no healthy Endpoints.
8. **Q: What does a 404 directly from the Ingress Controller usually indicate?** A: No rule matched the request's Host/path.
9. **Q: Give a real-world benefit of centralizing TLS termination at the Ingress layer.** A: Certificates are managed in one place rather than duplicated across every backend service.
10. **Q: What common examples of Ingress Controllers exist?** A: NGINX Ingress Controller, Traefik, among others.

### Quiz
1. Does an Ingress resource work without a controller? (a) Yes (b) No — **b**
2. Which field selects the Ingress Controller? (a) spec.controller (b) spec.ingressClassName (c) metadata.class (d) spec.type — **b**
3. pathType options include: (a) Prefix, Exact, ImplementationSpecific (b) Strict, Loose (c) Full, Partial (d) Static, Dynamic — **a**
4. How to test routing without real DNS? (a) Impossible (b) curl with a manual Host header (c) Edit /etc/passwd (d) Use ping — **b**
5. TLS in Ingress references: (a) A ConfigMap (b) A Secret (c) A Namespace (d) A ServiceAccount — **b**
6. A 502/503 from the controller with a matched rule usually means: (a) DNS failure (b) No healthy backend Endpoints (c) Wrong node (d) Invalid YAML — **b**
7. A 404 from the controller usually means: (a) Backend crashed (b) No rule matched the request (c) TLS failure (d) Node down — **b**
8. Ingress primarily solves routing for: (a) TCP databases (b) HTTP/HTTPS traffic (c) UDP streaming (d) SSH — **b**
9. Example Ingress Controllers include: (a) kube-proxy (b) NGINX, Traefik (c) CoreDNS (d) etcd — **b**
10. Best practice for TLS: (a) Terminate at every Pod individually (b) Centralize at the Ingress layer (c) Never use TLS (d) Use ClusterIP for TLS — **b**

### Homework
Write a complete Ingress manifest for a fictional three-service application (web frontend, REST API, admin panel) using host-based routing for two services and path-based routing for the third, and explain your routing design choices in a short paragraph.

### Summary
Ingress provides HTTP/HTTPS-aware routing rules that let one entry point serve many backend Services, dramatically reducing the cost and complexity of exposing multiple applications. An Ingress resource alone does nothing without a controller — Lab 5 covers installing and understanding Ingress Controllers themselves.

---

# Lab 5 — Ingress Controllers

**Estimated Duration:** 55 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Install a real Ingress Controller into a local cluster.
2. Explain what an Ingress Controller does internally.
3. Understand IngressClass and how multiple controllers can coexist.
4. Diagnose common Ingress Controller installation issues.

### Prerequisites
- Completion of Lab 4.

### Theory

An **Ingress Controller** is ordinary cluster software — typically a Deployment (or DaemonSet) running a reverse proxy such as NGINX, plus a controller process that continuously watches Ingress resources via the Kubernetes API and reconfigures the proxy to match. It is exposed to the outside world via its own Service, usually of type `LoadBalancer` or `NodePort` (everything from Labs 2–3 applies directly here — the Ingress Controller's own exposure is just an ordinary Service).

The **IngressClass** resource formally declares an available controller (e.g., `nginx`, `traefik`) and can mark one as the cluster's default, used automatically for any Ingress resource that omits `ingressClassName`. Multiple Ingress Controllers can coexist in one cluster — for example, an internal-only controller for private services and a separate internet-facing one — with each Ingress resource explicitly choosing which to use.

Installing a real Ingress Controller is usually done via a single YAML manifest maintained by that project, or via Helm (a package manager for Kubernetes, referenced but not covered in depth in this manual). Minikube's built-in `ingress` addon is the simplest path for local learning, since it handles this installation for you.

### Architecture Diagram

```
   IngressClass: nginx (default: true)
        |
   Deployment: ingress-nginx-controller  (the actual reverse proxy + watcher)
        |
   Service: ingress-nginx-controller (type: LoadBalancer or NodePort)
        |
   Watches all Ingress resources cluster-wide, reconfigures itself
   automatically whenever an Ingress resource is created/updated/deleted
```

### Installation

**Option A — Minikube addon (simplest for local learning):**
```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

**Option B — Kind, via the official NGINX Ingress manifest:**
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

### Hands-on Practice

```bash
kubectl get ingressclass
```
**Purpose:** Lists all registered Ingress Controllers and which one (if any) is marked default.
**Expected Output:**
```
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       2m
```
**Common mistakes:** Assuming an IngressClass exists automatically without installing a controller — it's created by the controller's own installation manifest.
**Real-world usage:** First check when diagnosing "my Ingress has no address" issues.

```bash
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=50
```
**Purpose:** Confirms the controller Pod is healthy, and inspects its logs for reconciliation errors related to specific Ingress resources.
**Real-world usage:** Primary place to look when routing rules aren't being applied as expected, even though the Ingress resource itself looks correct.

```bash
kubectl apply -f shop-ingress.yaml   # from Lab 4
kubectl get ingress shop-ingress -w
```
**Purpose:** Confirms that once a real controller is running, a previously-pending Ingress resource picks up a real `ADDRESS`.

### YAML Files

Marking an IngressClass as default (only needed if not already set by the controller's installer):
```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"   # Used automatically when ingressClassName is omitted
spec:
  controller: k8s.io/ingress-nginx
```

### Expected Output
```
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       5m

NAMESPACE       NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx   ingress-nginx-controller-7d9f8c-abcde        1/1     Running   0          5m
```

### Validation Steps
1. `kubectl get ingressclass` shows at least one entry.
2. `kubectl get pods -n ingress-nginx` shows the controller Pod `Running` and `1/1` Ready.
3. An Ingress resource from Lab 4 now shows a real `ADDRESS` instead of staying empty.

### Common Errors
- **Controller Pod stuck `Pending`** — usually insufficient cluster resources on a small local cluster.
- **Ingress still shows no `ADDRESS` after installing a controller** — `ingressClassName` mismatch, or the controller's own Service has no external IP yet (Lab 3 concepts apply directly).
- **404 "default backend" page from the controller** — the controller is running correctly, but no Ingress rule matched the specific request.

### Troubleshooting
For a `Pending` controller Pod, check `kubectl describe pod` for resource-related Events and consider increasing your Kind/Minikube resource allocation. For a persistently empty `ADDRESS`, confirm `ingressClassName` matches exactly (`kubectl get ingressclass`) and check the controller's own Service type/status. A "default backend" 404 is actually a healthy signal — it means the controller works, but your specific request didn't match a rule; recheck Host headers and paths from Lab 4.

### Practice Tasks
1. Install the NGINX Ingress Controller via your chosen method and confirm its Pod becomes Ready.
2. Apply an Ingress resource and confirm it picks up a real address.
3. Deliberately set `ingressClassName: doesnotexist` on an Ingress and observe it never gets an address.
4. Tail the controller's logs while applying/removing Ingress resources and observe the reconciliation messages.
5. Mark your IngressClass as default and confirm an Ingress resource omitting `ingressClassName` is still picked up correctly.

### Challenge Lab
Install two different Ingress Controllers side by side in the same cluster (for example, NGINX and Traefik) with different `IngressClass` names, and create one Ingress resource for each, confirming via `kubectl get ingress` that each is served by its intended, distinct controller.

### Best Practices
- Always verify controller Pod health first (`kubectl get pods -n <controller-namespace>`) before debugging individual Ingress resources — a broken controller explains every downstream symptom at once.
- Explicitly set `ingressClassName` on every Ingress resource once a cluster has (or might ever have) more than one controller, even if only one exists today.
- Monitor Ingress Controller resource usage in production — as the single entry point for HTTP traffic, it is a critical, high-traffic component worth dedicated observability.

### Interview Questions
1. **Q: What is an Ingress Controller, concretely?** A: Ordinary cluster software (often a Deployment running a reverse proxy like NGINX) that watches Ingress resources and configures itself to implement their routing rules.
2. **Q: What does IngressClass represent?** A: A formally registered available Ingress Controller, which Ingress resources reference via `ingressClassName`.
3. **Q: How is an Ingress Controller itself typically exposed externally?** A: Via its own Service, usually type LoadBalancer or NodePort.
4. **Q: Can multiple Ingress Controllers coexist in one cluster?** A: Yes, each with its own IngressClass, with individual Ingress resources choosing which to use.
5. **Q: What's the simplest way to get an Ingress Controller running on Minikube?** A: `minikube addons enable ingress`.
6. **Q: What does a "default backend" 404 typically indicate?** A: The controller is healthy, but no Ingress rule matched the specific request.
7. **Q: What's the first thing to check when an Ingress has no address after installing a controller?** A: Whether `ingressClassName` matches an existing, registered IngressClass.
8. **Q: How do you mark an IngressClass as the cluster default?** A: With the `ingressclass.kubernetes.io/is-default-class: "true"` annotation.
9. **Q: Where would you look to debug a routing rule that "looks correct" but isn't working?** A: The Ingress Controller Pod's logs.
10. **Q: Why is the Ingress Controller a particularly important component to monitor in production?** A: It's typically the single entry point for all HTTP/HTTPS traffic into the cluster.

### Quiz
1. An Ingress Controller is: (a) A built-in Kubernetes core component (b) Ordinary software (often NGINX-based) you install (c) Always pre-installed (d) A CLI tool only — **b**
2. IngressClass represents: (a) A network policy (b) A registered, available Ingress Controller (c) A Service type (d) A namespace — **b**
3. How is a controller usually exposed externally? (a) Via its own Service (LoadBalancer/NodePort) (b) It cannot be exposed (c) Only via kubectl proxy (d) Automatically via DNS — **a**
4. Can multiple controllers coexist? (a) No (b) Yes, each with its own IngressClass — **b**
5. Simplest Minikube installation method: (a) Manual YAML only (b) minikube addons enable ingress (c) Not possible (d) Requires Helm — **b**
6. A "default backend" 404 usually means: (a) Controller is broken (b) Controller is healthy but no rule matched (c) DNS failure (d) Node down — **b**
7. First check for "no address" after installing a controller: (a) Restart cluster (b) Check ingressClassName matches (c) Reboot host (d) Check node CPU — **b**
8. Where to debug a "correct-looking" but non-working rule? (a) etcd (b) Controller Pod logs (c) kube-scheduler logs (d) DNS logs — **b**
9. Default IngressClass annotation: (a) default: true (b) ingressclass.kubernetes.io/is-default-class: "true" (c) spec.default (d) metadata.default — **b**
10. Why closely monitor the Ingress Controller in production? (a) No reason (b) It's the single entry point for HTTP traffic (c) It never fails (d) It's optional — **b**

### Homework
Install an Ingress Controller from scratch (not via the Minikube addon shortcut) following only official upstream documentation you look up yourself, apply the Lab 4 Ingress resource against it, and document every step and any issues encountered.

### Summary
Ingress Controllers are the real software implementing the routing rules described by Ingress resources, typically exposed via their own Service and configurable via IngressClass. With routing now understood end-to-end, Lab 6 turns to DNS — how Pods and Services actually resolve each other's names inside the cluster.

---

# Lab 6 — DNS Fundamentals

**Estimated Duration:** 40 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Explain the standard DNS naming pattern for Services.
2. Resolve Services and Pods by name from within the cluster.
3. Understand the search domain suffixes applied automatically inside Pods.
4. Explain headless Service DNS behavior (connecting back to Volume 2's StatefulSets).

### Prerequisites
- Completion of Lab 5.

### Theory

Every Kubernetes cluster runs an internal DNS service (implemented by CoreDNS, covered in depth in Lab 7) that automatically creates DNS records for every Service. The standard format is:

```
<service-name>.<namespace>.svc.cluster.local
```

For example, a Service named `backend` in the `default` namespace resolves at `backend.default.svc.cluster.local`. From within the **same** namespace, you can use just the short name `backend`, since every Pod's `/etc/resolv.conf` is automatically configured with a search domain list that tries the current namespace first. From a **different** namespace, you need at least `backend.other-namespace` (or the fully qualified name) since the short name alone won't resolve to the right namespace.

Resolving `backend.default.svc.cluster.local` for a normal (non-headless) Service returns the Service's single, stable ClusterIP — DNS doesn't do the load balancing itself; it just hands back that one IP, and kube-proxy's routing rules handle spreading traffic across Pods from there. A **headless** Service (from Volume 2's StatefulSet lab, `clusterIP: None`) behaves completely differently: resolving its name returns the individual Pod IPs directly (one A record per backing Pod) rather than a single virtual IP, and each Pod additionally gets its own predictable name (`<pod-name>.<headless-service-name>...`), which is exactly how StatefulSet Pods address each other.

Pods themselves can also be resolved directly (less commonly used) via `<pod-ip-with-dashes>.<namespace>.pod.cluster.local`, though Service-based and StatefulSet-per-Pod DNS are what you'll use in almost all real applications.

### Architecture Diagram

```
   Pod's /etc/resolv.conf search list (typical):
     default.svc.cluster.local
     svc.cluster.local
     cluster.local

   Resolving "backend"  -> tries default.svc.cluster.local first -> matches -> ClusterIP returned
   Resolving "backend.other-ns" -> namespace explicit -> ClusterIP in other-ns returned
   Resolving "backend.default.svc.cluster.local" -> fully qualified, always works, from any namespace
```

### Installation
No new installation — cluster DNS is installed automatically with Kind/Minikube.

### Hands-on Practice

```bash
kubectl run debug --image=busybox --rm -it --restart=Never -- nslookup backend
```
**Purpose:** Resolves a Service by its short name from within the same namespace.
**Expected Output:**
```
Name:      backend.default.svc.cluster.local
Address 1: 10.96.20.8 backend.default.svc.cluster.local
```
**Common mistakes:** Running this same short-name lookup from a debug Pod in a *different* namespace and being confused why it fails — the search domain only covers the Pod's own namespace by default.
**Real-world usage:** Standard first debugging step for "my app can't reach the database" issues.

```bash
kubectl run debug --image=busybox --rm -it --restart=Never -- cat /etc/resolv.conf
```
**Purpose:** Shows exactly which search domains and nameserver a Pod is configured with.
**Expected Output:**
```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```
**Real-world usage:** Confirms exactly why a short name resolves (or doesn't) in a given namespace context.

```bash
kubectl run debug --image=busybox --rm -it --restart=Never -- nslookup db-0.db-headless.default.svc.cluster.local
```
**Purpose:** Resolves a specific StatefulSet Pod's stable DNS name (revisiting Volume 2 Lab 8), confirming headless Service DNS behavior directly.

### YAML Files
No new manifests are required for this lab — it exercises DNS resolution against Services and StatefulSets already created in prior labs. If needed for practice, recreate a simple Service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 80
```

### Expected Output
```
Name:      backend.default.svc.cluster.local
Address 1: 10.96.20.8 backend.default.svc.cluster.local
```

### Validation Steps
1. Short-name resolution (`nslookup backend`) succeeds from the same namespace.
2. The same short name fails (or resolves incorrectly) from a different namespace, while the fully qualified name always succeeds from anywhere.
3. A headless Service name resolves to multiple individual Pod IPs rather than one ClusterIP.

### Common Errors
- **"server can't find backend: NXDOMAIN"** — either the Service doesn't exist, or you're resolving a short name from the wrong namespace.
- **Resolves to an IP, but connections still fail** — DNS is working correctly; the problem is one layer down (Endpoints/readiness, per Lab 1).
- **Confusing Pod IP-based DNS with Service DNS** — different formats and different use cases; Service DNS is what you want almost always.

### Troubleshooting
For NXDOMAIN errors, always check `cat /etc/resolv.conf` first to understand exactly what search domains are being tried, then use the fully qualified name (`<svc>.<ns>.svc.cluster.local`) to eliminate namespace ambiguity while debugging. If DNS resolves correctly but connections still fail, the issue is downstream (Service Endpoints, readiness, or NetworkPolicy from Lab 8), not DNS itself.

### Practice Tasks
1. Resolve a Service by short name, then by `<name>.<namespace>`, then by fully qualified name, confirming all three work from the same namespace.
2. Attempt short-name resolution from a different namespace and observe the failure.
3. Compare `nslookup` output for a normal Service versus a headless Service from Volume 2.
4. Inspect `/etc/resolv.conf` in a few different Pods and confirm they're consistent.
5. Time how DNS caching (if any, depending on your CoreDNS configuration) affects repeated lookups.

### Challenge Lab
Create Services with identical short names (`backend`) in two different namespaces, and from a third namespace, write out the exact fully-qualified names you'd need to unambiguously reach each one, then verify both resolve correctly.

### Best Practices
- Always use the shortest reliable name for your context (short name within the same namespace) for cleaner configuration, but understand fully qualified names as the unambiguous fallback for cross-namespace or debugging scenarios.
- Never hardcode IP addresses in application configuration — always use DNS names, since IPs (even Service ClusterIPs, in rare cluster-rebuild scenarios) are not guaranteed stable forever.
- When debugging connectivity, always separate DNS resolution from actual connectivity as two distinct troubleshooting steps.

### Interview Questions
1. **Q: What is the standard DNS format for a Kubernetes Service?** A: `<service-name>.<namespace>.svc.cluster.local`.
2. **Q: Can you resolve a Service using just its short name?** A: Yes, but only reliably from within the same namespace, due to the Pod's configured search domains.
3. **Q: What does resolving a normal Service's DNS name return?** A: The Service's single, stable ClusterIP.
4. **Q: What does resolving a headless Service's DNS name return?** A: Individual Pod IPs directly (one record per backing Pod), not a single virtual IP.
5. **Q: Where can you see a Pod's configured DNS search domains?** A: In `/etc/resolv.conf` inside the Pod.
6. **Q: If DNS resolves correctly but the connection still fails, where should you look next?** A: Service Endpoints/readiness, or NetworkPolicies.
7. **Q: What does NXDOMAIN typically indicate?** A: The name doesn't exist as queried — often a missing Service or an incomplete short name used from the wrong namespace.
8. **Q: How do StatefulSet Pods get individually addressable names?** A: Through DNS records provided by a headless Service, one per Pod ordinal.
9. **Q: Why should applications use DNS names instead of hardcoded IPs?** A: IPs can change; DNS names remain stable and are the intended, supported mechanism.
10. **Q: What component actually implements this internal DNS resolution?** A: CoreDNS, covered in depth in Lab 7.

### Quiz
1. Standard Service DNS format: (a) <svc>.cluster (b) <svc>.<namespace>.svc.cluster.local (c) <namespace>.<svc> (d) <svc>.local — **b**
2. Short-name resolution works reliably from: (a) Any namespace (b) The same namespace only (c) Nowhere (d) Only kube-system — **b**
3. Resolving a normal Service returns: (a) All Pod IPs (b) The Service's single ClusterIP (c) Node IPs (d) Nothing — **b**
4. Resolving a headless Service returns: (a) One ClusterIP (b) Individual Pod IPs (c) Nothing (d) Node IPs — **b**
5. Where to see a Pod's DNS search domains: (a) kubectl get svc (b) /etc/resolv.conf inside the Pod (c) /etc/hosts (d) kubeconfig — **b**
6. NXDOMAIN typically means: (a) Success (b) The name doesn't exist as queried (c) Network is down (d) TLS failure — **b**
7. If DNS works but connection fails, check: (a) Nothing further needed (b) Endpoints/readiness or NetworkPolicy (c) Only DNS again (d) Node hardware — **b**
8. StatefulSet Pods get individual DNS names via: (a) Regular ClusterIP Services (b) Headless Services (c) NodePort (d) Ingress — **b**
9. Applications should use: (a) Hardcoded IPs (b) DNS names (c) MAC addresses (d) Node names — **b**
10. What implements internal cluster DNS? (a) kube-proxy (b) CoreDNS (c) etcd (d) kubelet — **b**

### Homework
Create Services in three different namespaces, all with different names, and write a short reference table documenting the shortest valid name to reach each one from every other namespace, testing each with `nslookup` to confirm your table is correct.

### Summary
Kubernetes DNS gives every Service a predictable, stable name, with short names resolving within a namespace and fully qualified names working from anywhere — and headless Services behaving distinctly for per-Pod addressing. Lab 7 goes one level deeper into CoreDNS itself, the component that implements all of this.

---

# Lab 7 — CoreDNS Deep Dive

**Estimated Duration:** 45 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Locate and inspect the CoreDNS Deployment running in any cluster.
2. Read and understand a basic CoreDNS `Corefile` configuration.
3. Diagnose CoreDNS-specific failures separately from application-level DNS issues.
4. Understand CoreDNS's plugin-based architecture at a high level.

### Prerequisites
- Completion of Lab 6.

### Theory

**CoreDNS** is the DNS server that implements everything from Lab 6, running as an ordinary Deployment (typically with 2 replicas for availability) in the `kube-system` namespace, fronted by its own Service (conventionally named `kube-dns` for historical compatibility, even though the software itself is CoreDNS, not the older `kube-dns` project it replaced).

CoreDNS is configured via a **Corefile**, stored in a ConfigMap (ConfigMaps are covered fully in Volume 4, but you'll see one here) in `kube-system`. The Corefile uses a plugin-chain architecture — each line represents a plugin handling some aspect of DNS resolution: the `kubernetes` plugin implements the Service/Pod DNS records covered in Lab 6, the `forward` plugin sends non-cluster queries (e.g., `google.com`) upstream to your configured external DNS resolvers, and the `cache` plugin caches results for performance.

Because CoreDNS is just a normal Deployment, it can fail like any other workload — CrashLoopBackOff from a malformed Corefile, resource exhaustion under heavy query load, or simply being scaled to zero accidentally. When *every* Service and Pod DNS lookup fails cluster-wide (not just one specific Service), CoreDNS itself, rather than any individual application, is almost always the actual root cause.

### Architecture Diagram

```
   kube-system namespace
   +----------------------------------------------+
   |  ConfigMap: coredns (contains the Corefile)     |
   |            |                                    |
   |            v                                    |
   |  Deployment: coredns (2 replicas)                |
   |    Pod: coredns-xxxx     Pod: coredns-yyyy        |
   |            |                                    |
   |  Service: kube-dns (ClusterIP: 10.96.0.10)        |
   +----------------------------------------------+
                    ^
                    |
   Every Pod's /etc/resolv.conf "nameserver" points here
```

### Installation
No new installation — CoreDNS ships automatically with Kind and Minikube.

### Hands-on Practice

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get deployment coredns -n kube-system
```
**Purpose:** Confirms CoreDNS Pods are healthy and shows the desired/current replica counts.
**Expected Output:**
```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-76f75df574-abcde   1/1     Running   0          20m
coredns-76f75df574-fghij   1/1     Running   0          20m
```
**Common mistakes:** Overlooking `-n kube-system` and concluding "CoreDNS isn't installed" when it's simply in a different namespace than expected.
**Real-world usage:** First health check whenever DNS resolution fails cluster-wide rather than for just one Service.

```bash
kubectl get configmap coredns -n kube-system -o yaml
```
**Purpose:** Views the actual Corefile configuration currently in effect.
**Expected Output:** A YAML-wrapped Corefile showing plugin chains like `kubernetes cluster.local ...`, `forward . /etc/resolv.conf`, and `cache 30`.
**Real-world usage:** Necessary when customizing DNS behavior, e.g., adding a custom internal domain or changing upstream resolvers.

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```
**Purpose:** Views CoreDNS's own logs, useful for spotting malformed queries, plugin errors, or crash loops.
**Real-world usage:** Primary diagnostic step when DNS is failing cluster-wide and Pod health alone doesn't explain why.

### YAML Files

A simplified example Corefile (as stored inside the `coredns` ConfigMap):
```
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```
Each line is a plugin: `kubernetes` implements the cluster's Service/Pod DNS records from Lab 6; `forward` sends anything outside `cluster.local` (e.g., public internet domains) to the upstream resolvers listed in `/etc/resolv.conf` on the CoreDNS Pod's own host; `cache` caches answers for 30 seconds to reduce load.

### Expected Output
```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-76f75df574-abcde   1/1     Running   0          25m
coredns-76f75df574-fghij   1/1     Running   0          25m
```

### Validation Steps
1. `kubectl get pods -n kube-system -l k8s-app=kube-dns` shows all replicas `Running` and `1/1` Ready.
2. `kubectl run debug --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default` (resolving the cluster's own API service) succeeds — a reliable cluster-wide DNS health check.

### Common Errors
- **All DNS lookups failing cluster-wide** — CoreDNS Pods are down, crashing, or the Corefile is malformed.
- **CoreDNS in CrashLoopBackOff after a ConfigMap edit** — a syntax error was introduced into the Corefile.
- **External domains (e.g., google.com) failing to resolve, but internal cluster names work fine** — the `forward` plugin's upstream resolver configuration is broken.

### Troubleshooting
For cluster-wide DNS failure, always check CoreDNS Pod health and logs first, exactly as shown above, before assuming an application-level problem. For Corefile syntax errors after an edit, `kubectl describe pod` on a crashing CoreDNS Pod usually surfaces the parsing error directly in its Events or logs. For external-domain-only failures, verify the `forward` plugin's target and the underlying node's own DNS resolver configuration.

### Practice Tasks
1. Locate CoreDNS's Deployment, Service, and ConfigMap in your own cluster and record their exact names.
2. Read through your cluster's actual Corefile line by line and identify each plugin's purpose.
3. Scale CoreDNS to 0 temporarily (in a disposable test cluster only) and observe every DNS lookup in the cluster fail, then scale back up and confirm recovery.
4. Resolve `kubernetes.default` as a quick cluster-wide DNS health check and explain why this specific name is a reliable canary.
5. Review CoreDNS logs while performing several lookups from Lab 6 and correlate the log entries with your queries.

### Challenge Lab
(In a disposable test cluster only) intentionally introduce a syntax error into the CoreDNS ConfigMap, observe the resulting CrashLoopBackOff, and practice the full recovery: identify the error from logs/Events, correct the ConfigMap, and confirm CoreDNS returns to healthy and DNS resolution resumes cluster-wide.

### Best Practices
- Never edit the CoreDNS ConfigMap casually in a production cluster without a tested rollback plan — a broken Corefile takes down cluster-wide DNS immediately.
- Monitor CoreDNS Pod health and query latency in production; as the single resolver for the entire cluster, it is a critical shared dependency.
- Keep CoreDNS at 2 or more replicas in any cluster beyond local learning, for basic availability.

### Interview Questions
1. **Q: What is CoreDNS?** A: The DNS server implementing cluster-internal Service and Pod name resolution, running as a Deployment in kube-system.
2. **Q: Where is CoreDNS's configuration stored?** A: In a ConfigMap containing a Corefile, in the kube-system namespace.
3. **Q: What does the `kubernetes` CoreDNS plugin do?** A: Implements DNS records for Services and Pods based on the live cluster state.
4. **Q: What does the `forward` plugin do?** A: Sends queries outside the cluster domain (e.g., public internet names) to configured upstream resolvers.
5. **Q: What does the `cache` plugin do?** A: Caches DNS answers for a configured duration to reduce load and improve performance.
6. **Q: What Service name conventionally fronts CoreDNS?** A: `kube-dns`, for historical compatibility.
7. **Q: What's a reliable cluster-wide DNS health check?** A: Resolving `kubernetes.default`, the cluster's own API service.
8. **Q: If DNS fails for every Service, not just one, what should you check first?** A: CoreDNS Pod health and logs.
9. **Q: What typically causes CoreDNS to CrashLoopBackOff after a config change?** A: A syntax error introduced into the Corefile.
10. **Q: Why should CoreDNS run with multiple replicas?** A: For basic availability, since it's a critical, cluster-wide shared dependency.

### Quiz
1. CoreDNS runs as: (a) A kubelet plugin (b) An ordinary Deployment in kube-system (c) Part of etcd (d) A CLI tool — **b**
2. CoreDNS configuration is stored in: (a) A Secret (b) A ConfigMap (Corefile) (c) A YAML file on each node (d) Environment variables only — **b**
3. Which plugin implements Service/Pod DNS records? (a) forward (b) kubernetes (c) cache (d) errors — **b**
4. Which plugin sends external queries upstream? (a) kubernetes (b) forward (c) cache (d) loop — **b**
5. Conventional Service name fronting CoreDNS: (a) coredns-svc (b) kube-dns (c) dns-service (d) cluster-dns — **b**
6. Reliable cluster-wide DNS health check: (a) ping google.com (b) nslookup kubernetes.default (c) kubectl get pods (d) curl localhost — **b**
7. All-DNS-failing-cluster-wide symptom points to: (a) One broken app (b) CoreDNS itself (c) A single Pod (d) Ingress — **b**
8. CrashLoopBackOff after a Corefile edit usually means: (a) Normal behavior (b) A syntax error was introduced (c) Cluster is out of nodes (d) DNS cache expired — **b**
9. Why run CoreDNS with 2+ replicas? (a) No reason (b) Basic availability for a critical shared dependency (c) Required by law (d) Improves image pulls — **b**
10. CoreDNS's plugin-chain architecture means: (a) One monolithic config (b) Modular plugins each handling one concern (c) No configuration possible (d) Only works with NGINX — **b**

### Homework
Document your own cluster's full Corefile, plugin by plugin, in your own words, and write a short incident-response checklist (5–7 steps) for "DNS appears to be down cluster-wide," ordered from fastest/most-likely-useful check to slowest/most-invasive.

### Summary
CoreDNS is the ordinary-but-critical Deployment implementing all cluster DNS resolution, configured via a plugin-based Corefile stored in a ConfigMap. Understanding it as "just another Deployment that can fail" demystifies a component that's otherwise easy to treat as invisible infrastructure. Lab 8 shifts to Network Policies, which control *whether* traffic between Pods is allowed at all, independent of whether it can be resolved or routed.

---

# Lab 8 — Network Policies

**Estimated Duration:** 55 minutes
**Difficulty Level:** Intermediate–Advanced

### Learning Objectives
1. Explain the default (fully open) Pod networking model and why NetworkPolicies exist.
2. Write ingress and egress NetworkPolicy rules using label selectors.
3. Understand the "default deny" pattern and why it's a security best practice.
4. Confirm a CNI plugin actually enforces NetworkPolicies before relying on them.

### Prerequisites
- Completion of Lab 7.

### Theory

By default, Kubernetes networking is **fully open**: any Pod can reach any other Pod across the entire cluster, regardless of namespace, with no restrictions at all. This is simple but insecure — a compromised or misconfigured Pod in one namespace could freely reach a sensitive database Pod in a completely different namespace. A **NetworkPolicy** is a namespaced resource that restricts this, defining which traffic is allowed to/from a set of Pods (selected, as always, via labels).

Critically, NetworkPolicies are **allow-list only** — you cannot write a policy that explicitly "denies" traffic; instead, once *any* NetworkPolicy selects a given Pod for ingress (or egress), that Pod's traffic in that direction becomes default-deny, and only traffic matching at least one policy's rules is permitted. This leads to the standard **"default deny" pattern**: create an empty-rule policy selecting all Pods in a namespace, which blocks all traffic, then layer additional policies that explicitly re-allow only the specific, intended communication paths.

An essential and frequently-missed detail: **NetworkPolicies do nothing unless your CNI plugin supports enforcing them.** Some simple CNI plugins (including Kind's default `kindnet`) do not enforce NetworkPolicies at all — the objects can be created successfully but silently have no actual effect. Calico, Cilium, and several other CNI plugins do enforce them properly, and are commonly installed specifically for this capability.

### Architecture Diagram

```
   Before any NetworkPolicy: fully open
   Pod A (namespace: dev)  <---->  Pod B (namespace: prod)   [ALLOWED by default]

   After a "default deny" policy in prod, plus one allow rule:
   Pod A (namespace: dev)  --X-->  Pod B (namespace: prod)   [BLOCKED - no matching allow rule]
   Pod C (app=frontend)    ----->  Pod B (namespace: prod)   [ALLOWED - matches explicit rule]
```

### Installation

For Kind, install Calico for real NetworkPolicy enforcement (kindnet, the default, does not enforce policies):
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n kube-system --timeout=120s
```
Minikube: `minikube start --cni=calico` (must be set at cluster creation time).

### Hands-on Practice

```bash
kubectl apply -f default-deny.yaml -n prod
kubectl run debug --image=busybox -n dev --rm -it --restart=Never -- wget -qO- --timeout=3 db.prod
```
**Purpose:** Applies a default-deny policy to the `prod` namespace, then confirms cross-namespace traffic is now blocked.
**Expected Output:** The `wget` command times out or fails, confirming the policy is enforced (assuming a policy-enforcing CNI is installed).
**Common mistakes:** Testing this on a CNI that doesn't enforce policies (like default Kind) and wrongly concluding the YAML is broken, when actually enforcement itself is simply unavailable.
**Real-world usage:** Standard first step of a namespace-level "zero trust" network security posture.

```bash
kubectl apply -f allow-frontend-to-db.yaml -n prod
kubectl run debug --image=busybox -n prod --rm -it --restart=Never --labels="app=frontend" -- wget -qO- --timeout=3 db
```
**Purpose:** Adds a specific allow rule and confirms the intended, labeled traffic now succeeds despite the default-deny policy still being in effect.
**Real-world usage:** Incrementally re-opening only the specific, necessary communication paths after establishing a default-deny baseline.

### YAML Files

`default-deny.yaml` — blocks all ingress traffic to every Pod in the namespace:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}          # Empty selector = applies to ALL Pods in this namespace
  policyTypes:
    - Ingress               # This policy governs incoming traffic only
  # No "ingress" rules block at all = nothing is allowed in
```

`allow-frontend-to-db.yaml` — re-allows only frontend Pods to reach the database on port 5432:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-db
spec:
  podSelector:
    matchLabels:
      app: db               # This policy governs traffic TO Pods labeled app=db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend   # Only allow traffic FROM Pods labeled app=frontend
      ports:
        - protocol: TCP
          port: 5432
```

Restricting egress (outbound) traffic too:
```yaml
spec:
  podSelector:
    matchLabels: { app: db }
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: monitoring     # Only allow outbound traffic to the "monitoring" namespace
      ports:
        - protocol: TCP
          port: 9090
```

### Expected Output
```
$ kubectl get networkpolicy -n prod
NAME                        POD-SELECTOR   AGE
default-deny-ingress        <none>          2m
allow-frontend-to-db        app=db          1m
```

### Validation Steps
1. Before any policy: cross-namespace traffic to `db` succeeds.
2. After `default-deny-ingress`: all traffic to every Pod in `prod`, including previously-working traffic, is blocked.
3. After adding `allow-frontend-to-db`: only traffic from `app=frontend`-labeled Pods on port 5432 succeeds; everything else remains blocked.

### Common Errors
- **Policies created successfully but have zero effect** — the CNI plugin doesn't enforce NetworkPolicies (very common on default Kind).
- **Default-deny also blocks needed system traffic** — e.g., blocking DNS resolution to CoreDNS if egress rules are too restrictive without explicitly allowing DNS (UDP/TCP port 53) to `kube-system`.
- **Rule seems correct but still blocks traffic** — label selector typo, or forgetting a policy only governs the direction (`Ingress`/`Egress`) explicitly listed in `policyTypes`.

### Troubleshooting
First confirm your CNI plugin actually enforces NetworkPolicies (Calico, Cilium, etc.) — this single check resolves the majority of "my policy isn't working" confusion. When adding default-deny egress policies, always remember to explicitly allow DNS traffic to `kube-system` (typically UDP/TCP port 53), or literally everything will break, including name resolution needed for the Pod to function at all. Double check label selectors match exactly using `kubectl get pods --show-labels`.

### Practice Tasks
1. Confirm whether your own cluster's CNI enforces NetworkPolicies (install Calico if using default Kind).
2. Apply a default-deny-ingress policy to one namespace and confirm all previously-working traffic now fails.
3. Add a specific allow rule and confirm only the intended traffic is restored.
4. Write a default-deny-egress policy that still explicitly allows DNS traffic to kube-system, and confirm the Pod can still resolve names.
5. Write a policy allowing traffic only from a specific *namespace* (using `namespaceSelector`) rather than specific Pod labels.

### Challenge Lab
Design a full "zero trust" NetworkPolicy set for a three-tier application (`frontend`, `backend`, `db`) in one namespace: default-deny everything, then explicitly allow only `frontend -> backend` and `backend -> db`, blocking `frontend -> db` directly even though all three Pods share a namespace.

### Best Practices
- Always verify your CNI plugin enforces NetworkPolicies before relying on them for actual security — an unenforced policy provides a false sense of protection.
- Adopt "default deny" as a baseline in any namespace handling sensitive data, then incrementally allow only necessary paths.
- Never forget to explicitly allow DNS (port 53 to kube-system) in restrictive egress policies — this is the single most common way teams accidentally break their own applications.

### Interview Questions
1. **Q: What is the default Pod-to-Pod networking behavior in Kubernetes without any NetworkPolicies?** A: Fully open — any Pod can reach any other Pod cluster-wide.
2. **Q: Are NetworkPolicies allow-list or deny-list based?** A: Allow-list only; you cannot write an explicit "deny" rule.
3. **Q: What happens once any NetworkPolicy selects a Pod for ingress?** A: That Pod's ingress traffic becomes default-deny, except for traffic matching at least one policy's rules.
4. **Q: What is the "default deny" pattern?** A: An empty-selector, no-rules policy blocking all traffic in a namespace, layered with specific allow policies re-opening only necessary paths.
5. **Q: Do all CNI plugins enforce NetworkPolicies?** A: No — some, like Kind's default kindnet, do not enforce them at all; Calico and Cilium are common choices that do.
6. **Q: What's a common mistake with restrictive egress policies?** A: Forgetting to explicitly allow DNS traffic (port 53) to kube-system, breaking name resolution entirely.
7. **Q: How do NetworkPolicies select which Pods they apply to?** A: Via `podSelector`, using the same label mechanism as Services and Deployments.
8. **Q: Can a NetworkPolicy restrict traffic based on namespace rather than specific Pod labels?** A: Yes, using `namespaceSelector`.
9. **Q: What field(s) determine whether a policy governs incoming, outgoing, or both directions of traffic?** A: `spec.policyTypes` (`Ingress`, `Egress`, or both).
10. **Q: Why is verifying CNI enforcement support an essential first step?** A: Otherwise, policies may appear correctly configured while providing zero actual security enforcement.

### Quiz
1. Default Kubernetes Pod networking is: (a) Fully closed (b) Fully open (c) Namespace-restricted by default (d) Randomly restricted — **b**
2. NetworkPolicies are: (a) Deny-list based (b) Allow-list only (c) Both equally (d) Irrelevant to security — **b**
3. Once a Pod is selected by any ingress policy: (a) Nothing changes (b) It becomes default-deny except for matching rules (c) All traffic is allowed (d) The Pod is deleted — **b**
4. The "default deny" pattern involves: (a) One policy allowing everything (b) An empty-selector deny-all policy plus specific allow policies (c) Deleting all Pods (d) Disabling DNS — **b**
5. Does every CNI plugin enforce NetworkPolicies? (a) Yes, always (b) No, it depends on the CNI (c) Only on cloud providers (d) Only for Ingress — **b**
6. Kind's default CNI (kindnet): (a) Fully enforces NetworkPolicies (b) Does not enforce NetworkPolicies (c) Is required for enforcement (d) Blocks all traffic by default — **b**
7. A common mistake with restrictive egress policies: (a) Too permissive by default (b) Forgetting to allow DNS (port 53) (c) Not using labels (d) Overusing Ingress — **b**
8. NetworkPolicies select Pods using: (a) IP ranges only (b) Label selectors (podSelector) (c) Node names (d) Container images — **b**
9. namespaceSelector allows restricting traffic based on: (a) Specific Pod labels only (b) Namespace-level labels (c) Node labels (d) Nothing — **b**
10. Why check CNI enforcement support first? (a) Not necessary (b) Policies may have zero actual effect otherwise (c) It affects DNS only (d) It's required for Services — **b**

### Homework
Design and implement a full default-deny-plus-explicit-allow NetworkPolicy set for a fictional three-tier application in your own cluster (installing Calico first if needed), test every intended and unintended path with `wget`/`curl` from debug Pods, and document your full test matrix (source, destination, expected result, actual result).

### Summary
NetworkPolicies provide allow-list-based traffic control between Pods, most effectively used via the "default deny plus explicit allow" pattern — but only if your CNI plugin actually enforces them. Lab 9 ties Services, DNS, and NetworkPolicies together into broader service discovery patterns used in real applications.

---

# Lab 9 — Service Discovery Patterns

**Estimated Duration:** 45 minutes
**Difficulty Level:** Intermediate

### Learning Objectives
1. Compare DNS-based and environment-variable-based service discovery.
2. Understand ExternalName Services for referencing services outside the cluster.
3. Design a realistic multi-tier application's discovery and connectivity strategy.
4. Recognize common service discovery anti-patterns.

### Prerequisites
- Completion of Lab 8.

### Theory

**Service discovery** is the general problem of how one component finds another's current network location. You've already learned the primary Kubernetes-native mechanism throughout this volume: **DNS-based discovery**, using the stable Service names from Labs 1 and 6. This is the recommended approach for virtually all real applications, since it requires no special client-side logic — any application capable of a normal DNS lookup and TCP/HTTP connection works without modification.

A secondary, legacy mechanism also exists: Kubernetes automatically injects **environment variables** into every Pod for Services that existed *before* that Pod was created (e.g., `BACKEND_SERVICE_HOST`, `BACKEND_SERVICE_PORT`). This approach is fragile — it doesn't work for Services created after the Pod starts, and doesn't update if the Service later changes — so DNS-based discovery is strongly preferred in essentially all modern applications; environment-variable discovery is worth knowing about mainly to recognize it in legacy systems and avoid relying on it in new ones.

For referencing something **outside** the cluster (a managed cloud database, a third-party API) using the same internal DNS conventions your application already expects, Kubernetes provides the **ExternalName** Service type — it creates a DNS CNAME record pointing your internal Service name at an external DNS name, letting your application code use one consistent discovery pattern (`some-service-name`) regardless of whether the actual backend lives inside or outside the cluster.

Putting it all together, a well-designed multi-tier application on Kubernetes typically uses: ClusterIP Services for internal component-to-component traffic (Lab 1), an Ingress Controller for unified external HTTP entry (Labs 4–5), DNS names exclusively for how components find each other (Labs 6–7), NetworkPolicies to restrict which of those DNS-discoverable paths are actually permitted (Lab 8), and ExternalName Services for cleanly referencing external dependencies through that same internal naming convention.

### Architecture Diagram

```
   Application code always does: connect("payments-db")
                                        |
                    +-------------------+-------------------+
                    |                                       |
        Internal Kubernetes Service               ExternalName Service
        (ClusterIP, routes to Pods)               (CNAME -> external RDS endpoint)
                    |                                       |
              Pods in-cluster                    Managed database outside the cluster

   Same discovery pattern from the application's perspective either way.
```

### Installation
No new installation.

### Hands-on Practice

```bash
kubectl run debug --image=busybox --rm -it --restart=Never -- env | grep BACKEND
```
**Purpose:** Confirms the legacy environment-variable injection behavior for a Service that existed before this Pod started.
**Expected Output:** `BACKEND_SVC_SERVICE_HOST=10.96.20.8`, `BACKEND_SVC_SERVICE_PORT=80`
**Common mistakes:** Relying on these variables for a Service created *after* the Pod — they simply won't exist, silently breaking the application in a confusing way.
**Real-world usage:** Mostly encountered when working with older applications not originally designed for Kubernetes.

```bash
kubectl apply -f external-db.yaml
kubectl run debug --image=busybox --rm -it --restart=Never -- nslookup external-db
```
**Purpose:** Confirms an ExternalName Service correctly resolves to the configured external hostname via CNAME.
**Expected Output:** A CNAME pointing to `mydb.cloudprovider.example.com`, which then resolves further to that provider's actual IP.
**Real-world usage:** Referencing a managed cloud database or third-party API using the same internal naming convention as everything else.

### YAML Files

`external-db.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: mydb.cloudprovider.example.com   # Real external DNS name being aliased
```
Applications simply connect to `external-db` (or `external-db.default.svc.cluster.local`) exactly as they would any in-cluster Service — no special external-specific logic required in application code.

### Expected Output
```
Name:      external-db.default.svc.cluster.local
Address:   mydb.cloudprovider.example.com
```

### Validation Steps
1. `env | grep BACKEND` inside a Pod shows injected variables for pre-existing Services.
2. `nslookup external-db` resolves via CNAME to the configured external hostname.
3. An application using `external-db` as its connection target successfully reaches the real, external service.

### Common Errors
- **Application relying on environment variables breaks after a Service is recreated later** — classic ordering problem with env-var-based discovery.
- **ExternalName Service resolves, but connection fails** — the external hostname/service itself is unreachable, misconfigured, or blocked by network/firewall rules outside Kubernetes' control.
- **Mixing discovery patterns inconsistently across an application** — some components using DNS, others hardcoding IPs — leading to confusing, inconsistent failure modes.

### Troubleshooting
For environment-variable ordering issues, the fix is almost always "switch to DNS-based discovery" rather than trying to control Pod/Service creation order. For ExternalName connection failures, remember Kubernetes' role ends at DNS resolution — actual reachability of the external endpoint is outside the cluster's control and must be debugged using standard external network troubleshooting. Standardize on DNS-based discovery cluster-wide to avoid inconsistent patterns.

### Practice Tasks
1. Create a Service, then a Pod, and confirm the Pod has environment variables for that Service.
2. Create a second Service *after* the same Pod already exists, and confirm no corresponding environment variables appear — proving the ordering limitation directly.
3. Create an ExternalName Service pointing at a real public hostname (e.g., a well-known public API) and confirm DNS resolution works via `nslookup`.
4. Refactor a hypothetical application design (on paper) that currently relies on hardcoded IPs to use DNS-based discovery instead.
5. Document, in your own words, why DNS-based discovery is preferred over environment-variable-based discovery in virtually all modern applications.

### Challenge Lab
Design a complete service discovery strategy (as a short written document plus supporting YAML) for a fictional application with an in-cluster frontend, an in-cluster backend, and an external, cloud-managed database — using ClusterIP Services for the internal components and an ExternalName Service for the database, with every component using DNS-based discovery exclusively.

### Best Practices
- Standardize on DNS-based service discovery across your entire application; avoid environment-variable-based discovery in new code.
- Use ExternalName Services to give external dependencies the same discovery experience as in-cluster ones, simplifying application configuration.
- Document your service naming conventions clearly, since DNS names effectively become a permanent, load-bearing part of your application's architecture.

### Interview Questions
1. **Q: What are the two main service discovery mechanisms in Kubernetes?** A: DNS-based discovery and legacy environment-variable-based discovery.
2. **Q: Why is DNS-based discovery generally preferred?** A: It requires no special client logic and doesn't suffer from the Pod/Service creation-order limitations of environment variables.
3. **Q: What's the key limitation of environment-variable-based discovery?** A: It only works for Services that existed before the consuming Pod was created, and doesn't update on later Service changes.
4. **Q: What does an ExternalName Service do?** A: Creates a DNS CNAME record aliasing an internal Service name to an external DNS name.
5. **Q: Why use an ExternalName Service instead of hardcoding an external hostname in application code?** A: It keeps discovery consistent, letting the application use the same internal naming convention regardless of where the backend actually lives.
6. **Q: Does Kubernetes guarantee reachability for an ExternalName Service's target?** A: No — it only provides DNS resolution; actual network reachability is outside Kubernetes' control.
7. **Q: What environment variable naming pattern does Kubernetes inject for Services?** A: `<SERVICE_NAME>_SERVICE_HOST` and `<SERVICE_NAME>_SERVICE_PORT` (uppercased, hyphens converted to underscores).
8. **Q: What layer of Kubernetes networking from this volume do NetworkPolicies interact with service discovery through?** A: They restrict which DNS-discoverable paths are actually permitted, independent of whether the name resolves.
9. **Q: Why is consistent service discovery strategy important across a whole application?** A: Mixing patterns leads to confusing, inconsistent failure modes that are harder to debug.
10. **Q: What's the recommended default discovery approach for any new Kubernetes-native application?** A: DNS-based discovery via standard Service names.

### Quiz
1. Preferred service discovery mechanism in modern Kubernetes apps: (a) Environment variables (b) DNS-based (c) Hardcoded IPs (d) Manual configuration — **b**
2. Environment-variable discovery's key limitation: (a) Too fast (b) Only works for Services existing before the Pod started (c) Requires DNS (d) Works everywhere reliably — **b**
3. ExternalName Services create: (a) A ClusterIP (b) A DNS CNAME to an external name (c) A NodePort (d) A Deployment — **b**
4. Does Kubernetes guarantee ExternalName target reachability? (a) Yes (b) No, only DNS resolution — **b**
5. Injected environment variable pattern: (a) SVC_NAME_PORT (b) <NAME>_SERVICE_HOST / <NAME>_SERVICE_PORT (c) Random (d) None injected ever — **b**
6. Mixing discovery patterns across an app leads to: (a) Better reliability (b) Confusing, inconsistent failure modes (c) No effect (d) Faster performance — **b**
7. NetworkPolicies relate to service discovery by: (a) Providing DNS (b) Restricting which discoverable paths are actually permitted (c) Replacing DNS (d) Creating Services — **b**
8. Best practice for external dependencies: (a) Hardcode external IPs (b) Use ExternalName Services (c) Avoid connecting at all (d) Use environment variables only — **b**
9. Recommended default discovery approach for new apps: (a) Environment variables (b) DNS-based via standard Service names (c) Manual IP configuration (d) None needed — **b**
10. Why prefer DNS over env-var discovery? (a) No real reason (b) No special client logic needed and no creation-order limitation (c) DNS is always faster (d) Env vars are deprecated entirely — **b**

### Homework
Audit a hypothetical (or real, if you have access to one) multi-service application's connection configuration, identify every place it uses hardcoded IPs or environment-variable-based discovery, and rewrite a migration plan moving everything to consistent DNS-based discovery, including any ExternalName Services needed for outside dependencies.

### Summary
DNS-based discovery, using the stable Service names established throughout this volume, is the standard, recommended approach for Kubernetes-native applications — with ExternalName Services extending that same convention to external dependencies. Lab 10 combines everything from this volume into one complete, realistic networking mini project.

---

# Lab 10 — Mini Project

**Estimated Duration:** 100 minutes
**Difficulty Level:** Intermediate (capstone for Volume 3)

### Learning Objectives
1. Combine Services, Ingress, DNS, and NetworkPolicies into one realistic system.
2. Practice full end-to-end connectivity debugging using every tool from this volume.
3. Design and defend a complete networking architecture for a small multi-tier application.
4. Build confidence before advancing to Volume 4 (Storage & Configuration).

### Prerequisites
- Completion of Labs 1–9.
- An Ingress Controller (Lab 5) and, ideally, a policy-enforcing CNI plugin (Lab 8) installed.

### Theory

This mini project brings together a realistic three-tier application — `frontend`, `backend`, and `db` — using every major concept from this volume: each tier is a Deployment (Volume 2) fronted by a ClusterIP Service (Lab 1); the `frontend` is additionally exposed externally via an Ingress resource (Labs 4–5); every component finds every other component exclusively through DNS names (Labs 6–7); and NetworkPolicies (Lab 8) enforce that `frontend` can reach `backend`, `backend` can reach `db`, but `frontend` cannot reach `db` directly — a realistic security boundary for a real production system.

### Architecture Diagram

```
                        Internet
                            |
                  Ingress (host: app.example.com)
                            |
                    Service: frontend-svc (ClusterIP)
                            |
                    Deployment: frontend
                            |
                    (DNS: backend-svc)
                            |
                    Service: backend-svc (ClusterIP)
                            |
                    Deployment: backend
                            |
                    (DNS: db-svc)
                            |
                    Service: db-svc (ClusterIP)
                            |
                    Deployment: db

   NetworkPolicy allows: frontend->backend, backend->db
   NetworkPolicy blocks: frontend->db (direct)
```

### Installation
No new installation beyond the Ingress Controller and CNI already set up in Labs 5 and 8.

### Hands-on Practice / Project Steps

**Step 1 — Deploy all three tiers and their Services:**
```bash
kubectl apply -f three-tier-app.yaml
kubectl get deployments,svc
```

**Step 2 — Apply the Ingress:**
```bash
kubectl apply -f three-tier-ingress.yaml
kubectl get ingress
```

**Step 3 — Apply NetworkPolicies:**
```bash
kubectl apply -f three-tier-netpol.yaml
kubectl get networkpolicy
```

**Step 4 — Validate the full path from outside in:**
```bash
curl -H "Host: app.example.com" http://<ingress-address>/
```

**Step 5 — Validate internal DNS-based connectivity and policy enforcement:**
```bash
kubectl run debug --image=busybox --rm -it --restart=Never --labels="app=frontend" -- wget -qO- --timeout=3 backend-svc
kubectl run debug --image=busybox --rm -it --restart=Never --labels="app=frontend" -- wget -qO- --timeout=3 db-svc
```
The first should succeed; the second should time out, confirming the NetworkPolicy boundary is actually enforced.

### YAML Files

`three-tier-app.yaml` (Deployments + Services):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: frontend, labels: { app: frontend } }
spec:
  replicas: 2
  selector: { matchLabels: { app: frontend } }
  template:
    metadata: { labels: { app: frontend } }
    spec:
      containers:
        - name: frontend
          image: hashicorp/http-echo
          args: ["-text=hello from frontend"]
          ports: [{ containerPort: 5678 }]
          readinessProbe: { httpGet: { path: /, port: 5678 }, initialDelaySeconds: 3 }
---
apiVersion: v1
kind: Service
metadata: { name: frontend-svc }
spec:
  selector: { app: frontend }
  ports: [{ port: 80, targetPort: 5678 }]
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: backend, labels: { app: backend } }
spec:
  replicas: 2
  selector: { matchLabels: { app: backend } }
  template:
    metadata: { labels: { app: backend } }
    spec:
      containers:
        - name: backend
          image: hashicorp/http-echo
          args: ["-text=hello from backend"]
          ports: [{ containerPort: 5678 }]
          readinessProbe: { httpGet: { path: /, port: 5678 }, initialDelaySeconds: 3 }
---
apiVersion: v1
kind: Service
metadata: { name: backend-svc }
spec:
  selector: { app: backend }
  ports: [{ port: 80, targetPort: 5678 }]
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: db, labels: { app: db } }
spec:
  replicas: 1
  selector: { matchLabels: { app: db } }
  template:
    metadata: { labels: { app: db } }
    spec:
      containers:
        - name: db
          image: hashicorp/http-echo
          args: ["-text=hello from db"]
          ports: [{ containerPort: 5678 }]
          readinessProbe: { httpGet: { path: /, port: 5678 }, initialDelaySeconds: 3 }
---
apiVersion: v1
kind: Service
metadata: { name: db-svc }
spec:
  selector: { app: db }
  ports: [{ port: 80, targetPort: 5678 }]
```

`three-tier-ingress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: three-tier-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: frontend-svc, port: { number: 80 } }
```

`three-tier-netpol.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-frontend-to-backend }
spec:
  podSelector: { matchLabels: { app: backend } }
  policyTypes: [Ingress]
  ingress:
    - from: [{ podSelector: { matchLabels: { app: frontend } } }]
      ports: [{ protocol: TCP, port: 5678 }]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-backend-to-db }
spec:
  podSelector: { matchLabels: { app: db } }
  policyTypes: [Ingress]
  ingress:
    - from: [{ podSelector: { matchLabels: { app: backend } } }]
      ports: [{ protocol: TCP, port: 5678 }]
```

### Expected Output
```
$ curl -H "Host: app.example.com" http://<ingress-address>/
hello from frontend

$ (from a frontend-labeled debug Pod) wget -qO- backend-svc
hello from backend

$ (from a frontend-labeled debug Pod) wget -qO- --timeout=3 db-svc
wget: download timed out
```

### Validation Steps
1. External `curl` through the Ingress reaches `frontend`.
2. `frontend -> backend` succeeds (allowed by policy).
3. `frontend -> db` fails/times out (blocked by policy — no direct rule permits it).
4. `backend -> db` succeeds (allowed by its own policy).

### Common Errors
- **Everything blocked, including intended paths** — forgot that `db`'s NetworkPolicy only allows from `backend`, so testing `frontend -> db` correctly fails, but double-check `backend -> db` also works before assuming the whole policy set is broken.
- **Ingress shows no address** — revisit Lab 5's controller installation and health checks.
- **All internal traffic blocked including intended paths, even before adding policies** — check whether a `default-deny` policy from Lab 8 practice is still lingering in this namespace from earlier experimentation.

### Troubleshooting
Work through this project's connectivity layer by layer, exactly as structured in this volume: confirm DNS resolution first (Lab 6/7 techniques), then confirm Endpoints exist (Lab 1 techniques), then confirm NetworkPolicies allow the intended path specifically (Lab 8 techniques), and finally confirm the Ingress layer separately (Lab 4/5 techniques) for the external path. Isolating which layer is failing, rather than guessing, is the core skill this whole volume has been building toward.

### Practice Tasks
1. Build the full mini project from scratch without copying YAML directly, from memory.
2. Verify all four connectivity assertions from the Validation Steps section yourself.
3. Add a fourth tier (`cache`) and design (and implement) the correct NetworkPolicy rules allowing only `backend -> cache`.
4. Break the Ingress deliberately (wrong service name) and use Lab 4/5 techniques to diagnose it.
5. Break a NetworkPolicy deliberately (wrong label selector) and use Lab 8 techniques to diagnose it.

### Challenge Lab
Extend the mini project with TLS termination at the Ingress (self-signed certificate acceptable for practice, referencing Volume 4 Secrets conceptually), and write out the complete list of every Kubernetes object involved in serving a single external HTTPS request all the way through to the `db` tier's response, in the correct order.

### Best Practices
- Always build connectivity in layers and test each layer independently — this project's structure (DNS, then Endpoints, then NetworkPolicy, then Ingress) mirrors how real production debugging should proceed.
- Keep all NetworkPolicy, Service, and Ingress definitions in version control together with their corresponding Deployments, since they are all interdependent parts of one system.
- Document your intended traffic flow (which tier talks to which) before writing any NetworkPolicy YAML — policies enforce a design decision, they don't make it for you.

### Interview Questions
1. **Q: In this mini project, why can't `frontend` reach `db` directly?** A: No NetworkPolicy rule explicitly allows it, and `db`'s ingress policy only permits traffic from Pods labeled `app=backend`.
2. **Q: What's the correct debugging order when a multi-tier connectivity issue arises?** A: DNS resolution, then Service Endpoints, then NetworkPolicy rules, then Ingress (for the external-facing path).
3. **Q: Why is only `frontend-svc` exposed via Ingress, not `backend-svc` or `db-svc`?** A: `backend` and `db` are internal-only components that should never be reachable directly from outside the cluster.
4. **Q: What would happen if the NetworkPolicy for `db` were deleted entirely?** A: Traffic to `db` would become fully open again, since removing the only policy selecting it removes the default-deny restriction along with it.
5. **Q: Why use `hashicorp/http-echo` for this mini project instead of a real database image?** A: It's a lightweight way to demonstrate connectivity and routing concepts without the complexity of a real stateful backend.
6. **Q: How would you extend this project for HTTPS?** A: Add a `tls` block to the Ingress referencing a Secret containing a certificate and key.
7. **Q: What ensures `backend-svc` and `db-svc` are only reachable using their DNS names, not hardcoded IPs?** A: Nothing enforces this technically, but it's the recommended pattern from Lab 9 for reliability.
8. **Q: If `backend` Pods fail their readiness probe, what happens to traffic routed through `backend-svc`?** A: It stops being routed to those specific Pods, since Services only route to Ready Pods (Lab 1).
9. **Q: What's the risk of not having any NetworkPolicies in a project like this?** A: Any Pod, including ones outside this application entirely, could reach any tier directly, including `db`.
10. **Q: How does this mini project's structure mirror a real production application?** A: Layered internal Services, centralized external exposure via Ingress, DNS-based discovery, and explicit least-privilege network access control.

### Quiz
1. Which tier is exposed externally in this project? (a) db (b) backend (c) frontend (d) all three — **c**
2. Why can't frontend reach db directly? (a) DNS doesn't resolve it (b) No NetworkPolicy rule allows it (c) It's a different cluster (d) Ingress blocks it — **b**
3. Correct debugging order: (a) Ingress, DNS, Endpoints, NetworkPolicy (b) DNS, Endpoints, NetworkPolicy, Ingress (c) Random order (d) NetworkPolicy only — **b**
4. What exposes frontend to the internet? (a) NodePort directly (b) Ingress (c) NetworkPolicy (d) DNS — **b**
5. What connects frontend to backend internally? (a) Hardcoded IP (b) DNS name via ClusterIP Service (c) NodePort (d) Ingress — **b**
6. If db's NetworkPolicy is deleted: (a) Nothing changes (b) db becomes fully open again (c) db becomes unreachable entirely (d) Ingress breaks — **b**
7. Extending to HTTPS requires: (a) A NetworkPolicy change (b) A tls block referencing a Secret on the Ingress (c) A new Deployment (d) DNS changes only — **b**
8. If backend Pods fail readiness: (a) Traffic still routes to them (b) They stop receiving traffic via the Service (c) The Service is deleted (d) DNS breaks — **b**
9. Risk of no NetworkPolicies at all: (a) None (b) Any Pod could reach any tier directly, including db (c) Improved performance only (d) DNS stops working — **b**
10. This project's structure mirrors real production via: (a) Random configuration (b) Layered Services, centralized ingress, DNS discovery, least-privilege network policy (c) Single monolithic Pod (d) No networking at all — **b**

### Homework
Build the entire mini project from scratch, then write a one-page network architecture document (as if explaining it to a new team member) covering every object involved, the intended traffic flow, and exactly how you'd verify each hop is working correctly during an incident.

### Summary
This capstone combined Services, Ingress, DNS, and NetworkPolicies into one realistic, layered three-tier application with proper external exposure and internal least-privilege access control — exercising nearly every concept from this volume. You are now ready for Volume 4, which covers persistent storage and configuration management (ConfigMaps, Secrets, Volumes, PersistentVolumes, and StorageClasses).

---

# Volume 3 — End of Volume Materials

## Volume Review

Volume 3 built the complete networking layer connecting the workloads from Volume 2:

- **Lab 1:** Services and ClusterIP — the stable internal networking abstraction in front of ephemeral Pod IPs.
- **Lab 2–3:** NodePort and LoadBalancer — mechanisms for exposing Services outside the cluster.
- **Lab 4–5:** Ingress and Ingress Controllers — HTTP/HTTPS-aware routing to consolidate many services behind one entry point.
- **Lab 6–7:** DNS fundamentals and CoreDNS internals — how every component finds every other component by stable name.
- **Lab 8:** Network Policies — allow-list-based traffic control and the default-deny security pattern.
- **Lab 9:** Service discovery patterns — DNS-based discovery as the standard, plus ExternalName for outside dependencies.
- **Lab 10:** A full three-tier mini project combining everything into one realistic, secured system.

You should now be able to design, deploy, expose, and secure the network connectivity for any multi-tier application on Kubernetes, and debug connectivity issues systematically layer by layer. Volume 4 covers storage and configuration — ConfigMaps, Secrets, Volumes, PersistentVolumes, and StorageClasses — completing the picture of how applications get their data and configuration.

## Practice Exam (25 Questions)

1. Why do Services exist? — To provide a stable IP/DNS name in front of ephemeral Pod IPs.
2. What is the default Service type? — ClusterIP.
3. Is ClusterIP reachable from outside the cluster? — No.
4. What object shows the actual Pod IPs behind a Service? — Endpoints/EndpointSlice.
5. What does NodePort add on top of ClusterIP? — A fixed port opened on every node.
6. What is the default NodePort range? — 30000–32767.
7. What does a LoadBalancer Service provision? — A real, external cloud load balancer with a public IP.
8. Why does LoadBalancer's EXTERNAL-IP stay pending on plain Kind/Minikube? — No cloud provider/load-balancer implementation is available.
9. What problem does Ingress solve? — Routing HTTP/HTTPS traffic to many Services through one entry point.
10. Does an Ingress resource work without a controller? — No.
11. What does IngressClass represent? — A registered, available Ingress Controller.
12. What is the standard Service DNS format? — `<service>.<namespace>.svc.cluster.local`.
13. What does resolving a headless Service return? — Individual Pod IPs, not a single ClusterIP.
14. What implements cluster DNS? — CoreDNS.
15. Where is CoreDNS's config stored? — A ConfigMap containing a Corefile.
16. What's a reliable cluster-wide DNS health check? — Resolving `kubernetes.default`.
17. What is the default Pod-to-Pod networking behavior without NetworkPolicies? — Fully open.
18. Are NetworkPolicies allow-list or deny-list based? — Allow-list only.
19. What is the "default deny" pattern? — An empty-selector, no-rules policy plus specific allow policies.
20. Do all CNI plugins enforce NetworkPolicies? — No.
21. What's the preferred service discovery mechanism? — DNS-based discovery.
22. What's the key limitation of environment-variable-based discovery? — Only works for Services existing before the Pod started.
23. What does an ExternalName Service do? — Creates a DNS CNAME to an external hostname.
24. In the Volume 3 mini project, why can't frontend reach db directly? — No NetworkPolicy rule permits it.
25. What's the correct debugging order for connectivity issues? — DNS, then Endpoints, then NetworkPolicy, then Ingress.

## Practical Assessment

Without referring back to the labs:
1. Deploy a Deployment and expose it via a ClusterIP Service.
2. Confirm the Service's Endpoints match the Deployment's Ready Pods.
3. Create a NodePort Service for the same Deployment and access it externally.
4. Install an Ingress Controller and create an Ingress routing to the ClusterIP Service by hostname.
5. Resolve the Service by short name and fully qualified name from a debug Pod.
6. Apply a default-deny NetworkPolicy, then a specific allow rule, and confirm both blocked and allowed paths behave as expected.
7. Create an ExternalName Service pointing at a real public hostname and confirm it resolves.
8. Clean up every resource created in this assessment.

**Self-grading:** You should complete every step from memory, correctly explaining each choice, in under 35 minutes.

## Mini Project Reference
See Lab 10 for the full three-tier (frontend/backend/db) mini project with Ingress and NetworkPolicies. Use it as your reference implementation for the Practical Assessment above.

## Troubleshooting Exercises

1. **Scenario:** A Service has an empty Endpoints list. **Diagnosis:** Check selector match and Pod readiness (Lab 1).
2. **Scenario:** An Ingress shows no ADDRESS. **Diagnosis:** Check Ingress Controller installation and `ingressClassName` (Lab 5).
3. **Scenario:** Every DNS lookup fails cluster-wide. **Diagnosis:** Check CoreDNS Pod health and logs (Lab 7).
4. **Scenario:** A NetworkPolicy seems to have no effect. **Diagnosis:** Confirm the CNI plugin enforces NetworkPolicies (Lab 8).
5. **Scenario:** An application breaks entirely after adding a restrictive egress policy. **Diagnosis:** Check whether DNS (port 53 to kube-system) is still explicitly allowed (Lab 8).

## Volume 3 Cheat Sheet

| Task | Command |
|---|---|
| Expose a Deployment (ClusterIP) | `kubectl expose deployment <name> --port=<p> --target-port=<tp>` |
| Expose as NodePort | `kubectl expose deployment <name> --type=NodePort --port=<p>` |
| Expose as LoadBalancer | `kubectl expose deployment <name> --type=LoadBalancer --port=<p>` |
| Show Service backing Pods | `kubectl get endpoints <svc-name>` |
| List Ingress resources | `kubectl get ingress` |
| List IngressClasses | `kubectl get ingressclass` |
| Test DNS resolution | `kubectl run debug --image=busybox --rm -it --restart=Never -- nslookup <name>` |
| View resolv.conf | `... -- cat /etc/resolv.conf` |
| List NetworkPolicies | `kubectl get networkpolicy` |
| Minikube LoadBalancer tunnel | `minikube tunnel` |
| Minikube NodePort URL | `minikube service <name> --url` |

## kubectl Command Reference (Volume 3 subset)

`expose`, `get svc`, `get endpoints`, `get ingress`, `get ingressclass`, `get networkpolicy`, `describe ingress`, `describe svc`, plus `run --labels=` for policy-testing debug Pods.

## YAML Reference (Volume 3 subset)

| Object | apiVersion | Key spec fields |
|---|---|---|
| Service | v1 | type, selector, ports (port/targetPort/nodePort) |
| Ingress | networking.k8s.io/v1 | ingressClassName, rules (host/path/backend), tls |
| IngressClass | networking.k8s.io/v1 | controller |
| NetworkPolicy | networking.k8s.io/v1 | podSelector, policyTypes, ingress, egress |

---

*End of Volume 3 — Networking & Services. Continue to Volume 4: Storage & Configuration.*
