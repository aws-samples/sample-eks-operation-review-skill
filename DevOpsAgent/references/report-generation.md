# Report Generation

## Purpose
After all section checks are complete, generate the EKS Operation Review report.

## Consistency Checks (MANDATORY before writing)

Before writing the report, validate consistency:

1. **Build a master list** of all findings with their ratings from sections 01-10
2. **For each RED item:** confirm it appears in "Critical" (or "Quick Wins" if fixable in < 1 hour)
3. **For each AMBER item:** confirm it appears in "Important" (or "Quick Wins" if fixable in < 1 hour)
4. **For the Executive Summary:** only mention ratings that match the master list — do not call something a "critical gap" if it's AMBER, or omit a RED from the summary
5. **For Prioritized Actions:** every entry must reference the finding ID (e.g., "4.1 — Control Plane Logging")

## Workflow

### Step 1: Build Master Finding List

```
| Section | Item ID | Item Name | Rating |
```

> For the two evidence-only checks (10.1 and 10.3, whose ratings are owned by 1.4 and 1.3 respectively), use the literal Rating value `Evidence-only (see 1.4)` and `Evidence-only (see 1.3)` — they contribute no GREEN/AMBER/RED/N/A count to the Maturity Score.

### Step 2: Calculate Maturity Score

- Count GREEN, AMBER, RED, N/A, UNKNOWN
- Calculate percentages (exclude both N/A and UNKNOWN from denominator — N/A means the check does not apply to this cluster)

### Step 3: Write Executive Summary

From the master list, identify:
- **Top strengths** (GREEN items with highest operational impact)
- **Top gaps** (RED items, ordered by blast radius: security > availability > cost)
- Write 2-3 paragraphs. Every rating mentioned must match the master list.

### Step 4: Write Findings Tables

One table per section. Every item from the master list must appear.

### Step 5: Write Prioritized Actions

Cross-reference against the master list:
- **Critical (30 days):** All RED items except those fixable in < 1 hour, which may instead go in Quick Wins. Column: `Finding | Action | References`
- **Important (90 days):** All AMBER items except those fixable in < 1 hour, which may instead go in Quick Wins. Column: `Finding | Action | References`
- **Quick Wins:** Items (RED or AMBER) fixable in < 1 hour. Column: `Finding | Action | Effort | Impact | References`

Every entry must include the finding ID and name (e.g., "4.1 — Control Plane Logging 🔴").

**One row per finding.** Never bundle multiple findings into a single row (e.g., "2.2/2.3 — GitOps & Drift Detection"). Each finding has its own context, action, and references — collapsing them hides information and breaks the consistency rule that every RED and AMBER must appear in Prioritized Actions. If two findings genuinely share an action, list them on separate rows that point to the same action.

**Ordering within Critical:** List RED items by blast radius category:
1. Security first — public API endpoint, hardcoded credentials, no PSA/network policies, overly broad RBAC
2. Availability next — no PDBs, single-replica critical workloads, missing health probes, no alerting
3. Cost last — extended support billing, deprecated storage classes

Within each category, order by scope (cluster-wide before namespace-scoped).

### Step 6: Write Investigate Manually

All UNKNOWN items with specific questions the user should answer.

### Step 7: Apply AWS Reference Links

Use the pre-verified reference map below. These URLs are pre-verified and mapped by section — no documentation lookup is needed during report generation.

**Section 01 — Cluster Lifecycle & Upgrades**
- Version calendar: `https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html`
- Upgrade cluster: `https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html`
- Best practices for upgrades: `https://docs.aws.amazon.com/eks/latest/best-practices/cluster-upgrades.html`
- Platform versions: `https://docs.aws.amazon.com/eks/latest/userguide/platform-versions.html`
- Managed node groups: `https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html`
- EKS Auto Mode: `https://docs.aws.amazon.com/eks/latest/userguide/automode.html`

**Section 02 — Infrastructure as Code & GitOps**
- EKS User Guide (general): `https://docs.aws.amazon.com/eks/latest/userguide/`
- Best practices (general): `https://docs.aws.amazon.com/eks/latest/best-practices/`

**Section 03 — Access & Identity**
- IRSA: `https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html`
- EKS Pod Identity: `https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html`
- Access entries: `https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html`
- Grant K8s access: `https://docs.aws.amazon.com/eks/latest/userguide/grant-k8s-access.html`
- RBAC hardening: `https://docs.aws.amazon.com/eks/latest/userguide/rbac-hardening.html`
- API server endpoint: `https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html`
- Security best practices: `https://docs.aws.amazon.com/eks/latest/best-practices/security.html`
- Pod Security Standards: `https://docs.aws.amazon.com/eks/latest/best-practices/pod-security.html`

**Section 04 — Observability**
- Control plane logging: `https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html`
- Observability overview: `https://docs.aws.amazon.com/eks/latest/userguide/eks-observe.html`

**Section 05 — Workload Configuration**
- EBS CSI driver: `https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html`
- Reliability best practices: `https://docs.aws.amazon.com/eks/latest/best-practices/reliability.html`

**Section 06 — Networking**
- VPC CNI: `https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html`
- Prefix delegation: `https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html`
- Custom networking: `https://docs.aws.amazon.com/eks/latest/userguide/cni-custom-network.html`
- CoreDNS: `https://docs.aws.amazon.com/eks/latest/userguide/managing-coredns.html`
- Networking best practices: `https://docs.aws.amazon.com/eks/latest/best-practices/networking.html`

**Section 07 — Autoscaling**
- Karpenter best practices: `https://docs.aws.amazon.com/eks/latest/best-practices/karpenter.html`
- Scalability best practices: `https://docs.aws.amazon.com/eks/latest/best-practices/scalability.html`
- Cost optimization: `https://docs.aws.amazon.com/eks/latest/best-practices/cost-opt.html`

**Section 08 — Deployment Practices**
- Reliability best practices: `https://docs.aws.amazon.com/eks/latest/best-practices/reliability.html`

**Section 09 — Operational Processes**
- Reliability best practices: `https://docs.aws.amazon.com/eks/latest/best-practices/reliability.html`

**Section 10 — Add-on Management**
- Managed add-ons: `https://docs.aws.amazon.com/eks/latest/userguide/managing-add-ons.html`
- Node health & auto-repair: `https://docs.aws.amazon.com/eks/latest/userguide/node-health.html`

**Fallback (any topic):**
- EKS Best Practices Guide: `https://docs.aws.amazon.com/eks/latest/best-practices/`
- EKS User Guide: `https://docs.aws.amazon.com/eks/latest/userguide/`

Do NOT fabricate URLs beyond this list. If a finding doesn't match a specific URL above, use the fallback section-level page.

### Step 8: Final Consistency Validation

Before outputting, scan the report for:
- Any RED item missing from Prioritized Actions → add it
- Any item mentioned in Executive Summary with wrong rating → fix it
- Any Prioritized Action without a finding ID → add the ID

### Step 8b: Append Sample-Code Disclaimer

Add the following footer at the very end of the report, after the AWS Reference Links section, separated by a horizontal rule:

```markdown
---

*This report was generated by an AWS DevOps Agent skill provided as sample code for educational and demonstration purposes only. Findings should be reviewed and validated before acting on them. See the project's README for full terms and licensing information.*

*Before sharing this report outside your organization, mask or omit the AWS account ID and any cluster ARNs.*
```

### Step 9: Deliver the Report

Deliver the complete markdown report **inline in your response**. Inline delivery is the only always-available channel — the DevOps Agent has no guaranteed workspace file access, so do not depend on writing a file.

Use this suggested title line for the report so it is easy to identify:
`EKS Operation Review — <cluster-name> — <YYYY-MM-DD HH:MM>`

If (and only if) a workspace is available in your environment, you MAY additionally save a copy to a `reports/` subfolder using that same name with a `.md` extension — but always deliver the report inline regardless.

### Step 10: Report Format Notes

Deliver the report as markdown. Do NOT pause to ask whether to convert it to HTML or any other format — the agent runs autonomously and never waits for mid-run input. If the requester wants a different format, they can ask in a follow-up.
