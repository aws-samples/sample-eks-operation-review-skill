# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in this project, please report it responsibly.

**Do not open a public GitHub issue for security vulnerabilities.**

Instead, please report security issues by emailing the repository maintainers directly. Include:

1. A description of the vulnerability
2. Steps to reproduce the issue
3. Any potential impact

We will acknowledge receipt within 48 hours and aim to provide a fix or mitigation plan within 7 days.

## Scope

This skill performs **read-only** operations against AWS and Kubernetes APIs. It does not create, modify, or delete any AWS or Kubernetes resources. All operations are `describe`, `list`, and `get` calls.

The skill runs locally on your machine using your existing AWS credentials. It does not transmit credentials or cluster data to any third-party service beyond the Claude Code runtime and configured MCP servers.

## Supply Chain

MCP server versions in `.mcp.json` are pinned to known-good releases to keep behaviour reproducible and avoid pulling unreviewed upstream updates. To upgrade, bump the version strings in `.mcp.json` after reviewing the upstream changelog at [awslabs/mcp](https://github.com/awslabs/mcp/releases).

## Supported Versions

Security updates are applied to the latest release only.
