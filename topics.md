Here's the mapping. I'll go through your list in the order you gave it, and clearly flag anything **not covered** in the 5 volumes so you know the real gaps.

## Core Topics

| Topic | Volume(s) | Notes |
|---|---|---|
| Kubernetes Architecture | **Vol 1** (Lab 2) | |
| Control Plane | **Vol 1** (Lab 2) | Component-level only, not install |
| Worker Nodes | **Vol 1** (Lab 2), **Vol 5** (Lab 3) | |
| Pods | **Vol 1** (Lab 6–7), **Vol 2** (Lab 1) | |
| ReplicaSets | **Vol 2** (Lab 3) | |
| Deployments | **Vol 2** (Lab 4–6) | |
| Services | **Vol 3** (Lab 1–3) | |
| Ingress | **Vol 3** (Lab 4–5) | |
| ConfigMaps | **Vol 4** (Lab 1) | |
| Secrets | **Vol 4** (Lab 2) | |
| Persistent Volumes (PV) | **Vol 4** (Lab 4) | |
| Persistent Volume Claims (PVC) | **Vol 4** (Lab 5) | |
| StatefulSets | **Vol 2** (Lab 8) | |
| DaemonSets | **Vol 2** (Lab 7) | |
| Jobs & CronJobs | **Vol 2** (Lab 9–10) | |
| Namespaces | **Vol 1** (Lab 9) | |
| Resource Limits | **Vol 4** (Lab 10) | QoS, requests/limits |
| Scheduling | **Vol 5** (Lab 4–6) | |
| Helm | **Not covered** | No Helm content anywhere |
| Networking | **Vol 3** (entire volume) | |
| Storage | **Vol 4** (entire volume) | |
| RBAC | **Vol 5** (Lab 1) | |
| Monitoring | **Not covered** | No Prometheus/Grafana lab |
| Logging | **Partially** — `kubectl logs` only (**Vol 1**, Lab 6) | No Loki/ELK |

## Typical Activities

| Activity | Volume(s) |
|---|---|
| Deploy applications | **Vol 1–2** |
| Scale applications | **Vol 2** (Lab 3–4), **Vol 3** (Lab 10) |
| Rollback deployments | **Vol 2** (Lab 6) |
| Use YAML manifests | All 5 volumes |
| Learn kubectl commands | All 5 volumes (each has a cheat sheet + command reference) |

## Cluster Installation

| Topic | Volume(s) | Notes |
|---|---|---|
| kubeadm | **Vol 5** (Lab 7) | Referenced/conceptual only — no full walkthrough |
| Control Plane setup | **Not covered** | Labs use Kind/Minikube (**Vol 1**, Lab 3–4), not `kubeadm init` |
| Worker Node joining | **Not covered** | No `kubeadm join` walkthrough |
| Upgrading clusters | **Vol 5** (Lab 7) | |

## Cluster Maintenance

| Topic | Volume(s) | Notes |
|---|---|---|
| Backup etcd | **Vol 5** (Lab 8) | |
| Restore etcd | **Vol 5** (Lab 8) | |
| Certificate management | **Not covered** | |
| Cluster upgrades | **Vol 5** (Lab 7) | |

## Networking

| Topic | Volume(s) |
|---|---|
| CNI | **Vol 1** (Lab 2–3, mention), **Vol 3** (Lab 8) |
| Network Policies | **Vol 3** (Lab 8) |
| DNS | **Vol 3** (Lab 6–7) |
| Ingress | **Vol 3** (Lab 4–5) |

## Scheduling

| Topic | Volume(s) | Notes |
|---|---|---|
| Taints | **Vol 5** (Lab 5) | |
| Tolerations | **Vol 5** (Lab 5) | |
| Node Affinity | **Vol 5** (Lab 6) | |
| Pod Affinity | **Vol 5** (Lab 6) | |
| Static Pods | **Not covered** | |

## Security

| Topic | Volume(s) | Notes |
|---|---|---|
| RBAC | **Vol 5** (Lab 1) | |
| Service Accounts | **Vol 5** (Lab 2) | |
| Secrets | **Vol 4** (Lab 2) | |
| Security Contexts | **Not covered** | |
| Pod Security Standards | **Not covered** | |

## Troubleshooting

| Topic | Volume(s) | Notes |
|---|---|---|
| Pods | **Vol 1** (Lab 6–7), **Vol 5** (Lab 9) | |
| Nodes | **Vol 5** (Lab 3, Lab 9) | |
| Services | **Vol 3** (Lab 1) | |
| DNS | **Vol 3** (Lab 6–7) | |
| Control Plane | **Vol 5** (Lab 9) | General only |
| kubelet | **Not directly covered** | Mentioned architecturally, never troubleshot on its own |
| etcd | **Vol 5** (Lab 8) | |

## Storage

| Topic | Volume(s) |
|---|---|
| PV | **Vol 4** (Lab 4) |
| PVC | **Vol 4** (Lab 5) |
| Storage Classes | **Vol 4** (Lab 6) |

## CSI / Advanced Storage

| Topic | Volume(s) | Notes |
|---|---|---|
| EBS CSI Driver | **Not covered specifically** | Vol 4 Lab 7 covers CSI generically, names it only as an example |
| EFS CSI Driver | **Not covered** | |
| Dynamic provisioning | **Vol 4** (Lab 8) | |
| Static provisioning | **Vol 4** (Lab 4–5) | |
| Volume expansion | **Vol 4** (Lab 8) | |
| StatefulSets with persistent storage | **Vol 2** (Lab 8) + **Vol 4** (Lab 8) | |

## Security (extended)

| Topic | Volume(s) | Notes |
|---|---|---|
| RBAC / ClusterRole / RoleBinding | **Vol 5** (Lab 1) | |
| ServiceAccounts | **Vol 5** (Lab 2) | |
| SecurityContext | **Not covered** | |
| Pod Security Standards | **Not covered** | |
| Secrets | **Vol 4** (Lab 2) | |
| IRSA (EKS) | **Not covered** | |

## Troubleshooting Scenarios

| Scenario | Volume(s) |
|---|---|
| CrashLoopBackOff | **Vol 1** (Lab 6–7) |
| ImagePullBackOff | **Vol 1** (Lab 6) |
| Pending Pods | **Vol 1** (Lab 7), **Vol 5** (Lab 4) |
| Failed scheduling | **Vol 5** (Lab 4, 9) |
| DNS failures | **Vol 3** (Lab 6–7) |
| Volume mount issues | **Vol 4** (Lab 4–5, troubleshooting sections) |
| OOMKilled | **Vol 4** (Lab 10) |
| Failed probes | **Vol 1** (Lab 6), **Vol 2** (Lab 4–5) |
| Node NotReady | **Vol 5** (Lab 3, 9) |
| API server connectivity | **Partially** — Vol 1 (Lab 5), Vol 5 (Lab 9), general only |

## Monitoring

| Topic | Volume(s) | Notes |
|---|---|---|
| Prometheus | **Not covered** | |
| Grafana | **Not covered** | |
| Metrics Server | **Partially** — Vol 1 (Lab 4, install), Vol 5 (Lab 9, `kubectl top`) | |
| Loki / ELK | **Not covered** | |

## Helm — **Entirely not covered** (chart structure, templates, values, helpers, dependencies, releases, upgrades/rollbacks)

## Deployment Strategies & Autoscaling

| Topic | Volume(s) | Notes |
|---|---|---|
| Rolling updates | **Vol 2** (Lab 5) | |
| Rollbacks | **Vol 2** (Lab 6) | |
| HPA | **Not covered** | |
| Cluster Autoscaler / Karpenter | **Not covered** | |
| Pod Disruption Budgets | **Mentioned only** — Vol 5 Lab 3 (drain respects PDB), no dedicated lab | |
| Taints/tolerations | **Vol 5** (Lab 5) | |
| Node affinity/anti-affinity | **Vol 5** (Lab 6) | |

## GitOps — **Entirely not covered** (Argo CD, Flux)

---

### Summary of real gaps across your list
The 5 volumes solidly cover: **architecture, workloads (Pods/ReplicaSets/Deployments/DaemonSets/StatefulSets/Jobs/CronJobs), networking (Services/Ingress/DNS/NetworkPolicy), storage (PV/PVC/StorageClass/dynamic provisioning), configuration (ConfigMaps/Secrets/resource limits), and administration (RBAC/ServiceAccounts/scheduling/taints/affinity/upgrades/etcd backup/troubleshooting)**.

**Missing entirely:** Helm, GitOps (Argo CD/Flux), Monitoring stack (Prometheus/Grafana/Loki/ELK), HPA, Cluster Autoscaler/Karpenter, SecurityContext/Pod Security Standards, cloud-specific CSI drivers (EBS/EFS) and IRSA, real `kubeadm` install/join walkthrough, certificate management, and static Pods.

Want me to scope a **Volume 6 (or a standalone lab set)** covering these gaps — Helm, monitoring/logging, autoscaling, security contexts, and GitOps?
