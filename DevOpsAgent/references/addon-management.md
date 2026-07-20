# Add-on & Component Management

## Purpose
Assess add-on management maturity, node health monitoring, and cluster insights usage.

## Checks to Execute

### 10.1 — Core Add-ons Managed via EKS Managed Add-ons

**What to check:**
- All EKS managed add-ons: name, version, status, health
- Compare installed versions against latest compatible for the cluster version
- Self-managed add-ons in kube-system (Helm releases)
- Deprecated in-tree EBS plugin usage

**How to check:**
1. List addons → describe each for version, status, health issues
2. For each of the 4 core add-ons (vpc-cni, coredns, kube-proxy, aws-ebs-csi-driver):
   - Check if installed as managed add-on
   - Compare installed version vs latest compatible
3. List PersistentVolumes → check for `spec.awsElasticBlockStore` (deprecated in-tree)

**Rating:**
- **Evidence-only — rated under check 1.4 (Add-on Version Compatibility).** Collect add-on inventory, versions, managed-vs-self-managed status, health, and deprecated in-tree plugin usage as supporting evidence, but do not assign an independent GREEN/AMBER/RED here; the add-on-compatibility rating is owned by check 1.4.

**Key talking point:** EKS does NOT auto-update add-ons when you upgrade the control plane. Clusters upgraded to 1.31 still running vpc-cni from 1.27 is a ticking time bomb.

---

### 10.2 — Node Health Monitoring & Auto-Repair

**What to check:**
- EKS Node Monitoring Agent add-on (`eks-node-monitoring-agent`)
- Node auto-repair configuration on managed node groups
- GPU nodes (need NMA for GPU failure detection)
- Current node conditions

**How to check:**
1. Describe addon `eks-node-monitoring-agent`
2. List node groups → describe each → check `nodeRepairConfig`
3. List nodes → check for `nvidia.com/gpu` in capacity (GPU nodes)
4. List nodes → inspect conditions for MemoryPressure, DiskPressure, etc.

**Rating:**
- 🟢 GREEN: NMA installed and node auto-repair enabled
- 🟡 AMBER: No NMA but node conditions monitored, or NMA without auto-repair
- 🔴 RED: No node health monitoring beyond basic Kubernetes conditions, especially with GPU workloads
- ⬜ UNKNOWN: Should not happen with live access

---

### 10.3 — EKS Cluster Insights Reviewed

**What to check:**
- All cluster insights with status
- Count by status (PASSING, WARNING, ERROR)
- Details on any ERROR or WARNING insights

**How to check:**
1. Get EKS Insights for the cluster
2. For any non-PASSING insights → get detailed description and recommendation

**Rating:**
- **Evidence-only — rated under check 1.3 (Upgrade Readiness & Deprecated API Detection).** Collect the cluster insights and their statuses as supporting evidence, but do not assign an independent GREEN/AMBER/RED here; the upgrade-readiness / insights rating is owned by check 1.3.
