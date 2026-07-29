# Changelog

All notable changes to this project will be documented in this file.

## [1.2.0] - 2026-07-21

### Fixed
- IAM policy correctness: resource-type ARNs for DescribeNodegroup/DescribeAddon/DescribeAccessEntry, DescribeAddonVersions and DescribeClusterVersions moved to the account-level statement, and an accurate note on which actions support resource-level permissions
- Credential handling: names-only Secret/env detection with never-record/never-echo rules (split designs for the Claude Code skill and the DevOps Agent port)
- Rubric decidability: non-overlapping RED/AMBER/GREEN bands with explicit RED→AMBER→GREEN evaluation order, an N/A rating excluded from scoring, and one owning check per cross-category signal
- Factual corrections: live DescribeClusterVersions version-currency check with a dated fallback table, corrected extended-support cost math, network-policy detection method, CoreDNS 1-per-16-nodes ratio, AL2 EOL status, and container-insights log prefix
- DevOps Agent port autonomy: removed the interactive HTML prompt and workspace file-write assumption, added HARD STOP for a nonexistent named cluster, and disambiguated routing for current-state (no-target) reviews
- HTML report converter: single-asterisk emphasis rendering and single-row table handling

## [1.1.0] - 2026-04-24

### Added
- Pod Security Admission (PSA) check (item 3.4) in access-identity assessment
- Pod Security Standards reference URL in report generation
- Concrete thresholds throughout steering files: OOMKilled event counts, CoreDNS scaling ratios, NodeLocal DNSCache cluster size guidance, HPA minReplicas context, maxUnavailable math for small deployments, CloudWatch log retention recommendation, LB deregistration delay alignment
- `logs:DescribeLogGroups` and `eks:DescribeAddonVersions` to README IAM permissions
- Python and environment patterns to .gitignore
- CHANGELOG.md

### Changed
- Version calendar in cluster-lifecycle.md now includes "Last verified" date and link to live AWS calendar
- CLAUDE.md expanded from 18 lines to full overview with prerequisites, workflow phases, and critical rules
- Cluster Autoscaler rated AMBER (legacy) vs Karpenter/Auto Mode rated GREEN (AWS-preferred path)
- Code block language tags now preserved in HTML converter output

## [1.0.0] - 2026-04-02

### Added
- Initial release with 10 assessment areas and 37 check items
- Steering file architecture for modular check definitions
- Markdown to HTML report converter (zero dependencies)
- Pre-verified AWS documentation reference map
- Support for both AWS-managed and open-source EKS MCP servers
