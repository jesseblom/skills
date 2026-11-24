---
name: kubernetes-cluster-testing
description: >-
  Use when testing, validating, verifying, or hardening a Kubernetes cluster —
  conformance, functional/end-to-end, health & smoke checks, security/CIS/RBAC/runtime,
  chaos/resilience/disaster-recovery, performance/scale/load, or policy & config validation.
  Helps pick the right testing approach for the goal, then points to a per-domain reference
  file with runnable, command-level workflows for the standard tools (Sonobuoy, Chainsaw,
  helm test, kube-bench, Trivy, kubescape, Falco, Chaos Mesh, Velero, k6, ClusterLoader2,
  kubeconform, Kyverno, Gatekeeper, and more).
---

# Kubernetes Cluster Testing

Testing a Kubernetes cluster is not one activity — it's five distinct questions, each with its
own tools and its own blast radius:

1. **Does it work?** — conformance, functional/e2e, health & smoke testing
2. **Is it secure?** — CIS/posture, RBAC, runtime, supply chain, secrets
3. **Does it survive failure?** — chaos engineering, backup/DR, HA/failover
4. **Does it scale?** — scale, load, autoscaling, resource/node benchmarking
5. **Is its config valid & compliant?** — schema validation, policy-as-code, drift

Work top-down from the question you're actually trying to answer: use the decision table to pick a
domain, read **Before you start**, then open the matching reference file for command-level detail.

> **Currency:** tool names, versions, and commands in the reference files were verified **2026-11-24**
> against official docs and release pages (Kubernetes line **v1.36**). Versions drift — pin the
> current release from each project before use. Deprecations are flagged inline in each file; see
> **Notes on currency** at the end.

## How this skill is organized

The detail lives in four reference files — read only the one(s) the task needs:

| File | Covers |
|---|---|
| [references/functional-health.md](references/functional-health.md) | Conformance (Sonobuoy), functional/e2e (Chainsaw, `helm test`, Ginkgo/envtest), health & readiness checks, post-provision smoke checklist |
| [references/security.md](references/security.md) | CIS/kube-bench, posture scanning (Trivy, kubescape, Polaris), attack surface, RBAC auditing, runtime (Falco/Tetragon), supply chain & admission, secrets & etcd encryption |
| [references/resilience-dr.md](references/resilience-dr.md) | Chaos platforms (Chaos Mesh, Litmus, Chaos Toolkit, kube-monkey), fault-injection recipes, backup/DR (Velero, etcd), HA/failover, RTO/RPO |
| [references/performance-policy.md](references/performance-policy.md) | Scale (ClusterLoader2, kube-burner), load (k6, Fortio, vegeta), autoscaling (HPA/KEDA/Karpenter), benchmarking; schema validation, policy-as-code (Kyverno/Gatekeeper/Conftest), linting, drift |

## How to choose what to test

| The question you're answering | Testing type | First-reach tools | Reference |
|---|---|---|---|
| Does the cluster obey the Kubernetes API spec (portability / certification)? | **Conformance** | Sonobuoy | functional-health |
| Is a freshly provisioned or upgraded cluster actually usable? | **Smoke / acceptance** | `kubectl` checklist, `helm test`, Chainsaw | functional-health |
| Do my workloads / operators / CRDs behave correctly? | **Functional / e2e** | Chainsaw, `helm test`, Ginkgo + envtest | functional-health |
| Is the cluster healthy *right now*? | **Health / readiness** | `/livez` + `/readyz`, Popeye, k9s, Node Problem Detector | functional-health |
| Is it configured securely / to CIS benchmark? | **Security posture** | kube-bench, Trivy, kubescape, Polaris | security |
| Who can do what — any overprivileged subjects? | **RBAC audit** | `kubectl auth can-i`, rbac-tool, kubectl-who-can | security |
| Does runtime threat detection actually fire? | **Runtime security** | Falco (+ event-generator), Tetragon | security |
| Does the supply chain / admission reject bad workloads? | **Supply chain / admission** | cosign, Kyverno, Gatekeeper, Pod Security Admission | security |
| Does it survive pod / node / network / zone failure? | **Resilience / chaos** | Chaos Mesh, LitmusChaos, kube-monkey | resilience-dr |
| Can I recover from catastrophic loss? | **Backup / DR** | Velero, etcd snapshot/restore | resilience-dr |
| Does redundancy (replicas/PDB/leader election) really hold? | **HA / failover** | drain + PDB + lease tests | resilience-dr |
| Does it scale under density / load? | **Performance / scale** | ClusterLoader2, kube-burner, k6 | performance-policy |
| Does autoscaling (pods & nodes) work? | **Autoscaling** | HPA, KEDA, Cluster Autoscaler / Karpenter | performance-policy |
| Are my manifests valid and policy-compliant? | **Policy / config** | kubeconform, Kyverno, Gatekeeper, Conftest | performance-policy |

## Before you start (prerequisites & safety)

**Confirm the target cluster — every time.** Destructive tests on the wrong context are the classic
incident.

```bash
kubectl config current-context          # WHICH cluster am I about to touch?
kubectl config get-contexts
kubectl auth can-i '*' '*'              # do I even have the access these tests need?
```

**Classify the test before running it:**

- **Safe anywhere (read-only / local):** health checks, posture/CVE/RBAC scanners, `kubeconform`,
  `kube-score`, `--dry-run`, `kubectl diff`, Popeye. Run freely.
- **Disruptive — staging first, explicit prod approval, controlled blast radius:**
  `sonobuoy --mode=certified-conformance`, all chaos injection, load/scale tests (ClusterLoader2,
  kube-burner, k6 at volume), DR restore drills, control-plane/`kube-system` experiments.

**Rules for the disruptive class:**

- Never run chaos against the control plane or `kube-system`, or load tests against production,
  without explicit owner sign-off — losing the API server also blinds your chaos controller.
- Define a measurable **steady-state hypothesis** before any chaos experiment (e.g. p99 < 300ms,
  error rate < 0.1%, N replicas Ready); assert it before and after. No hypothesis = vandalism.
- Scope blast radius: namespace/label selectors, `mode: one` / `fixed-percent` before `all`, and
  always set an experiment `duration`/`deadline` so faults auto-revert. Know the kill switch
  (`kubectl delete <chaos-crd>`).
- Prefer a throwaway / `kind` cluster for anything that mutates real state (Chainsaw e2e, Kyverno
  apply, restore drills into a scratch namespace).
- Clean up test resources when done.

**Managed clusters (EKS/GKE/AKS):** you cannot reach etcd or the managed control plane directly.
Control-plane CIS checks, etcd health/snapshot, and encryption-at-rest verification shift to the
provider's settings/console; node and workload checks still apply.

## Recommended workflows

Pick the sequence that matches the moment; each step's detail is in the referenced file.

**New cluster brought up:** (1) health — `kubectl get nodes`, `/readyz?verbose`, `kube-system` pods
Running; (2) smoke/acceptance checklist; (3) conformance (new distro/provider) via
`sonobuoy run --mode=certified-conformance`; (4) security baseline — kube-bench, `trivy k8s cluster`,
`kubescape scan framework nsa`, RBAC audit, PSA + etcd encryption; (5) policy gates — install
Kyverno/Gatekeeper and prove a bad workload is rejected; (6) resilience/DR baseline — Velero +
scheduled backup, one restore drill, one controlled node-drain.

**Before/after a version upgrade:** snapshot etcd + Velero backup *first* → upgrade → health + smoke
checklist → `sonobuoy run --mode=non-disruptive-conformance` for API regressions → re-run
`helm test`/Chainsaw suites and security scans (new k8s version = new CIS mapping).

**CI / pre-merge gates (fast, non-disruptive):** `kubeconform -strict` → kube-score / KubeLinter /
Polaris → `kyverno test` / `gator verify` → `trivy config` + `trivy image` → `conftest test` on
rendered Helm → `kubectl diff` for drift. All offline or read-only; fail the build on nonzero exit.

**Pre-production hardening sign-off:** full security sweep (kube-bench + Trivy + kubescape + RBAC +
Falco event-generator) + a scale/load run (ClusterLoader2 or k6) checked against SIG-Scalability SLOs
+ autoscaling validation + a game-day chaos workflow with a steady-state hypothesis + a timed DR
restore for RTO/RPO.

**Routine / ongoing:** standing in-cluster scanners (Trivy Operator, Falco/Tetragon, Node Problem
Detector), scheduled Velero backups with periodic restore drills, periodic `popeye` hygiene audits,
optionally small chaos experiments in CI.

## Notes on currency

Versions and statuses in the reference files were verified 2026-11-24 (Kubernetes **v1.36** line). Pin
the current release from each project's releases page before use; where a latest point release
couldn't be confirmed, the command is canonical but substitute the current version/tag. Confirmed
deprecations: `kubeval` → `kubeconform`; `datree` sunset; `kube-hunter` unmaintained → Trivy;
`starboard` → Trivy Operator; `/healthz` → `/livez` + `/readyz`; `componentstatuses` deprecated.
Confirmed revival: kube-monkey (active again as of 2026, despite stale Docker Hub tags).
