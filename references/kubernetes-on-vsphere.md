# Kubernetes on vSphere

## The Landscape

There are multiple ways to run Kubernetes on vSphere. The choice depends on how much VMware integration you want versus how much operational control you need.

| Approach                       | VMware Integration                  | Operational Burden                     | Use Case                                                             |
| ------------------------------ | ----------------------------------- | -------------------------------------- | -------------------------------------------------------------------- |
| Tanzu with Supervisor          | Deep (native vSphere object)        | Low (VMware manages k8s control plane) | VCF environments, vSphere-centric shops                              |
| Tanzu Kubernetes Grid (TKG)    | Moderate (Cluster API, runs in VMs) | Moderate                               | Multi-cloud consistency, standalone vSphere                          |
| Vanilla Kubernetes             | None (just VMs)                     | High (you manage everything)           | Maximum flexibility, existing k8s expertise                          |
| Rancher / OpenShift on vSphere | Minimal (VM provisioning only)      | Moderate                               | Multi-cluster management, enterprise features from non-VMware vendor |

## Tanzu with Supervisor (vSphere with Tanzu)

### Architecture

The Supervisor is a special Kubernetes control plane embedded directly into vSphere. It turns a vSphere cluster into a Kubernetes-aware platform.

```
vCenter
  |
  +-- vSphere Cluster (Supervisor enabled)
        |
        +-- Supervisor Control Plane (3 VMs, auto-managed)
        |     |
        |     +-- Kubernetes API server
        |     +-- vSphere Pod Service
        |     +-- TKG Service (provisions workload clusters)
        |
        +-- vSphere Namespaces (tenancy boundary)
        |     |
        |     +-- Namespace: team-alpha
        |     |     +-- Resource limits (CPU, memory, storage)
        |     |     +-- Permissions (mapped to vCenter SSO/AD)
        |     |     +-- Storage policies (vSAN, NFS)
        |     |     +-- Network policies (NSX or VDS)
        |     |
        |     +-- Namespace: team-beta
        |           +-- ...
        |
        +-- Workload Clusters (TKG clusters inside namespaces)
              |
              +-- team-alpha/prod-cluster (3 control, 5 worker nodes)
              +-- team-alpha/dev-cluster  (1 control, 2 worker nodes)
              +-- team-beta/app-cluster   (3 control, 3 worker nodes)
```

### Key Concepts

**Supervisor Cluster**: Not a cluster you deploy workloads to directly (in most cases). It's the management plane that provisions and manages workload clusters.

**vSphere Namespace**: The tenancy boundary. Maps to a Kubernetes namespace on the Supervisor. Controls:

- Who can access it (vCenter SSO users/groups)
- How much compute they can consume (resource quotas)
- Which storage policies are available
- Which VM classes (T-shirt sizes) are available for workload cluster nodes

**Workload Cluster**: A full, standalone Kubernetes cluster running as VMs inside a vSphere Namespace. This is where applications actually run. Provisioned via the TKG Service using a declarative YAML spec:

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: prod-cluster
  namespace: team-alpha
spec:
  topology:
    class: tanzukubernetescluster
    version: v1.27.5+vmware.1
    controlPlane:
      replicas: 3
    workers:
      machineDeployments:
        - class: node-pool
          name: worker
          replicas: 5
          variables:
            overrides:
              - name: vmClass
                value: best-effort-large
              - name: storageClass
                value: vsan-default-storage-policy
```

**VM Classes**: T-shirt sizes for workload cluster nodes. Defined at the vSphere level, made available per-namespace.

| Class              | vCPU | Memory | Use Case                      |
| ------------------ | ---- | ------ | ----------------------------- |
| best-effort-small  | 2    | 4GB    | Dev, testing                  |
| best-effort-medium | 4    | 8GB    | General workloads             |
| best-effort-large  | 8    | 16GB   | Production workers            |
| guaranteed-xlarge  | 16   | 32GB   | Databases, stateful workloads |

"Best-effort" means no vSphere reservations. "Guaranteed" sets CPU/memory reservations equal to configured values (impacts HA slot sizing -- see [vm-reservations.md](vm-reservations.md)).

### vSphere Pods (Optional, NSX Required)

vSphere Pods run containers directly on ESXi without a traditional VM, using a lightweight VM called a CRX (Container Runtime for ESXi). Each pod gets its own kernel -- stronger isolation than shared-kernel containers.

```
Traditional Kubernetes:
  VM → Linux Kernel → Container Runtime → Pod → Containers

vSphere Pod:
  ESXi → CRX (micro-VM per pod) → Linux Kernel → Containers
```

- **Advantage**: No VM overhead for the worker node OS. Near-native performance. Strong isolation.
- **Requirement**: NSX networking (no VDS support for vSphere Pods)
- **Limitation**: Not all Kubernetes features supported. No DaemonSets, limited volume types. Best for stateless workloads.
- **Reality check**: Most environments use workload clusters (regular VMs) rather than vSphere Pods. The VM overhead is modest and the compatibility is much broader.

## Tanzu Kubernetes Grid (TKG) Standalone

TKG without the Supervisor. Uses Cluster API to provision Kubernetes clusters as VMs on vSphere.

```
Management Cluster (small, runs Cluster API controllers)
  |
  +-- Workload Cluster: prod (VMs on vSphere)
  +-- Workload Cluster: staging (VMs on vSphere)
  +-- Workload Cluster: dev (VMs on vSphere)
```

**When to use over Supervisor:**

- vSphere without VCF licensing
- Multi-cloud strategy (TKG also deploys to AWS, Azure)
- Don't want or can't enable Supervisor on the cluster

**Trade-off**: Less vSphere integration. No vSphere Namespaces, no VM Classes via vCenter, no vSphere Pod support. You manage the management cluster yourself.

## Vanilla Kubernetes on vSphere VMs

Just run kubeadm, kubespray, or k3s inside regular VMs. No VMware-specific tooling.

**What you lose:**

- Automated node provisioning (build VMs manually or with Terraform)
- vSphere storage integration (need to install CSI driver yourself)
- Lifecycle management (upgrades are your problem)
- VM class governance through vCenter

**What you gain:**

- No VMware-specific tooling lock-in
- Exact Kubernetes version and configuration control
- Works with any vSphere license

**When it makes sense:** Teams with deep Kubernetes expertise who want full control, or environments where Tanzu licensing isn't justified.

## Storage Integration

### vSphere CSI Driver

The Container Storage Interface driver for vSphere. Provisions persistent volumes backed by vSphere datastores (vSAN, VMFS, NFS).

```
Pod → PVC → StorageClass → vSphere CSI → vSphere API → VMDK on datastore
```

**StorageClass examples:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: vsan-fast
provisioner: csi.vsphere.vmware.com
parameters:
  storagepolicyname: "vSAN Default Storage Policy"
  datastoreurl: "ds:///vmfs/volumes/vsan:xxxx/"
```

**What the CSI driver handles:**

- Dynamic PV provisioning (create VMDKs on demand)
- Volume expansion (grow PVs without pod restart)
- Topology-aware provisioning (place volumes on correct datastore for the node's host)
- Snapshot and restore (CSI snapshots backed by vSphere snapshots)

**What it doesn't handle:**

- ReadWriteMany (RWX) volumes -- vSphere CSI provides ReadWriteOnce (RWO) only. For RWX, use NFS or a distributed storage solution (Rook-Ceph, Longhorn, etc.)

### vSAN and Kubernetes Storage Policies

vSAN storage policies map directly to Kubernetes StorageClasses:

| vSAN Policy Setting        | What It Controls         | Kubernetes Impact |
| -------------------------- | ------------------------ | ----------------- |
| Failures to Tolerate (FTT) | Data copies across hosts | Durability of PVs |
| Stripe width               | Disk striping            | IOPS for PVs      |
| Deduplication/compression  | Space efficiency         | Capacity usage    |
| Encryption                 | Data-at-rest encryption  | Compliance        |

Create multiple StorageClasses for different tiers:

```
StorageClass: vsan-platinum  → FTT=2, stripe=2  (mission-critical databases)
StorageClass: vsan-gold      → FTT=1, stripe=1  (production stateful apps)
StorageClass: vsan-silver    → FTT=1, dedup+comp (general purpose, space efficient)
```

## Networking

### NSX-Based Networking (Supervisor with NSX)

NSX provides the full networking stack for Supervisor and workload clusters:

- **Pod networking**: Each pod gets an IP from NSX-managed segments. No CNI plugin needed -- NSX is the CNI.
- **Network policies**: Kubernetes NetworkPolicy objects enforced by NSX DFW rules
- **Load balancing**: NSX Advanced Load Balancer (Avi) or NSX-T native load balancer for Services of type LoadBalancer
- **Ingress**: NSX provides Ingress controller integration

### VDS-Based Networking (Supervisor without NSX)

Supervisor also works with standard vSphere Distributed Switch networking (HAProxy or NSX ALB for load balancing). Simpler but less capable:

- No vSphere Pods (VMs only)
- No NSX-backed network policies (use Calico or Antrea instead)
- Still functional for most workload cluster use cases

### Antrea CNI

Antrea is the default CNI for TKG workload clusters. It's an Open vSwitch-based CNI that provides:

- Pod networking
- Kubernetes NetworkPolicy enforcement
- ClusterNetworkPolicy (cluster-wide policies, Antrea extension)
- Egress controls
- Flow visibility and tracing

## Lifecycle Management

### Workload Cluster Upgrades

With TKG Service (Supervisor), upgrades are declarative:

1. Update the Kubernetes version in the Cluster spec
2. TKG rolls out new control plane nodes, then workers
3. Rolling update -- one node at a time, draining and cordoning

```yaml
spec:
  topology:
    version: v1.28.4+vmware.1 # changed from v1.27.5
```

The Supervisor manages the rolling update. Worker nodes are replaced (new VM with new version, old VM deleted), not upgraded in place.

### Supervisor Upgrades

Supervisor lifecycle is tied to vCenter:

```
vCenter upgrade → Supervisor compatibility matrix → Supervisor upgrade → TKG version availability
```

You can't run arbitrary Kubernetes versions. Available versions are those VMware has validated and bundled with the Supervisor release.

## Design Decisions

### How Many Workload Clusters?

| Strategy                           | When to Use                               | Trade-off                                                     |
| ---------------------------------- | ----------------------------------------- | ------------------------------------------------------------- |
| One big cluster                    | Small team, simple workloads              | Blast radius -- one misconfiguration affects everything       |
| Per-environment (dev/staging/prod) | Most common starting point                | Balances isolation with management overhead                   |
| Per-team                           | Large org, strong autonomy needs          | More clusters to manage, but teams are independent            |
| Per-application                    | Strict isolation requirements, compliance | High management overhead, justified only for regulatory needs |

### Worker Node Sizing

Fewer large nodes vs many small nodes:

| Approach                             | Advantage                                      | Disadvantage                                                           |
| ------------------------------------ | ---------------------------------------------- | ---------------------------------------------------------------------- |
| Fewer large nodes (16+ vCPU, 64+ GB) | Better bin-packing, less overhead per node     | Node failure has bigger blast radius, waste if not filled              |
| Many small nodes (4 vCPU, 16GB)      | Node failure affects fewer pods, easy to scale | More overhead (kubelet, kube-proxy per node), scheduling fragmentation |

**Practical guidance:**

- Start with medium nodes (8 vCPU, 32GB) and adjust based on workload patterns
- Control plane nodes: 4 vCPU, 16GB is sufficient for most clusters under 100 worker nodes
- Always run 3 control plane nodes for HA (etcd quorum requires odd number)

### Resource Quotas and Limit Ranges

Always set resource quotas on vSphere Namespaces and Kubernetes namespaces. Without them, one team can starve others:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "40"
    requests.memory: 80Gi
    limits.cpu: "80"
    limits.memory: 160Gi
    persistentvolumeclaims: "20"
```

Set LimitRanges to enforce per-pod defaults and maximums -- prevents a single pod from claiming all namespace resources.

## Common Mistakes

| Mistake                                             | Consequence                                              | Fix                                                               |
| --------------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------- |
| Running production workloads on Supervisor directly | Supervisor is management plane, not application platform | Deploy workload clusters for applications                         |
| No resource quotas on namespaces                    | One team consumes all cluster resources                  | Set quotas at both vSphere Namespace and k8s namespace level      |
| Single control plane node                           | etcd loss = cluster down, unrecoverable                  | Always 3 control plane nodes in production                        |
| Ignoring vSphere HA interaction                     | Worker node VMs not restarted after host failure         | Ensure cluster is HA-enabled, VM restart priority set             |
| Using guaranteed VM classes everywhere              | Memory reservations inflate HA slot sizes                | Use best-effort for most workloads, guaranteed only for databases |
| No persistent storage planning                      | Stateful apps fail on pod reschedule                     | Configure vSphere CSI, create appropriate StorageClasses          |
| Skipping network policy                             | All pods can talk to all pods                            | Deploy Antrea/Calico policies, default deny, allow explicitly     |
