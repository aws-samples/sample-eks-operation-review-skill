# EKS Operation Review Skill

A Claude Code skill that performs automated EKS Operational Excellence assessments against live clusters.

## Structure

```
.claude/commands/eks-operation-review.md   # Skill entry point — full workflow and rules
steering/                                  # Per-section check instructions (loaded on demand)
tools/report_to_html.py                    # Markdown-to-HTML report converter
.mcp.json                                  # MCP server configuration
SKILL.md                                   # Skill metadata for platform auto-triggering
evals/                                     # Skill evaluation harness (evals.json)
docs/                                      # Additional documentation
DevOpsAgent/                               # AWS DevOps Agent port (SKILL.md, references/, README)
```

## MCP Servers

- `awslabs.eks-mcp-server` — queries EKS cluster state (discovery, describe, Kubernetes API)
- `awslabs.aws-documentation-mcp-server` — optional; used for setup/reference lookups only. It is NOT called during an assessment (the report generator uses the pre-verified reference map instead).

## Running

`/eks-operation-review` — all workflow steps, rules, and report format are defined in the command file.
