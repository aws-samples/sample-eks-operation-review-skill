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

The DevOps Agent IAM role must have access to the target EKS cluster.

**Official guide:** [Configuring EKS access for DevOps Agent](https://docs.aws.amazon.com/devopsagent/latest/userguide/configuring-integrations-and-knowledge-aws-eks-access-setup.html)

Quick setup:
1. Ensure cluster authentication mode includes **EKS API**
2. Create an EKS access entry for the DevOps Agent's IAM role:
   ```
   aws eks create-access-entry \
     --cluster-name <CLUSTER> \
     --region <REGION> \
     --type STANDARD \
     --principal-arn <DEVOPS_AGENT_ROLE_ARN>

   aws eks associate-access-policy \
     --cluster-name <CLUSTER> \
     --region <REGION> \
     --principal-arn <DEVOPS_AGENT_ROLE_ARN> \
     --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonAIOpsAssistantPolicy \
     --access-scope type=cluster
   ```

3. The Agent Space primary account role also needs these AWS API permissions:
   - `eks:Describe*`, `eks:List*`
   - `ec2:DescribeSubnets`, `ec2:DescribeVpcs`
   - `iam:ListAttachedRolePolicies`, `iam:ListRolePolicies`
   - `logs:DescribeLogGroups`
   - `cloudwatch:DescribeAlarms`

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
