# Security, compliance & hardening testing

Kubernetes security testing splits into static posture (config/misconfig/CIS), access control
(RBAC), runtime behavior (threat detection), supply chain (signing/admission), and exposure
(secrets, encryption-at-rest). Below are the current, maintained tools with runnable commands.
Version numbers drift—pin the latest from each project's GitHub releases page before use.

> Maintenance flags up front: **kube-hunter is deprecated** (no active development since 2024; Aqua
> points users to Trivy). **Starboard is deprecated and archived** → use `trivy k8s` (CLI) or
> **Trivy Operator** (in-cluster). **Trivy is now the dominant scanner** and absorbed Starboard's
> functionality.

## 1. CIS Benchmark & config hardening

**kube-bench** (Aqua, actively maintained, ~v0.12.0) — runs the CIS Kubernetes Benchmark checks
against control-plane and worker nodes by inspecting running process flags, file permissions, and
config files. Reach for it first on any self-managed cluster to get a pass/fail/warn report mapped
to numbered CIS controls. It auto-detects the node role and Kubernetes version and selects the
matching benchmark (e.g. `cis-1.12` for recent k8s); control-plane tests only run on a node running
the API server.

```bash
# In-cluster as a Job (recommended): applies to whichever node the pod lands on
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench

# Docker on a node, auto-detect by k8s version
docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro \
  -v $(which kubectl):/usr/local/mount-from-host/bin/kubectl \
  -v ~/.kube:/.kube -e KUBECONFIG=/.kube/config \
  -t docker.io/aquasec/kube-bench:latest

# Pin a benchmark explicitly (overrides auto-detect; --version takes precedence over KUBE_BENCH_VERSION)
kube-bench --benchmark cis-1.12
# Managed clusters use dedicated job manifests: job-eks.yaml, job-gke.yaml, job-eks-stig.yaml
```

Managed-provider benchmark values: `gke-1.6.0`, `eks-1.x`, `ack-1.0`, `rke-cis-1.7`, `rke2-cis-1.7`,
`k3s-cis-1.7`, `rh-1.0` (OpenShift). Gotchas: (1) on EKS/GKE/AKS you **cannot** scan the managed
control plane — only node/policy checks run, so expect "info" results for master controls;
(2) kube-bench reports what CIS *recommends*, not what's exploitable — triage warnings against your
threat model.

The **CIS Kubernetes Benchmark** itself (current release **v1.12.0**, maps to ~k8s 1.34/1.35) is
the free PDF spec from cisecurity.org that kube-bench, Trivy, and kubescape all encode. Download it
to understand the rationale and remediation behind a control ID.

## 2. Posture & misconfiguration scanning

**Trivy** (Aqua, the dominant all-in-one scanner) — single binary that scans images, filesystems,
IaC/manifests, and live clusters for CVEs, misconfigurations, exposed secrets, and licenses. This is
the default reach-for tool; it replaced Starboard and kube-hunter's scanning role.

```bash
brew install trivy   # or: curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

trivy k8s --report summary cluster              # whole cluster: CVEs + misconfig + secrets
trivy k8s --scanners misconfig --report all cluster
trivy image --severity HIGH,CRITICAL nginx:1.27 # image CVE scan
trivy config ./manifests/                        # IaC/misconfig scan of YAML/Helm/Terraform
trivy k8s --format cyclonedx --output kbom.json cluster   # KBOM (cluster bill of materials)
```

Gotchas: (1) `trivy k8s cluster` pulls and scans **every** image by default — on large clusters add
`--skip-images` or scope with `--include-namespaces`; (2) the first run downloads the vuln DB (needs
registry/internet egress) — pre-warm with `trivy image --download-db-only` in air-gapped setups.

**Trivy Operator** (in-cluster continuous scanning; replaces starboard-operator) — deploys as a
controller that watches workloads and writes results to CRDs. Use it for standing posture instead of
one-shot CLI runs.

```bash
helm repo add aqua https://aquasecurity.github.io/helm-charts/ && helm repo update
helm install trivy-operator aqua/trivy-operator -n trivy-system --create-namespace --version <pin-latest>
kubectl get vulnerabilityreports,configauditreports,exposedsecretreports -A
```

Gotcha: uninstalling the chart leaves the CRDs; deleting the CRDs deletes all reports.

**kubescape** (ARMO, CNCF, **v4.0** GA March 2026) — scans a live cluster or manifests against named
compliance frameworks (NSA/CISA hardening, MITRE ATT&CK, CIS) and produces a risk score with
remediation. Reach for it when you want framework-mapped compliance output rather than raw CVEs.

```bash
curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash
kubescape scan                              # live cluster, all frameworks
kubescape scan framework nsa                # NSA/CISA hardening guidance
kubescape scan framework mitre              # MITRE ATT&CK
kubescape scan framework cis-v1.23-t1.0.1   # CIS (use `kubescape list frameworks` for exact names)
kubescape scan ./manifests/                 # local YAML/Helm/directory
```

Gotcha: framework identifiers are version-pinned strings — run `kubescape list frameworks` on your
installed build rather than guessing. v4.0 added a GA runtime-detection component and
AI-agent-workload scanning; the free CLI posture scan is what's described here.

**Polaris** (Fairwinds, maintained) — opinionated best-practice linter for manifests and live
workloads (security contexts, resource limits, probes). Lighter and more prescriptive than
Trivy/kubescape; good as a CI gate.

```bash
brew install FairwindsOps/tap/polaris
polaris audit --audit-path ./k8s/ --format pretty   # manifests (helm template first — Polaris reads raw YAML only)
polaris audit --format pretty                        # live cluster
```

**Checkov** (Prisma Cloud / Palo Alto, maintained) — static IaC policy scanner covering Kubernetes
manifests, Helm, Kustomize, Terraform. Reach for it when you want one scanner across k8s + cloud IaC
in CI.

```bash
pip install checkov    # or: brew install checkov
checkov -d . --framework kubernetes
checkov -d ./chart/ --framework helm
```

## 3. Penetration & attack-surface testing

**kube-hunter** (Aqua) — **DEPRECATED.** README states "kube-hunter is not under active development
anymore"; deprecated ~May 2023 in favor of Trivy, last meaningful commit 2024. Do not build new
tooling on it; use Trivy for misconfig/KBOM and the manual kubelet/API checks below for
attack-surface testing.

**kubeletctl** (CyberArk, maintained) — implements the full kubelet API; use it to test whether a
kubelet's read/exec endpoints on port **10250** are reachable without auth (`--anonymous-auth=true`
is the classic finding). Only run against clusters you are authorized to test.

```bash
# install: download binary from github.com/cyberark/kubeletctl releases
kubeletctl scan --cidr 10.0.0.0/24          # find nodes with open kubelet ports on a subnet
kubeletctl pods -s <node-ip>                 # list pods via kubelet (200 + pod list = exposed)
# exec into a container via kubelet (proof-of-exposure; destructive-capable — authorize first)
kubeletctl exec "id" -p <pod> -c <container> -s <node-ip>
```

Manual attack-surface checks (no tool needed):

```bash
curl -sk https://<node-ip>:10250/pods            # unauthenticated kubelet read → should 401
curl -sk https://<api-server>:6443/version        # anonymous API reachability
kubectl auth can-i --list --as=system:anonymous   # what anonymous/unauthenticated can do
```

Remediation to verify: kubelet `--anonymous-auth=false`, `--authorization-mode=Webhook`, API server
`--anonymous-auth=false`, and `system:anonymous`/`system:unauthenticated` not bound to anything
meaningful.

## 4. RBAC & access auditing

Goal: find overprivileged subjects, wildcard permissions, and privilege-escalation paths (especially
service accounts that can create/patch `clusterrolebindings`, `escalate`, or `bind` — which lets them
self-grant cluster-admin at runtime; plain reverse-lookups miss these chains).

**`kubectl auth can-i`** (built-in) — ground truth for "can subject X do Y."

```bash
kubectl auth can-i --list                                              # my own perms
kubectl auth can-i create pods --as=system:serviceaccount:default:myapp -n default
kubectl auth can-i '*' '*' --as=system:serviceaccount:kube-system:foo  # effectively cluster-admin?
kubectl auth can-i --list --as=system:serviceaccount:default:myapp     # full matrix for an SA
```

**kubectl-who-can** (Aqua, krew plugin) — the reverse question: *who* can perform a verb on a resource.

```bash
kubectl krew install who-can
kubectl who-can create pods
kubectl who-can '*' secrets          # who can do anything to secrets
kubectl who-can escalate clusterroles   # privilege-escalation path finder
```

**rbac-tool** (alcideio, krew plugin) — the most complete RBAC toolkit: reverse lookups, who-can,
effective policy rules, and a visual graph.

```bash
kubectl krew install rbac-tool
kubectl rbac-tool lookup                    # subject → roles/bindings
kubectl rbac-tool who-can escalate clusterroles
kubectl rbac-tool policy-rules -e '.*'      # flattened effective rules for all subjects
kubectl rbac-tool viz --outformat dot       # RBAC graph
```

**rbac-lookup** (Fairwinds) — quick, friendly subject→role reverse map.

```bash
brew install FairwindsOps/tap/rbac-lookup
rbac-lookup --kind serviceaccount           # all SAs and what they're bound to
rbac-lookup rob --output wide
```

**kubeaudit** (Shopify) — audits workloads for insecure settings (runAsNonRoot, privileged,
allowPrivilegeEscalation, capabilities, hostNetwork, automountServiceAccountToken).

```bash
brew install kubeaudit
kubeaudit all                     # audit live cluster
kubeaudit all -f ./manifests.yaml # audit manifests
```

Also worth knowing: **KubiScan** (CyberArk) hunts specifically for risky roles/bindings and exposed
tokens. Gotcha across all RBAC tooling: a dangerous permission only matters if a reachable workload
actually mounts that SA token — correlate findings with `automountServiceAccountToken` and which pods
use the SA.

## 5. Runtime threat detection testing

**Falco** (CNCF **graduated** Feb 2024; **v0.45.0**, Sep 2026) — the de-facto runtime security
engine; watches syscalls (eBPF or kernel module) plus k8s audit events and fires on rule matches
(shell in container, write to sensitive path, unexpected outbound, etc.). Test both that it's loaded
and that detections actually fire.

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts && helm repo update
helm install falco falcosecurity/falco -n falco --create-namespace
falcoctl artifact install falco-rules   # rules are managed by falcoctl
```

Validate detections fire using the official **event-generator** (needs Falco ≥0.37):

```bash
# one-shot run of all sample attacks as a Job, then check Falco logs for alerts
docker run -it --rm falcosecurity/event-generator run
# or in-cluster via helm with config.command=test
kubectl logs -n falco -l app.kubernetes.io/name=falco | grep Warning
# targeted manual test:
kubectl exec -it <pod> -- cat /etc/shadow   # should trigger "Read sensitive file" rule
```

Gotchas: (1) wait until Falco logs that it has loaded rules and opened the syscall source before
running tests, or you'll see false "no detection"; (2) validate your *custom* rules and the outbound
alert path (Falcosidekick → SIEM/Slack) end-to-end — a rule that fires but isn't routed is a silent
gap.

**Tetragon** (Cilium/Isovalent, CNCF; **v1.7.x**, 2026) — eBPF runtime observability and
*enforcement* (can kill/block, not just alert). Reach for it when you want kernel-level
process/network/file visibility with low overhead and inline enforcement via `TracingPolicy` CRDs.

```bash
helm repo add cilium https://helm.cilium.io && helm repo update
helm install tetragon cilium/tetragon -n kube-system
kubectl exec -n kube-system ds/tetragon -c tetragon -- tetra getevents -o compact   # live event stream
```

Validate by applying a `TracingPolicy`, performing the action it watches, and confirming the event
(and, for enforcement policies, that the action was blocked).

## 6. Supply chain & admission

**cosign / sigstore** (cosign 2.x) — sign and verify container images and attestations.

```bash
brew install cosign
# keyless (Fulcio/Rekor) verify — require the expected signer identity:
cosign verify \
  --certificate-identity-regexp 'https://github.com/myorg/.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  myorg/app:1.2.3
cosign verify --key cosign.pub myorg/app:1.2.3   # key-based
cosign verify-attestation --type slsaprovenance ... myorg/app:1.2.3
```

Gotcha: keyless verification is meaningless without `--certificate-identity[-regexp]` and
`--certificate-oidc-issuer` — without them you only prove *something* signed it, not *who*.

**Admission enforcement** — verify the cluster actually *rejects* bad workloads, not just that
policies exist:

- **Kyverno** (CNCF; **1.17**, Feb 2026, promoted CEL-based `ImageValidatingPolicy` to v1) — native
  policy engine; `verifyImages`/`ImageValidatingPolicy` rules integrate with cosign to block
  unsigned/tampered images at admission. Set rules to `Enforce` (deny) and fail-closed.
- **OPA Gatekeeper** — structural policy via Rego/ConstraintTemplates; **cannot verify cosign
  signatures itself** — needs an external data provider (Ratify or a cosign provider).
- **sigstore policy-controller** — standalone signature-verification webhook (adds a second webhook).

Test admission by trying to apply violations and asserting they're denied:

```bash
kubectl run bad --image=nginx --dry-run=server --overrides='{"spec":{"containers":[{"name":"c","image":"nginx","securityContext":{"privileged":true}}]}}'
# expect: admission webhook denied the request
kubectl apply -f unsigned-image-pod.yaml   # expect denial from Kyverno verifyImages
```

**Pod Security Admission (PSA)** — built-in; enforces the three Pod Security Standards
(`privileged` / `baseline` / `restricted`) per namespace via labels, in `enforce`, `audit`, and
`warn` modes. Test with server-side dry-run to see what *would* be flagged before enforcing:

```bash
kubectl label ns prod \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted
# dry-run audit a namespace against a level without enforcing:
kubectl label --dry-run=server ns prod pod-security.kubernetes.io/enforce=restricted
# prove enforcement: a privileged pod should be rejected in a restricted ns
kubectl run priv --image=nginx -n prod --privileged   # expect: forbidden
```

Gotcha: don't blindly enforce `restricted` on `kube-system` — system workloads legitimately need
elevated settings; exempt or leave it at `privileged`.

**Network policy testing** — verify default-deny and that policies actually isolate traffic (a
`NetworkPolicy` is inert if the CNI doesn't enforce it):

```bash
# spot check with a throwaway pod
kubectl run probe --rm -it --image=nicolaka/netshoot -- curl -m 3 http://target-svc.other-ns
# Cilium's built-in connectivity test
cilium connectivity test
```

For exhaustive verification use **Cyclonus** (mattfenwick) — fuzzes the CNI against hundreds of
policy scenarios and reports where enforcement diverges from spec.

## 7. Secrets & exposure

Kubernetes Secrets are only **base64-encoded, not encrypted**; anyone with `get secrets` RBAC can
decode them. Scan manifests, Helm values, and CI logs (not just source) for hardcoded credentials and
`kind: Secret` leaks:

- **gitleaks** (`brew install gitleaks`; `gitleaks detect --source .`) — fast, good for pre-commit/CI.
- **trufflehog** (`brew install trufflehog`; `trufflehog filesystem ./`, `trufflehog git <repo>`) —
  slower but **verifies** found credentials against provider APIs (live/active vs. dead).
- Trivy also flags exposed secrets in images/filesystems (`trivy fs --scanners secret .`) and
  in-cluster (`exposedsecretreports` via the operator).
- Audit who can read secrets: `kubectl who-can get secrets` and prefer an external store (Vault, cloud
  secrets manager, External Secrets Operator) over raw `Secret` objects.

**etcd encryption-at-rest verification** — confirm Secrets aren't sitting in plaintext in etcd:

```bash
# 1. API server must run with the flag (no flag = no encryption at rest):
ps -ef | grep kube-apiserver | grep -- --encryption-provider-config
# 2. Read a secret's raw bytes straight from etcd and check the prefix:
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/apiserver-etcd-client.crt \
  --key=/etc/kubernetes/pki/apiserver-etcd-client.key \
  get /registry/secrets/default/my-secret | hexdump -C | head
#   encrypted → value begins k8s:enc:aescbc:v1:key1:...   plaintext → readable YAML (FAIL)
# 3. KMS-backed clusters: confirm the provider is healthy
kubectl get --raw='/readyz?verbose' | grep kms
```

Gotchas: (1) adding an encryption key does **not** re-encrypt existing Secrets — you must rewrite
them: `kubectl get secrets -A -o json | kubectl replace -f -`, then re-check etcd; (2) `aescbc`/
`secretbox` keys live in a local config file on the control plane — prefer a KMS provider so the key
isn't on disk next to the data it protects; (3) on managed clusters you can't read etcd directly —
verify via the provider's envelope-encryption/KMS settings instead.

## References

- kube-bench: https://github.com/aquasecurity/kube-bench
- CIS Kubernetes Benchmark: https://www.cisecurity.org/benchmark/kubernetes
- Trivy: https://trivy.dev/ · https://github.com/aquasecurity/trivy · Trivy Operator: https://github.com/aquasecurity/trivy-operator
- kubescape: https://github.com/kubescape/kubescape
- Polaris: https://polaris.docs.fairwinds.com/ · Checkov: https://www.checkov.io/
- kube-hunter (deprecated): https://github.com/aquasecurity/kube-hunter · kubeletctl: https://github.com/cyberark/kubeletctl
- kubectl-who-can: https://github.com/aquasecurity/kubectl-who-can · rbac-tool: https://github.com/alcideio/rbac-tool · rbac-lookup: https://github.com/FairwindsOps/rbac-lookup
- kubeaudit: https://github.com/Shopify/kubeaudit
- Falco: https://falco.org/ · event-generator: https://github.com/falcosecurity/event-generator · Tetragon: https://tetragon.io/
- cosign/sigstore: https://docs.sigstore.dev/cosign/verifying/verify/ · Kyverno image verification: https://kyverno.io/docs/writing-policies/verify-images/
- Pod Security Admission: https://kubernetes.io/docs/concepts/security/pod-security-admission/ · Encrypting data at rest: https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
- Kubelet authn/authz: https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/
- trufflehog: https://github.com/trufflesecurity/trufflehog · gitleaks: https://github.com/gitleaks/gitleaks · Cyclonus: https://github.com/mattfenwick/cyclonus
