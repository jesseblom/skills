# Functional, conformance & health testing

Scope: proving a Kubernetes cluster is *correct* (conformance), that *workloads behave* on it
(functional/e2e), and that it is *healthy and usable* (validation/smoke). Current upstream line as
of Nov 2026 is **Kubernetes v1.36** (released 2026-04-22); match tool and image versions to the
cluster's minor version.

## 1. Cluster conformance testing

"Certified Kubernetes" means a cluster passes the CNCF-defined **Conformance** test set — the
subset of the upstream e2e suite tagged `[Conformance]` that every compliant distribution must pass
identically. It proves the API behaves per spec (portability), *not* that the cluster is healthy,
secure, or performant. Run conformance when: standing up a new distro/provider, submitting for CNCF
certification, or validating a major version upgrade. Do **not** run it as a routine health check —
it is slow (full run ~1–2h) and `certified-conformance` mode can run disruptive tests, so avoid it
on production.

**Sonobuoy** — the canonical CNCF tool for running the conformance suite in-cluster (it schedules
the upstream e2e image as a pod and aggregates results). Reach for it for any certification or
portability run.

```bash
# install (macOS/Linux)
brew install sonobuoy            # or download from github.com/vmware-tanzu/sonobuoy/releases (v0.57.5)

# full certified-conformance run (required mode for CNCF submission), block until done
sonobuoy run --mode=certified-conformance --wait
sonobuoy status                                  # per-plugin progress
results=$(sonobuoy retrieve)                      # pull the result tarball
sonobuoy results "$results"                       # summarize pass/fail
sonobuoy delete --wait                            # tear down (also: --all for leaked e2e namespaces)
```

Modes: `quick` (one smoke test, ~minutes — use to validate Sonobuoy itself),
`non-disruptive-conformance` (**default**; conformance minus disruptive tests, safe-ish on live
clusters), `certified-conformance` (full set, may disrupt). Target a focused subset with
`--e2e-focus` / `--e2e-skip` (Go regex), and pin the suite image to the cluster version with
`--kube-conformance-image registry.k8s.io/conformance:v1.36.0`.

Gotchas: (1) For a **valid CNCF certification** run you must use `--mode=certified-conformance`,
skip *nothing* (`E2E_SKIP` empty), and focus exactly `[Conformance]` — any skip invalidates it.
(2) The conformance image minor version must match the cluster, or many tests fail spuriously; in
air-gapped clusters mirror images and pass `--e2e-repo-config`. (3) Needs cluster-admin and enough
scheduling headroom for the e2e namespaces it creates.

**Upstream e2e directly (`kubetest2` / `e2e.test`)** — only reach for this when you need raw
control over the Ginkgo suite outside Sonobuoy (e.g. CI for kubernetes/kubernetes itself, or
non-conformance e2e). Install `kubetest2` from `k8s.io/kubetest2`; most users should prefer
Sonobuoy, which wraps exactly this. The `[Conformance]` tag is the same selector either way.

## 2. Functional / end-to-end testing of workloads on the cluster

**Chainsaw (`kyverno-chainsaw`)** — the modern, actively developed declarative e2e tool and the
de-facto successor to KUTTL. CNCF project (Kyverno subproject, graduated 2026-03); latest
**v0.2.15**. Reach for it to test operators/CRDs and any "apply manifests → assert cluster state"
workflow, locally or in CI.

```bash
brew install kyverno-chainsaw                     # or: go install github.com/kyverno/chainsaw@latest
chainsaw test                                     # discovers tests recursively from cwd
chainsaw test ./tests --test-dir ./more           # target specific folders
```

A test is a `chainsaw-test.yaml` with steps containing `try`/`catch`/`finally`/`cleanup` blocks of
operations (`apply`, `assert`, `error`, `script`, `delete`):

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Test
metadata:
  name: quick-start
spec:
  steps:
  - try:
    - apply:   { file: configmap.yaml }   # create resource
    - assert:  { file: configmap.yaml }    # poll until cluster matches (partial match + JMESPath)
    - error:   { file: should-not-exist.yaml }  # fail if this ever matches
```

Migrate existing KUTTL suites: `chainsaw migrate kuttl config` and
`chainsaw migrate kuttl tests ./path`. Gotchas: (1) `assert` does a *subset* match and retries
until timeout, so a too-loose assertion passes silently — assert the specific fields that matter.
(2) Each test gets an ephemeral namespace by default; resources needing a fixed namespace must set
it explicitly.

**KUTTL (`kubectl kuttl test`, kudobuilder/kuttl)** — the original YAML assert/errors harness
(`00-install.yaml`, `01-assert.yaml`, `errors.yaml` naming convention). Still tagged (v0.26.0) but
low-activity; the Kyverno KUTTL fork itself now redirects users to Chainsaw. Only reach for it to
maintain existing KUTTL suites — **choose Chainsaw for anything new.**

```bash
kubectl krew install kuttl
kubectl kuttl test ./tests --start-kind           # or against current kubeconfig context
```

**`helm test`** — validates a *released* chart by running test Pods/Jobs the chart ships. Use right
after `helm install`/`upgrade` to confirm the release actually works (e.g. the service answers).
(Helm is now v4; behavior unchanged.)

```bash
helm install myrel ./mychart
helm test myrel                                   # runs pods annotated helm.sh/hook: test
```

Put test manifests under `templates/tests/`, annotate `helm.sh/hook: test`; a container exit 0 =
pass. Gotcha: the test pod lingers unless you set `helm.sh/hook-delete-policy: hook-succeeded`; and
`helm test` only exercises what the chart author wrote — absence of tests is not a pass.

**bats + kubectl** — lightweight shell assertions for black-box acceptance checks when YAML
harnesses are overkill. Install `bats-core` (`brew install bats-core`). Pattern:

```bash
@test "nginx deployment becomes available" {
  kubectl apply -f nginx.yaml
  run kubectl wait --for=condition=available --timeout=120s deploy/nginx
  [ "$status" -eq 0 ]
}
```

Gotcha: bats has no built-in polling — always wrap kubectl in `kubectl wait`/retry loops or tests
flake on timing.

**Ginkgo/Gomega + envtest (operators/CRDs)** — the Go-native BDD stack upstream and
controller-runtime use. Reach for it when testing a controller's reconcile logic in Go.
`setup-envtest` runs a real `kube-apiserver`+`etcd` with no kubelet (fast, no nodes); use a
real/kind cluster when you need pods to actually run.

```bash
go install github.com/onsi/ginkgo/v2/ginkgo@latest
go install sigs.k8s.io/controller-runtime/tools/setup-envtest@latest
export KUBEBUILDER_ASSETS=$(setup-envtest use -p path 1.36.x)
ginkgo -r ./...                                   # or: go test ./...
```

Gotcha: envtest has **no scheduler or kubelet** — Pods never reach Running and nodes don't exist,
so it validates API/controller behavior only, not real workload execution.

## 3. Cluster health & readiness validation

**Fast kubectl smoke checks** (no extra tooling):

```bash
kubectl get nodes -o wide                          # every node should be Ready
kubectl get --raw='/readyz?verbose'                # apiserver ready + per-check breakdown
kubectl get --raw='/livez?verbose'                 # apiserver liveness (restart-if-failing)
kubectl get --raw='/readyz?verbose&exclude=etcd'   # exclude a specific check
kubectl -n kube-system get pods                     # control-plane + CoreDNS pods Running
kubectl -n kube-system rollout status ds/kube-proxy
kubectl rollout status deploy/<name> -n <ns>        # workload rollout health
kubectl get events -A --sort-by=.lastTimestamp | tail
```

Version-sensitive caveats: `/healthz` is **deprecated since v1.16** — use `/livez` (liveness) and
`/readyz` (readiness) instead. `kubectl get componentstatuses` (`cs`) is **deprecated since v1.19**
and unreliable (it assumes control-plane components sit at fixed local addresses, false for
managed/HA clusters) — it still exists in v1.36 but prefer `/livez`+`/readyz` and checking
etcd/nodes directly. Machines should trust the HTTP status code, not the verbose text body.

**etcd health** — check directly from the etcd pod/host (datastore health isn't fully covered by
apiserver readyz):

```bash
kubectl -n kube-system exec etcd-<control-plane-node> -- etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health --cluster
```

Gotcha: on managed control planes (EKS/GKE/AKS) you cannot reach etcd — rely on provider health +
`/readyz`.

**Popeye** — read-only cluster *sanitizer*: scans live resources and scores misconfigurations
(dangling services, missing resource limits, unused configmaps, RBAC smells). Reach for it as a
post-deploy hygiene audit. Latest **v0.22.1** (release cadence has slowed; `derailed/popeye` is
upstream, an `undistro/popeye` fork also circulates).

```bash
brew install derailed/popeye/popeye    # or: go install github.com/derailed/popeye@latest
popeye                                  # scans current context; add -A for all namespaces
popeye -o html --save                   # export a report
```

Gotcha: it never mutates anything (safe to run anywhere); scores are opinionated — treat as
advisory, not gate.

**kube-score** — static analysis of manifests (not the live cluster) for reliability/security best
practices (probes, limits, anti-affinity, `runAsNonRoot`). Use in CI before apply. Latest
**v1.20.0**.

```bash
brew install kube-score                 # or: kubectl krew install score
kube-score score deployment.yaml
kubectl get deploy -o yaml | kube-score score -   # score what's live
```

Gotcha: it reads YAML only — pipe from `kubectl get -o yaml` to score running objects; exit code is
non-zero on findings, good for CI gating.

**kubectl-neat** — strips server-injected noise (managedFields, status, default annotations) from
`get -o yaml` to make diffing/inspection sane during validation.

```bash
kubectl krew install neat
kubectl get pod mypod -o yaml | kubectl neat
```

**k9s** — terminal UI for fast interactive triage (pods, logs, events, resource usage) when
eyeballing a cluster's state. Latest **v0.51.0**.

```bash
brew install derailed/k9s/k9s           # or: kubectl krew ... / download release
k9s
```

**Node Problem Detector (NPD)** — DaemonSet that watches each node for kernel/daemon/hardware
faults and surfaces them as Node Conditions and Events (e.g. `KernelDeadlock`, `DiskPressure`-style
custom monitors) so a scheduler/remediator can react. Reach for it for ongoing node-level health,
not one-shot validation. Latest **v1.36.0** (project moved v0→v1 to signal stability).

```bash
helm repo add deliveryhero https://charts.deliveryhero.io/
helm install npd deliveryhero/node-problem-detector -n kube-system
kubectl get nodes -o json | jq '.items[].status.conditions'   # see NPD-reported conditions
```

Gotcha: NPD only *reports* — it doesn't remediate; pair with Draino/descheduler or a cloud
auto-repair for action.

## 4. Smoke / acceptance testing after provisioning or upgrade

Minimal "is the cluster actually usable" checklist — run top-to-bottom; each step exercises a
distinct subsystem. Fail fast on the first that hangs.

```bash
# 0. control plane + nodes healthy
kubectl get --raw='/readyz?verbose' ; kubectl get nodes

# 1. scheduling + image pull: deploy a pod
kubectl create deployment smoke --image=nginx:stable --replicas=2
kubectl rollout status deploy/smoke --timeout=120s

# 2. cross-node pod networking (schedule onto 2 nodes, curl pod-to-pod)
kubectl get pods -l app=smoke -o wide              # confirm spread across nodes
kubectl run neta --image=nicolaka/netshoot -it --rm -- curl -s <other-pod-ip>

# 3. Service + cluster DNS (CoreDNS)
kubectl expose deployment smoke --port=80
kubectl run dns --image=nicolaka/netshoot -it --rm -- nslookup smoke.default.svc.cluster.local
kubectl run dns --image=nicolaka/netshoot -it --rm -- curl -s http://smoke.default

# 4. persistent storage: PVC binds + mounts
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: smoke-pvc }
spec: { accessModes: [ReadWriteOnce], resources: { requests: { storage: 1Gi } } }
EOF
kubectl get pvc smoke-pvc                           # should reach Bound (checks default StorageClass + provisioner)

# 5. LoadBalancer / Ingress (environment-dependent)
kubectl patch svc smoke -p '{"spec":{"type":"LoadBalancer"}}'
kubectl get svc smoke -w                            # EXTERNAL-IP should populate (cloud LB / MetalLB)
# ...apply an Ingress and curl the host if an ingress controller is installed

# cleanup
kubectl delete deploy/smoke svc/smoke pvc/smoke-pvc
```

What each catches: (1) scheduler+kubelet+registry, (2) CNI across nodes, (3) kube-proxy/Service +
CoreDNS, (4) CSI/provisioner + default StorageClass, (5) cloud-controller-manager / LB + ingress.
Gotchas: PVC stuck `Pending` almost always means no default StorageClass or no provisioner;
LoadBalancer stuck `<pending>` means no cloud LB integration or MetalLB/other LB controller. For a
repeatable, assertable version of this list, encode it as a Chainsaw test or `helm test` rather than
ad-hoc shell.

## References

- Sonobuoy docs & releases — https://sonobuoy.io/docs/main/ , https://github.com/vmware-tanzu/sonobuoy/releases
- CNCF conformance submission instructions — https://github.com/cncf/k8s-conformance/blob/master/instructions.md
- Kyverno Chainsaw — https://kyverno.github.io/chainsaw/ , https://github.com/kyverno/chainsaw/releases
- KUTTL — https://kuttl.dev/ , https://github.com/kudobuilder/kuttl
- Helm chart tests / `helm test` — https://helm.sh/docs/topics/chart_tests/ , https://helm.sh/docs/helm/helm_test/
- Kubernetes API health endpoints (livez/readyz) — https://kubernetes.io/docs/reference/using-api/health-checks/
- ComponentStatus (deprecated) — https://kubernetes.io/docs/reference/kubernetes-api/core/component-status-v1/
- Ginkgo — https://onsi.github.io/ginkgo/ ; controller-runtime envtest — https://book.kubebuilder.io/reference/envtest.html
- Popeye — https://github.com/derailed/popeye
- kube-score — https://github.com/zegl/kube-score
- k9s — https://k9scli.io/ ; kubectl-neat — https://github.com/itaysk/kubectl-neat
- Node Problem Detector — https://github.com/kubernetes/node-problem-detector
