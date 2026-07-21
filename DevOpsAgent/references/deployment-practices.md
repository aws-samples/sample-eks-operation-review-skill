# Deployment Practices

## Purpose
Assess deployment strategies, CI/CD integration, and graceful shutdown configuration.

## Automation Note
CI/CD pipeline details (approval gates, post-deployment tests) are not fully detectable from cluster state. The skill checks for tool presence and configuration; process maturity items are marked UNKNOWN.

## Checks to Execute

### 8.1 — Deployment Strategy & Rollback

**What to check:**
- Deployment strategies in use (RollingUpdate vs Recreate)
- maxUnavailable and maxSurge settings — evaluate the RESOLVED integer, not the raw percentage. The default 25% maxUnavailable rounds DOWN for small replica counts (25% of 2 = 0, 25% of 3 = 0), which is already safe; the risk is when it resolves to >= 1 (25% of 4 = 1), allowing capacity to drop mid-rollout. Recommend an explicit maxUnavailable: 0, maxSurge: 1 for latency-sensitive workloads so a rollout never reduces ready replicas.
- Argo Rollouts resources
- Flagger Canary resources
- terminationGracePeriodSeconds and preStop hooks

**How to check:**
1. List Deployments → inspect `spec.strategy.type`, `rollingUpdate.maxUnavailable`, `rollingUpdate.maxSurge`
2. For each deployment, resolve maxUnavailable against the replica count (round the percentage DOWN to an integer). Flag deployments where the resolved maxUnavailable is >= 1 (capacity can drop during a rollout); a resolved value of 0 is safe.
3. List Rollouts (Argo Rollouts CRD, if exists)
4. List Canaries (Flagger CRD, if exists)
5. Inspect Deployments for `terminationGracePeriodSeconds` and `lifecycle.preStop`

**Rating:**
- 🟢 GREEN: Resolved maxUnavailable is 0 (zero-downtime rollout), graceful shutdown configured
- 🟡 AMBER: Rolling update where resolved maxUnavailable is >= 1, or no progressive delivery
- 🔴 RED: Recreate strategy on user-facing workloads (full downtime), or no graceful shutdown
- ⬜ UNKNOWN: Cannot determine rollback speed or process — suggest user investigate
- **Evaluation order:** assess RED first; if not RED, assess AMBER; otherwise GREEN. Keeps the bands exhaustive and non-overlapping.

---

### 8.2 — CI/CD Pipeline Integration

**What to check:**
- ECR repositories: scan-on-push, tag immutability
- Admission webhooks enforcing image policies
- Image registries in use (ECR vs public)

**How to check:**
1. Describe ECR repositories → scanOnPush, imageTagMutability
2. List ValidatingWebhookConfigurations → filter for image/policy-related names
3. List running pods → aggregate image registries

**Rating:**
- 🟢 GREEN: CI/CD pipeline present with admission enforcement (image scan-on-push / tag immutability / registry trust are rated under 5.4)
- 🟡 AMBER: Pipeline exists but no admission enforcement
- 🔴 RED: No CI/CD evidence and no admission enforcement (registry trust is rated under 5.4)
- ⬜ UNKNOWN: Cannot determine full pipeline from cluster state — suggest user investigate
- **Scoring authority:** ECR scan-on-push and image tag immutability are rated under check 5.4 (Image Tag Hygiene); 8.2 assesses pipeline evidence and admission enforcement.

---

### 8.3 — Graceful Shutdown & Connection Draining

**What to check:**
- Deployments with preStop hooks vs without
- terminationGracePeriodSeconds (default 30s vs customized)
- Services with AWS Load Balancer annotations (deregistration delay matters)
- Ingress resources with target group attributes

**How to check:**
1. List Deployments → count those with `lifecycle.preStop` vs without
2. List Deployments → check `terminationGracePeriodSeconds` (null = default 30s)
3. List Services → check for `service.beta.kubernetes.io/aws-load-balancer-type` annotation
4. List Ingresses → check for `alb.ingress.kubernetes.io/target-group-attributes` annotation (`terminationGracePeriodSeconds` should be greater than or equal to the target-group deregistration delay — the pod must stay alive until in-flight connections finish draining)

**Rating:**
- 🟢 GREEN: preStop hooks on all externally-facing deployments, grace period tuned, LB drain aligned
- 🟡 AMBER: Some-but-not-all externally-facing deployments have preStop hooks, OR there are no externally-facing workloads and preStop is absent
- 🔴 RED: Externally-facing deployments exist (a deployment fronted by a LoadBalancer Service or backed by an Ingress) AND all of them lack preStop hooks, OR `terminationGracePeriodSeconds` is shorter than the target-group deregistration delay
- ⬜ UNKNOWN: Cannot determine if 502s occur during deployments — suggest user investigate
- **Externally-facing** = a deployment exposed via a LoadBalancer-type Service or an Ingress. Zero externally-facing deployments is NOT RED (it lands in AMBER when preStop is absent).
- **Evaluation order:** assess RED first; if not RED, assess AMBER; otherwise GREEN. Keeps the bands exhaustive and non-overlapping.

**Key talking point:** There's a race condition during pod termination. The LB still sends traffic for a few seconds after SIGTERM. A preStop sleep of 5-10s fixes it.
