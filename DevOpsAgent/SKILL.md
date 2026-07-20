---
name: eks-operation-review
description: >-
  Perform a structured EKS operational excellence assessment against a live cluster.
  Covers 10 areas: cluster lifecycle, infrastructure as code, access and identity,
  observability, workload configuration, networking, autoscaling, deployment practices,
  operational processes, and add-on management. Produces a GREEN/AMBER/RED rated report
  with prioritized recommendations. Activate for any request to audit, review,
  health-check, or score an EKS cluster operational posture, including section-scoped
  reviews, including current-state add-on and deprecated-API review with no target version. Not for upgrade readiness assessment against a target version, cluster discovery, or architectural design advice.
---

# EKS Operation Review

This skill performs a structured 10-section operational assessment of a live EKS cluster, producing a rated report with prioritized recommendations.

## When to Activate

Activate for any request to:
- Audit, review, health-check, or score an EKS cluster's operational posture
- Assess a specific operational area (e.g., "check my EKS networking", "review RBAC on my cluster", "EKS observability assessment")
- Generate an EKS operational review report

**Do NOT activate for:** upgrade readiness assessments, cluster discovery, architectural design advice, general Kubernetes questions, AWS troubleshooting, cluster creation, or one-off kubectl commands.

## Execution Model

This skill runs **autonomously** — it does NOT pause for user input mid-execution. It performs a **HARD STOP** on ambiguity instead of guessing.

### HARD STOP Conditions

If any of these conditions are met, produce a HARD STOP output and no report:

```
## HARD STOP
- **Reason:** [why execution cannot proceed]
- **What was found:** [clusters/resources discovered]
- **What is needed:** [specific input required to proceed]
```

HARD STOP triggers:
1. Multiple EKS clusters found and none specified by the user — list them and stop
2. No EKS clusters found in the configured region
3. Cluster access denied — IAM permissions or Kubernetes RBAC (the agent's role cannot describe the cluster or reach the Kubernetes API)
4. EKS or Kubernetes API access is not available or does not respond
5. A user-named cluster does not exist — if the user specified a cluster and DescribeCluster returns not-found (or it is absent from ListClusters), STOP immediately and report the error; do not fall back to a different cluster

### Passing Inputs

For a clean single-pass run, specify: cluster name, region (if not default), and optionally which sections to assess. Example: "Run EKS operation review for cluster `prod-app` in `us-west-2`"

## Prerequisites

The DevOps Agent must have:
- **EKS cluster access** via its IAM role — see [EKS access setup](https://docs.aws.amazon.com/devopsagent/latest/userguide/configuring-integrations-and-knowledge-aws-eks-access-setup.html)
- **AWS API permissions** for: `eks:Describe*`, `eks:List*`, `ec2:DescribeSubnets`, `ec2:DescribeVpcs`, `iam:ListAttachedRolePolicies`, `iam:ListRolePolicies`, `logs:DescribeLogGroups`, `cloudwatch:DescribeAlarms`
- **Kubernetes RBAC access** to list/get Nodes, Pods, Deployments, Services, DaemonSets, Namespaces, and related resources

## Steering File Map

Before executing checks for any section, load the corresponding reference file from `references/`.

| User Request | Reference File(s) to Load |
|---|---|
| Full review / assess / audit / health check | ALL files in order: cluster-lifecycle → addon-management, then report-generation |
| Version currency / lifecycle (current state) | `references/cluster-lifecycle.md` |
| IRSA / RBAC / access / pod identity / endpoint | `references/access-identity.md` |
| Logging / metrics / alerting / observability | `references/observability.md` |
| Resource requests / probes / PDB / image tags / storage | `references/workload-configuration.md` |
| IP / subnet / DNS / CoreDNS / network policy | `references/networking.md` |
| Autoscaling / Karpenter / HPA / topology spread | `references/autoscaling.md` |
| Deployment / rollout / CI/CD / graceful shutdown | `references/deployment-practices.md` |
| Runbook / on-call / backup / DR / Velero | `references/operational-processes.md` |
| Add-on / node monitoring / cluster insights | `references/addon-management.md` |
| Deliver report | `references/report-generation.md` |
| IaC / GitOps / ArgoCD / Flux / drift | `references/infrastructure-as-code.md` |

## Assessment Overview

| # | Section | Key Checks |
|---|---------|------------|
| 01 | Cluster Lifecycle & Upgrades | Version currency, data plane alignment, deprecated APIs, add-on compatibility, upgrade process |
| 02 | Infrastructure as Code & GitOps | IaC provenance, GitOps tools, drift detection, RBAC in code |
| 03 | Access & Identity | IRSA/Pod Identity, least privilege RBAC, API server endpoint security, Pod Security Admission |
| 04 | Observability | Control plane logging, metrics stack, log aggregation, alerting |
| 05 | Workload Configuration | Resource requests/limits, health probes, PDBs, image tags, storage |
| 06 | Networking | IP capacity, CoreDNS health, network policies |
| 07 | Autoscaling | Cluster autoscaler/Karpenter, HPA, topology spread |
| 08 | Deployment Practices | Rollout strategy, CI/CD, graceful shutdown |
| 09 | Operational Processes | Backup/DR, tool presence (Velero, AWS Backup) |
| 10 | Add-on Management | Managed add-ons, node health monitoring, cluster insights |

~70-75% of items are fully automatable. Items requiring human knowledge (runbooks, on-call processes) are marked UNKNOWN with suggestions for investigation.

## Assessment Workflow

### Step 0: Pre-flight

Verify access before starting the assessment:

1. **List clusters** — use EKS ListClusters API. If multiple found and none specified → HARD STOP with list. If one found → proceed with that cluster.
2. **Describe cluster** — use EKS DescribeCluster API. Report: cluster name, Kubernetes version, platform version, region, status, authentication mode. If the cluster status is not ACTIVE (e.g., UPDATING, CREATING), note the status in the report and proceed with the data available — do NOT HARD STOP on a non-ACTIVE status.
3. **Verify Kubernetes access** — list Nodes via Kubernetes API.
   - Success → proceed
   - Failure → HARD STOP with access error details

### Steps 1-10: Run Assessment

Load each reference file in section order. For each section:
1. Load the reference file from `references/`
2. Execute the checks described in it using available EKS and Kubernetes APIs
3. Rate each item using the rubric below

**Error recovery:** If a section fails entirely (API unreachable, permissions denied for all checks), mark all items as UNKNOWN with failure reason, then proceed to next section. Do not let one failed section block the entire assessment.

### Step 11: Generate Report

Load `references/report-generation.md` and produce the report following its structure and consistency rules.

## Rating Rubric

| Rating | Meaning |
|--------|---------|
| GREEN | Fully implemented — matches EKS best practices |
| AMBER | Partial or inconsistent — improvement opportunity |
| RED | Not implemented or significant gap — action needed |
| N/A | Check does not apply to this cluster (e.g. no stateful workloads) — excluded from scoring |
| UNKNOWN | Cannot be determined from cluster data — investigate manually |

### Rules

- Only rate based on what was actually observed — never assume
- If a check fails or returns no data, mark UNKNOWN
- Prioritize by blast radius: security > availability > cost
- Every RED finding must have a specific, actionable recommendation

## Report Output

### Filename

`EKS-Operation-Review-<cluster-name>-<YYYY-MM-DD>-<HHMM>.md`

### Report Template

```markdown
# EKS Operation Review Report
Cluster: [name] | Region: [region] | Version: [version]
Date: [YYYY-MM-DD HH:MM]

## Executive Summary
[2-3 paragraphs. Strengths first, then gaps. Every rating mentioned must match findings.]

## Maturity Score
| Rating | Count | Percentage |
|--------|-------|------------|
| GREEN | X | X% |
| AMBER | X | X% |
| RED | X | X% |
| N/A | X | -- |
| UNKNOWN | X | -- |

## Findings
[One table per section with columns: Item | Status | Current State | Recommendation | References]

## Prioritized Actions

### Critical (Address within 30 days)
[All RED items, ordered: security > availability > cost]

### Important (Address within 90 days)
[All AMBER items]

### Quick Wins
[Items fixable in < 1 hour]

## Items to Investigate Manually
[UNKNOWN items with specific questions]

## AWS Reference Links
[Grouped by topic — use pre-verified URLs from references/report-generation.md]
```

### Consistency Rules (MANDATORY)

1. Ratings must be consistent across the entire report
2. Prioritized Actions must reference the finding ID (e.g., "4.1 — Control Plane Logging RED")
3. Every RED must appear in Critical (or Quick Wins if fixable in < 1 hour); every AMBER in Important (or Quick Wins if fixable in < 1 hour)
4. Executive Summary must match the findings — do not call something "critical" if it's AMBER
5. One row per finding in Prioritized Actions — never bundle multiple findings

### Report Footer

Append at the end:

```markdown
---
*This report was generated by an AWS DevOps Agent skill provided as sample code for
educational and demonstration purposes only. Findings should be reviewed and validated
before acting on them. See the project's README and LICENSE for full terms.*
```

## Live Data Caveat

This skill determines version support status primarily from the live EKS **DescribeClusterVersions** API. It also includes embedded reference tables (a fallback EKS version table, compatibility data, pre-verified AWS documentation URLs) that may become stale over time; these are used only when the live API is unavailable.

If the live API is not available, flag results as: "⚠️ Version data based on embedded fallback table (last verified 2026-07-20). Verify against official docs."
