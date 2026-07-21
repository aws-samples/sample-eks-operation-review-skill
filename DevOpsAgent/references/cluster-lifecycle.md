# Cluster Lifecycle & Upgrades

## Purpose
Assess EKS cluster version currency, data plane alignment, upgrade readiness, add-on compatibility, and upgrade process maturity.

## EKS Version Support Status

> **Last verified:** 2026-07-20. Support status is determined primarily from the **live EKS API**; the dated table below is a fallback used only when the API is unavailable.

Determine support status from the live API first; fall back to the dated table only when the API cannot be reached. Do NOT guess or use training data.

**Primary (live) method — preferred:**
Call the EKS **DescribeClusterVersions** API to get the authoritative real-time list of EKS versions and each version's `status` (`STANDARD_SUPPORT` / `EXTENDED_SUPPORT` / `UNSUPPORTED`). Define **latest** = the highest version whose status is `STANDARD_SUPPORT` in the API response.

**Fallback method — only when the live API is unavailable:**
Use the dated table below. In fallback mode, **latest** = the highest `STANDARD_SUPPORT` version *in this table* (not the true current latest), so ratings are relative to the table and may lag reality — note fallback mode in the finding.

> This fallback table was sourced from the EKS upgrade skill's version data on the "as of" date shown in the table header; it goes stale as new EKS versions ship and support windows advance — refresh it periodically against the [official EKS version calendar](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html). Prefer the DescribeClusterVersions API with `includeAll` set (which also surfaces `UNSUPPORTED` versions) whenever the live API is reachable. If the cluster runs a version **not present** in this fallback table and the API is unavailable, do not guess — rate ⬜ UNKNOWN and note that the version is outside the fallback table.

| Version | Standard Support Until | Extended Support Until | Status (as of 2026-07-20) |
|---------|----------------------|----------------------|----------------|
| 1.36 | August 2, 2027 | August 2, 2028 | ✅ STANDARD_SUPPORT (latest in table) |
| 1.35 | March 27, 2027 | March 27, 2028 | ✅ STANDARD_SUPPORT |
| 1.34 | December 2, 2026 | December 2, 2027 | ✅ STANDARD_SUPPORT |
| 1.33 | July 29, 2026 | July 29, 2027 | ✅ STANDARD_SUPPORT |
| 1.32 | March 23, 2026 | March 23, 2027 | ⚠️ EXTENDED_SUPPORT (standard ended) |
| 1.31 | November 26, 2025 | November 26, 2026 | ⚠️ EXTENDED_SUPPORT (standard ended) |
| 1.30 | July 23, 2025 | July 23, 2026 | ⚠️ EXTENDED_SUPPORT (standard ended) |

**CRITICAL:** The `upgradePolicy.supportType` field from the API is a CONFIGURATION PREFERENCE, not the current billing status. A cluster on a standard-support version with `supportType: EXTENDED` is still on standard support and NOT paying the extended premium. Always determine actual support status from the DescribeClusterVersions API (or the fallback table), never from `supportType`.

## Checks to Execute

### 1.1 — Control Plane Version Currency

**What to check:**
- The cluster's current Kubernetes version and platform version
- The version's actual support status (STANDARD_SUPPORT / EXTENDED_SUPPORT / UNSUPPORTED)
- Report the actual support status, NOT the `supportType` API field

**How to check:**
1. Describe the cluster → get `version` and `platformVersion`
2. Call the EKS **DescribeClusterVersions** API → find the cluster's version and its `status`, and identify **latest** = the highest `STANDARD_SUPPORT` version. If the live API is unavailable, fall back to the dated table above and note fallback mode in the finding.
3. Report: version, standard/extended/unsupported status, and when the current support period ends

**Rating (evaluate RED first, then AMBER, then GREEN):**
- 🔴 RED: Version is on **extended support**, or **unsupported** (past its extended-support end date) — extended support incurs the cost premium below; unsupported means running with no support
- 🟡 AMBER: Version is on **standard support but older** — a standard version more than one minor behind the latest standard version
- 🟢 GREEN: Version is the **latest** standard-support version or **N-1** (one minor behind the latest standard version)
- ⬜ UNKNOWN: Cannot determine the version (should not happen with live access)

**Version out-of-range guard (fallback mode only):** If using the fallback table and the cluster version is higher than the highest version in the table, rate GREEN and note: "Version 1.X is newer than the fallback table (last verified 2026-07-20); the live API was unavailable. Rated GREEN as latest. Refresh the table when convenient."

**Extended-support cost impact:** Extended support has historically cost ~$0.60/hr vs ~$0.10/hr for standard support. These rates are indicative and subject to change — verify against the current [Amazon EKS pricing page](https://aws.amazon.com/eks/pricing/) before quoting figures. Compute, do not estimate:
```
extra_cost_per_month = (extended_rate - standard_rate) × 730
total_extended_cost  = extended_rate × 730
# With the indicative rates above: extra = (0.60 - 0.10) × 730 = ~$365/month per cluster
# 730 = average hours per month (365 days × 24 hours ÷ 12 months)
```

**Key talking point:** At indicative rates, extended support adds roughly $365/month per cluster — verify against the EKS pricing page before quoting a figure.

---

### 1.2 — Data Plane Version Alignment

**What to check:**
- List all node groups and their Kubernetes versions
- Compare each node group version against the control plane version
- Check AMI type (AL2 vs AL2023 vs Bottlerocket)
- Check for Karpenter NodePools or EKS Auto Mode

**How to check:**
1. List node groups → describe each for version, AMI type, capacity type
2. List nodes via Kubernetes API → get kubelet versions
3. Check for Karpenter NodePools (`nodepools.karpenter.sh`). If 404/NotFound (CRD not installed) → Karpenter is not deployed, rate based on other node management. If 403/Forbidden → mark Karpenter status UNKNOWN.
4. Describe cluster → check `computeConfig` for Auto Mode

**Rating:**
- 🟢 GREEN: All nodes within N-1 of control plane, using managed node groups/Karpenter/Auto Mode
- 🟡 AMBER: Nodes within one minor of the control plane but mixed versions, or self-managed nodes
- 🔴 RED: Any node more than 2 minors behind the control plane, or no visibility into node versions (this N-2 bound is a skill-defined operational standard — stricter than the upstream kubelet version-skew policy, which permits up to N-3)
- ⬜ UNKNOWN: No nodes found (possible if cluster is new or uses Fargate only)

**Red flags:** AL2 OS is past EOL (2026-06-30) and EKS AL2 AMIs ended with Kubernetes 1.32 — migrate to AL2023 or Bottlerocket; self-managed nodes with no automated upgrade path.

---

### 1.3 — Upgrade Readiness & Deprecated API Detection

**What to check:**
- EKS Cluster Insights for upgrade blockers
- Presence of deprecated API usage
- PodSecurityPolicy resources (removed in K8s 1.25)

**How to check:**
1. Get EKS Insights → filter for UPGRADE_READINESS category
2. List any PodSecurityPolicy resources via Kubernetes API
3. Check for Helm releases in kube-system (may use deprecated APIs)

**Rating:**
- 🟢 GREEN: No critical insights, no deprecated API usage detected
- 🟡 AMBER: WARNING-level insights, or no automated detection tooling
- 🔴 RED: ERROR insights, deprecated APIs actively in use
- ⬜ UNKNOWN: Insights API not accessible
- **Scoring authority:** this check owns the EKS Cluster Insights / upgrade-readiness signal; check 10.3 defers here and is evidence-only.

---

### 1.4 — Add-on Version Compatibility

**What to check:**
- List all EKS managed add-ons with versions and health
- Compare installed versions against latest compatible for the cluster version
- Check for self-managed add-ons in kube-system (Helm releases)

**How to check:**
1. List addons → describe each for version, status, health
2. For each core add-on (vpc-cni, coredns, kube-proxy, aws-ebs-csi-driver), compare installed vs latest compatible

**Rating:**
- 🟢 GREEN: All core add-ons are EKS Managed and on latest or N-1 compatible version
- 🟡 AMBER: Managed but behind, or mix of managed and self-managed
- 🔴 RED: Core add-ons self-managed with no version tracking, or health issues present
- ⬜ UNKNOWN: Cannot list add-ons
- **Scoring authority:** this check owns add-on version compatibility scoring; check 10.1 defers here and is evidence-only.

**Key talking point:** EKS does NOT auto-update add-ons when you upgrade the control plane. This is the #1 thing customers forget.

---

### 1.5 — Upgrade Process Maturity

**What to check (target cluster only):**
- Cluster tags for environment classification (dev, staging, prod)
- Evidence of IaC-managed upgrades (eksctl, CloudFormation, Terraform tags)

**How to check:**
1. Describe the target cluster → check tags for environment indicators

**Do NOT** list or describe other clusters in the account. Stay within the scope of the target cluster.

**Rating:**
- 🟢 GREEN: Cluster has environment tags, evidence of IaC management
- 🟡 AMBER: No environment tags, or unclear upgrade process
- 🔴 RED: No evidence of structured upgrade process
- ⬜ UNKNOWN: Cannot determine upgrade history from API alone — suggest user investigate

**Investigate manually:** Do you have a documented upgrade runbook? Do you test upgrades on a non-prod cluster first? Can more than one person execute an upgrade?
