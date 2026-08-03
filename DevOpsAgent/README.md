# EKS Operation Review — AWS DevOps Agent Skill

This folder contains the **AWS DevOps Agent** port of the EKS Operation Review skill. It performs the same 10-section operational excellence assessment as the Claude Code version, adapted for the DevOps Agent platform.

## Structure

```
DevOpsAgent/
├── SKILL.md              # Required: skill metadata + instructions
├── references/           # Per-section check logic (11 files)
│   ├── cluster-lifecycle.md
│   ├── infrastructure-as-code.md
│   ├── access-identity.md
│   ├── observability.md
│   ├── workload-configuration.md
│   ├── networking.md
│   ├── autoscaling.md
│   ├── deployment-practices.md
│   ├── operational-processes.md
│   ├── addon-management.md
│   └── report-generation.md
├── assets/               # Data files (currently empty — extend as needed)
└── README.md             # This file
```

## Install

### Option 1: Import from GitHub (recommended)

1. Open your DevOps Agent Space → Knowledge → Skills → **Add skill**
2. Select **Import from repository**
3. Point at this `DevOpsAgent/` directory (the one containing `SKILL.md`)

### Option 2: Upload as zip

```bash
cd DevOpsAgent
zip -r ../eks-operation-review-skill.zip .
# Upload the zip via: Knowledge → Skills → Add skill → Upload
```

Verify `SKILL.md` is at the zip root:
```bash
unzip -l ../eks-operation-review-skill.zip | head -5
```

### Option 3: Create in UI

Paste the contents of `SKILL.md` into the skill editor. Then add `references/` files as additional documents.

## Resource Access (Prerequisites)

The DevOps Agent IAM role must have access to the target EKS cluster. In order for the
DevOps Agent to run the full review, the cluster access permissions must be updated
beyond the default setup — follow both steps below.

**Official guide:** [Configuring EKS access for DevOps Agent](https://docs.aws.amazon.com/devopsagent/latest/userguide/configuring-integrations-and-knowledge-aws-eks-access-setup.html)

### Step 1: Create the access entry

Ensure the cluster authentication mode includes **EKS API** (`API` or
`API_AND_CONFIG_MAP`), then create an access entry for the DevOps Agent's IAM role.

In all commands below, replace:

- `<CLUSTER>` — your EKS cluster name
- `<REGION>` — the cluster's AWS region
- `<DEVOPS_AGENT_ROLE_ARN>` — the Agent Space IAM role ARN, e.g.
  `arn:aws:iam::111122223333:role/service-role/DevOpsAgentRole-AgentSpace-abc123`
  (find it in the DevOps Agent console under **Agent Space → Capabilities → Cloud →
  Primary source → Edit**)

**If the role has no access entry yet** (a fresh setup), run both commands:

   ```
   aws eks create-access-entry \
     --cluster-name <CLUSTER> \
     --region <REGION> \
     --type STANDARD \
     --kubernetes-groups eks-operation-reviewers \
     --principal-arn <DEVOPS_AGENT_ROLE_ARN>

   aws eks associate-access-policy \
     --cluster-name <CLUSTER> \
     --region <REGION> \
     --principal-arn <DEVOPS_AGENT_ROLE_ARN> \
     --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonAIOpsAssistantPolicy \
     --access-scope type=cluster
   ```

**If the access entry already exists** (e.g. created earlier via the EKS console, which
does not add Kubernetes groups), `create-access-entry` will fail with
`ResourceInUseException`. Add the group to the existing entry instead:

   ```
   aws eks update-access-entry \
     --cluster-name <CLUSTER> \
     --region <REGION> \
     --kubernetes-groups eks-operation-reviewers \
     --principal-arn <DEVOPS_AGENT_ROLE_ARN>
   ```

### Step 2: Grant review read permissions

Bind a least-privilege ClusterRole to the group from Step 1. This grants read-only
access to only the resources the review checks — no Secrets access — and the manifest
contains no IAM ARNs, so the same file works unchanged in every cluster.

   ```yaml
   # eks-operation-reviewer-rbac.yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: eks-operation-reviewer
   rules:
     - apiGroups: ["autoscaling"]
       resources: ["horizontalpodautoscalers"]
       verbs: ["get", "list"]
     - apiGroups: ["policy"]
       resources: ["poddisruptionbudgets"]
       verbs: ["get", "list"]
     - apiGroups: ["networking.k8s.io"]
       resources: ["networkpolicies"]
       verbs: ["get", "list"]
     - apiGroups: [""]
       resources: ["serviceaccounts"]
       verbs: ["get", "list"]
     - apiGroups: ["rbac.authorization.k8s.io"]
       resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]
       verbs: ["get", "list"]
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: eks-operation-reviewer
   subjects:
     - kind: Group
       name: eks-operation-reviewers
       apiGroup: rbac.authorization.k8s.io
   roleRef:
     kind: ClusterRole
     name: eks-operation-reviewer
     apiGroup: rbac.authorization.k8s.io
   ```

   Unlike the AWS CLI commands above, `kubectl` does not take a cluster or region flag —
   it applies to whatever cluster your current kubeconfig context points at. Point it at
   the target cluster first:

   ```
   aws eks update-kubeconfig --name <CLUSTER> --region <REGION>
   kubectl apply -f eks-operation-reviewer-rbac.yaml
   ```

   Verify both halves of the setup. First confirm the access entry actually carries the
   group — this is the step most often missed (the console access-entry flow does not add
   groups), and without it the binding applies to nobody:

   ```
   aws eks describe-access-entry \
     --cluster-name <CLUSTER> \
     --region <REGION> \
     --principal-arn <DEVOPS_AGENT_ROLE_ARN> \
     --query 'accessEntry.kubernetesGroups'
   ```

   Expected output: `["eks-operation-reviewers"]`. If it shows `[]`, the access entry is
   not in the group, so the ClusterRoleBinding applies to nobody. Fix it by adding the
   group, then re-run the check above:

   ```
   aws eks update-access-entry \
     --cluster-name <CLUSTER> \
     --region <REGION> \
     --kubernetes-groups eks-operation-reviewers \
     --principal-arn <DEVOPS_AGENT_ROLE_ARN>
   ```

   Then confirm the binding grants the reads (these test the RBAC objects only — they
   pass regardless of the access entry, so always check the group above too):

   ```
   kubectl auth can-i list horizontalpodautoscalers --as-group eks-operation-reviewers --as review-check
   kubectl auth can-i list clusterrolebindings --as-group eks-operation-reviewers --as review-check -A
   ```

   Both should print `yes`. The `-A` flag on cluster-scoped resources avoids a spurious
   "not namespace scoped" warning.

### Rolling out at scale

For fleets of clusters, script the two steps above with a per-cluster loop, or manage
them via Terraform (`aws_eks_access_entry`, `aws_eks_access_policy_association`, plus the
RBAC manifest) or GitOps (Argo CD / Flux syncing the ClusterRole and binding to every
cluster). Because the manifest is identical everywhere — no per-cluster ARNs — it can be
committed once and fanned out.

### Step 3: Verify AWS API permissions

Steps 1–2 grant access to the Kubernetes API. The review also calls AWS APIs directly
(describing the cluster, node groups, add-ons, VPC, IAM roles, logging, and alarms).
Agent Space setup normally attaches the AWS-managed
[`AIDevOpsAgentAccessPolicy`](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AIDevOpsAgentAccessPolicy.html)
to the primary cloud source role, which already covers all of these — in that case there
is nothing to do. Confirm with:

```
aws iam list-attached-role-policies --role-name <AGENT_SPACE_ROLE_NAME>
```

(Use the bare role name, not the ARN.)

If your organization replaces the managed policy with a custom scoped one, it must allow:

- `eks:Describe*`, `eks:List*`
- `ec2:DescribeSubnets`, `ec2:DescribeVpcs`
- `iam:ListAttachedRolePolicies`, `iam:ListRolePolicies`, `iam:GetPolicy`, `iam:GetPolicyVersion`
- `logs:DescribeLogGroups`
- `cloudwatch:DescribeAlarms`

Checks that hit a missing AWS API permission are marked UNKNOWN with the denied action
in the failure reason — add that action to the custom policy and re-run.

## Usage

Invoke with prompts like:
- "Run an EKS operation review for cluster `prod-app` in `us-west-2`"
- "Assess EKS cluster `my-cluster` operational posture"
- "Check EKS networking for cluster `demo`"
- "Review RBAC and access configuration on my EKS cluster"

For a clean single-pass run, specify cluster name and region up front.

## Differences from Claude Code Version

| Aspect | Claude Code | DevOps Agent |
|--------|-------------|--------------|
| Entry point | `.claude/commands/eks-operation-review.md` | `SKILL.md` (flat folder root) |
| Check logic | `steering/` directory | `references/` directory |
| HTML conversion | `tools/report_to_html.py` script | Agent generates HTML inline |
| Live AWS docs | `awslabs.aws-documentation-mcp-server` MCP | Not bundled; uses embedded reference URLs. Connect a docs MCP at Agent Space level for live lookups |
| EKS API access | `awslabs.eks-mcp-server` MCP (bundled) | Configured at Agent Space level (IAM role + EKS access entry) |
| Interaction model | Interactive (asks user mid-run) | Autonomous with HARD STOP on ambiguity |
| Tool names | Specific MCP tool names (`list_k8s_resources`, etc.) | Generic capability phrases (EKS APIs, Kubernetes APIs) |
| Executables | Python scripts allowed | No executables — documents only |

## Live Data / Freshness

The embedded EKS version calendar in `references/cluster-lifecycle.md` was last verified **2026-04-24**. If a documentation-capable MCP server (e.g., AWS Documentation MCP, AWS Knowledge MCP) is connected to your Agent Space, the agent can cross-check version data against live docs.

Without live lookup, the agent flags results as potentially stale.

## Agent Types

This skill targets: **Generic** (default — works with all DevOps Agent types).

It is also suitable for: On-demand, Evaluation.

## Size

Total skill size: ~65 KB (well under the 6 MB limit).

## License

Same as the parent project — [MIT-0](../LICENSE).
