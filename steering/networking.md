# Networking

## Purpose
Assess VPC CNI configuration, IP capacity, DNS health, and network segmentation.

## Checks to Execute

### 6.1 — VPC and Subnet IP Capacity

**What to check:**
- Subnets used by the cluster and available IP count
- VPC CNI configuration: prefix delegation, custom networking, WARM_IP_TARGET
- Current pod count vs IP capacity
- VPC CNI add-on version

**How to check:**
1. Describe cluster → get subnet IDs from `resourcesVpcConfig.subnetIds`
2. Get VPC config for the cluster (available IPs per subnet)
3. List pods (Running) → count total
4. List nodes → count total
5. Describe addon `vpc-cni` → check version and configuration
6. Check DaemonSet `aws-node` in kube-system → inspect env vars for `ENABLE_PREFIX_DELEGATION`, `AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG`
7. List ENIConfig resources (custom networking indicator)

**Rating:**
- 🟢 GREEN: >30% IP headroom (skill-defined heuristic — not an AWS-published threshold), prefix delegation or custom networking enabled
- 🟡 AMBER: Adequate IPs now but no prefix delegation and cluster is growing
- 🔴 RED: <15% IPs available (skill-defined heuristic — not an AWS-published threshold), or past IP exhaustion incidents
- ⬜ UNKNOWN: Cannot determine subnet sharing with other workloads
- **Evaluation order:** assess RED first; if not RED, assess AMBER; otherwise GREEN. Keeps the bands exhaustive and non-overlapping.

**Key talking point:** Prefix delegation assigns a /28 (16 IPs) per ENI slot instead of 1 IP — dramatically increases pod density.

---

### 6.2 — CoreDNS Health and Scaling

**What to check:**
- CoreDNS deployment: replica count, resource requests, pod placement
- Node count (to assess CoreDNS ratio — ~1 replica per 16 nodes or 256 cores, minimum 2)
- NodeLocal DNSCache DaemonSet
- CoreDNS HPA
- CoreDNS topology spread constraints

**How to check:**
1. Read Deployment `coredns` in kube-system → replicas, resources, topologySpreadConstraints
2. List pods with label `k8s-app=kube-dns` → check node placement
3. Count nodes
4. List DaemonSets → check for `node-local-dns` or `nodelocaldns` (recommended for clusters with 50+ nodes)
5. Check for CoreDNS autoscaling before flagging "no HPA": the EKS-managed CoreDNS add-on has a built-in `autoScaling` feature (enabled via the add-on `configurationValues`) that scales replicas WITHOUT creating an HPA object. Also list HPAs in kube-system with label `k8s-app=kube-dns`. If either the add-on built-in autoScaling is enabled OR an HPA is present, CoreDNS can auto-scale, so fewer static replicas is acceptable.

**Rating:**
- 🟢 GREEN: CoreDNS scaled to cluster size (~1 replica per 16 nodes or 256 cores, min 2) or add-on built-in autoScaling / HPA enabled, spread across AZs, NodeLocal DNSCache on large clusters
- 🟡 AMBER: Adequate replicas but no topology spread, or no NodeLocal DNSCache on 50+ node clusters, or no autoscaling of any kind (neither the add-on built-in autoScaling nor an HPA)
- 🔴 RED: CoreDNS under-provisioned (2 replicas for 50+ nodes with no autoscaling — neither built-in autoScaling nor HPA), or past DNS incidents
- ⬜ UNKNOWN: Cannot determine if DNS issues have occurred historically
- *Note: the ~1-replica-per-16-nodes (or 256-cores) ratio is the AWS-published CoreDNS autoscaler formula, not a skill-defined value. The "50+ nodes" NodeLocal DNSCache trigger and the >30%/<15% IP-headroom bands, by contrast, are skill-defined heuristics with no AWS-published source.*
- **Evaluation order:** assess RED first; if not RED, assess AMBER; otherwise GREEN. Keeps the bands exhaustive and non-overlapping.

---

### 6.3 — Network Policies & Segmentation

**What to check:**
- VPC CNI Network Policy Controller enabled (the `aws-network-policy-agent` container runs with `--enable-network-policy=true`, or the vpc-cni add-on `configurationValues` sets `enableNetworkPolicy: "true"`)
- Calico pods (alternative enforcement engine)
- NetworkPolicy resources across namespaces
- Default-deny policies (podSelector: {})
- Namespaces without any NetworkPolicy

**How to check:**
1. Read DaemonSet `aws-node` in kube-system → inspect the `aws-network-policy-agent` container for the `--enable-network-policy=true` arg (or check the vpc-cni add-on `configurationValues` for `enableNetworkPolicy: true`). A self-managed Helm install instead sets the `amazon-vpc-cni` ConfigMap key `enable-network-policy-controller: "true"`.
2. List pods with label `k8s-app=calico-node`
3. List NetworkPolicies across all namespaces
4. Inspect NetworkPolicies for default-deny (empty podSelector)
5. Compare namespaces with policies vs namespaces without

**Rating:**
- 🟢 GREEN: Enforcement enabled (VPC CNI controller or Calico), default-deny in production namespaces
- 🟡 AMBER: Policies defined but enforcement not verified, or policies in some namespaces only
- 🔴 RED: No network policies, or policies defined but enforcement not enabled (false security)
- ⬜ UNKNOWN: Cannot determine if policies have been tested

**Critical gotcha:** VPC CNI requires explicitly enabling the Network Policy Controller. Without it, NetworkPolicy resources are just YAML that does nothing.
