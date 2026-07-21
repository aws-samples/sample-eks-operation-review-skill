# Access & Identity

## Purpose
Assess IAM and RBAC configuration for security and operational excellence — pod-level permissions, least privilege, and API server access controls.

## Checks to Execute

### 3.1 — Pod-Level AWS Permissions (IRSA / EKS Pod Identity)

**What to check:**
- OIDC provider configured (prerequisite for IRSA)
- EKS Pod Identity associations
- Service accounts with IRSA annotations (`eks.amazonaws.com/role-arn`)
- Node IAM role policies (are they overly broad?)
- AWS credentials in Kubernetes Secrets or pod env vars

**How to check:**
1. Describe cluster → `identity.oidc.issuer` (OIDC configured?)
2. List Pod Identity associations
3. List ServiceAccounts across all namespaces → filter for IRSA annotation
4. List node groups → describe first one → get `nodeRole` → get policies for that role
5. List Secrets across namespaces → project **key names only** (never values), enriched with namespace/name so findings are attributable: `kubectl get secrets -A -o go-template='{{range .items}}{{.metadata.namespace}}/{{.metadata.name}}: {{range $k,$v := .data}}{{$k}} {{end}}{{"\n"}}{{end}}'` — flag secrets whose key names match the credential-shaped pattern set (exact: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`; substring: `SECRET`, `TOKEN`, `PASSWORD`, `API_KEY`). **Never record raw Secrets output — credential values must never appear in output.**
6. List pods → detect credential-shaped env var **names only** (never values) across containers, initContainers, and `envFrom` refs: `kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\t"}{range .spec.containers[*].env[*].name}{@}{" "}{end}{range .spec.containers[*].envFrom[*].configMapRef.name}{@}{" "}{end}{range .spec.containers[*].envFrom[*].secretRef.name}{@}{" "}{end}{range .spec.initContainers[*].env[*].name}{@}{" "}{end}{range .spec.initContainers[*].envFrom[*].configMapRef.name}{@}{" "}{end}{range .spec.initContainers[*].envFrom[*].secretRef.name}{@}{" "}{end}{"\n"}{end}' | grep -iE 'SECRET|TOKEN|PASSWORD|API_KEY|AWS_ACCESS|AWS_SECRET|AWS_SESSION'`. Credential-shaped = exact `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN` or substring `SECRET`, `TOKEN`, `PASSWORD`, `API_KEY`. Per output line, the tokens after the tab are env var names first, then `envFrom` secretRef/configMapRef names, in command order. Report each finding as `<ns>/<pod> — credential-shaped <env var|secret ref|configMap ref> name found: <name> (value never read)`.

**Rating:**
- 🟢 GREEN: All AWS-accessing pods use IRSA or Pod Identity, node role is minimal, no hardcoded credentials
- 🟡 AMBER: IRSA partially adopted, or node role has some extra permissions
- 🔴 RED: No IRSA/Pod Identity, node role has broad permissions (S3FullAccess, DynamoDBFullAccess), or hardcoded AWS credentials found — report the pod/secret **location and key/env-var name only**; **never record or echo the credential value**
- ⬜ UNKNOWN: Cannot determine which pods need AWS access vs which don't

**Key talking point:** Node-level IAM = every pod on that node inherits the same permissions. One compromised pod gets access to everything.

---

### 3.2 — Least Privilege RBAC

**What to check:**
- ClusterRoleBindings to cluster-admin (count and subjects)
- ClusterRoles with wildcard permissions (`*` verbs on `*` resources)
- Ratio of namespace-scoped RoleBindings vs cluster-scoped ClusterRoleBindings
- Service accounts with cluster-admin

**How to check:**
1. List ClusterRoleBindings → filter `roleRef.name == "cluster-admin"` → count and list subjects
2. List ClusterRoles → check rules for `verbs: ["*"]` and `resources: ["*"]`
3. Count RoleBindings across all namespaces vs ClusterRoleBindings
4. List application namespaces (exclude kube-system, kube-public, kube-node-lease, default)

**Rating:**
- 🟢 GREEN: Namespace-scoped RBAC, cluster-admin limited to 1-2 break-glass bindings
- 🟡 AMBER: Some namespace isolation but cluster-admin overused (>3 bindings)
- 🔴 RED: Developers have cluster-admin in production, wildcard service accounts, no namespace isolation
- ⬜ UNKNOWN: Cannot determine if RBAC is reviewed periodically — suggest user investigate
- **Evaluation order:** assess RED first; if not RED, assess AMBER; otherwise GREEN. Keeps the bands exhaustive and non-overlapping.
- **Scoring authority:** this check owns least-privilege RBAC / cluster-admin scoring; check 2.4 defers here for the cluster-admin signal.

---

### 3.3 — EKS API Server Endpoint & Network Access

**What to check:**
- Public/private endpoint configuration
- Public access CIDR restrictions
- Cluster security group inbound rules
- Whether audit logging is enabled

**How to check:**
1. Describe cluster → `resourcesVpcConfig.endpointPublicAccess`, `endpointPrivateAccess`, `publicAccessCidrs`
2. Describe cluster → `resourcesVpcConfig.clusterSecurityGroupId`, then describe that security group's inbound rules (e.g. `ec2:DescribeSecurityGroupRules` filtered to the group) → check for overly-broad ingress (0.0.0.0/0 on sensitive ports)
3. Describe cluster → `logging.clusterLogging` (check if audit log type is enabled)

**Rating:**
- 🟢 GREEN: Private endpoint enabled, public either disabled or CIDR-restricted
- 🟡 AMBER: Public endpoint CIDR-restricted with private access disabled, or private enabled but audit logging off (audit logging is independently owned by check 4.1)
- 🔴 RED: Public endpoint open to `0.0.0.0/0`
- ⬜ UNKNOWN: Cannot determine MFA/SSO requirements — suggest user investigate
- **Evaluation order:** assess RED first; if not RED, assess AMBER; otherwise GREEN. Keeps the bands exhaustive and non-overlapping.

**Key talking point:** An API server open to 0.0.0.0/0 is exposed to the internet. You're relying entirely on authentication.

---

### 3.4 — Pod Security Admission (PSA)

**What to check:**
- Pod Security Standards enforcement via namespace labels (`pod-security.kubernetes.io/enforce`)
- Which namespaces have PSA labels and at what level (privileged, baseline, restricted)
- Production namespaces without PSA enforcement

**How to check:**
1. List namespaces → inspect labels for `pod-security.kubernetes.io/enforce`, `pod-security.kubernetes.io/warn`, `pod-security.kubernetes.io/audit`
2. Count namespaces with enforcement vs without (exclude kube-system, kube-public, kube-node-lease)
3. Check if any application namespaces use `privileged` enforce level

**Rating:**
- 🟢 GREEN: PSA labels on all application namespaces, `baseline` or `restricted` enforcement
- 🟡 AMBER: PSA labels on some namespaces but not all, or only `warn`/`audit` mode (no enforcement)
- 🔴 RED: No PSA labels on any namespace, or application namespaces set to `privileged`
- ⬜ UNKNOWN: Cannot determine if third-party admission controller (OPA/Gatekeeper, Kyverno) handles pod security instead

**Key talking point:** PodSecurityPolicy was removed in Kubernetes 1.25. Pod Security Admission is the built-in replacement. Without it (or a third-party equivalent), any pod spec is accepted — including privileged containers.
