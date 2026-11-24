# Performance, scale, load & policy/config testing

Two domains: (A) performance / scale / load testing and (B) policy & configuration validation.
Commands and versions verified November 2026. Deprecations are flagged inline.

## Domain A — Performance, scale & load testing

### A1. Cluster scale testing

**ClusterLoader2 (CL2)** — the upstream Kubernetes scalability/performance framework in
`kubernetes/perf-tests`. Reach for it to validate a cluster against the official SIG-Scalability SLOs
under a defined load/density profile. It drives resource churn and measures SLIs via Prometheus.

```bash
git clone https://github.com/kubernetes/perf-tests
cd perf-tests/clusterloader2
go run cmd/clusterloader.go \
  --testconfig=testing/load/config.yaml \
  --provider=kind \
  --kubeconfig=$HOME/.kube/config \
  --nodes=10 \
  --report-dir=./results \
  --enable-prometheus-server
```

Built-in measurements: `PodStartupLatency`, `APIResponsivenessPrometheus`, `SchedulingThroughput`,
`SchedulingMetrics`, `EtcdMetrics`, `APIAvailabilityMeasurement`, `ResourceUsageSummary`. Density
tuned via `CL2_*` env vars. Gotchas: needs `--provider` matching your environment
(`gce|gke|kind|aws|kubemark|local|skeleton`); SLO-measuring tests require a reachable Prometheus (use
`--enable-prometheus-server`). Keep configs modular — the monolithic ~900-line `load/config.yaml` is
an anti-pattern.

**kube-burner** (CNCF Sandbox, v2.8.7) — lighter-weight density/churn generator that
creates/patches/deletes objects at scale, collects pod-latency measurements, and indexes Prometheus
metrics to Elasticsearch/OpenSearch/local.

```bash
go install github.com/kube-burner/kube-burner/v2@latest   # or a release binary
kube-burner init -c config.yml --uuid $(uuidgen)          # runs the workload
```

The OpenShift/cloud wrapper `kube-burner-ocp` ships prebuilt workloads
(`kube-burner-ocp cluster-density-v2 --iterations=100`, `node-density`, `node-density-heavy`).
Gotchas: indexing needs a reachable Prometheus + ES/OpenSearch endpoint; set
`measurements: [{name: podLatency}]` or you get churn without latency SLIs. Pass a stable `--uuid` to
correlate a run across the index.

**Cluster SLIs/SLOs that matter** (official SIG-Scalability thresholds, per-cluster-day): stateless
**pod startup latency p99 ≤ 5s** (creation → all containers started, excluding image pull/init);
**mutating API call p99 ≤ 1s** per (resource, verb); **read-only API call p99 ≤ 1s** at
`scope=resource`, **≤ 30s** at `scope=namespace|cluster`; prerequisite **churn ≤ 20** (pod
create/update/delete + user requests per second).

### A2. Application / HTTP load testing

**k6** (Grafana, v2.x) — the default for scripted HTTP/gRPC/WebSocket load. Reach for it for
realistic scenarios, thresholds-as-pass/fail, and CI gates.

```bash
brew install k6
k6 run --vus 50 --duration 60s script.js
k6 run -o experimental-prometheus-rw script.js   # stream metrics to Prometheus
```

**k6-operator** — run distributed/in-cluster k6 via the `TestRun` CRD for load beyond one pod:

```bash
curl https://raw.githubusercontent.com/grafana/k6-operator/main/bundle.yaml | kubectl apply -f -
kubectl create configmap my-test --from-file test.js
kubectl apply -f run-k6.yaml   # TestRun with spec.parallelism:4, spec.script.configMap
```

Gotchas: ConfigMap and TestRun must share a namespace; `parallelism` splits VUs across pods, so total
load = per-pod VUs × parallelism.

Other load tools, by use case: **Fortio** (v1.75.x; HTTP/1.1, HTTP/2, gRPC, TCP/UDP with precise QPS
control — ideal for service-mesh latency/throughput), **vegeta** (v12.13.0; constant-rate attacker,
great for latency-distribution reports in CI), **Locust** (Python-scripted programmable user
behavior; `locust-k8s-operator` for in-cluster), **hey** (lightly maintained — quick smoke checks
only, prefer k6/vegeta for anything serious).

```bash
fortio load -qps 500 -t 60s -c 50 http://myservice.default.svc/   # -grpc for gRPC
echo "GET http://target/" | vegeta attack -rate=100 -duration=30s | vegeta report
hey -z 30s -c 50 http://target/
```

### A3. Autoscaling validation

**HPA** — deploy a CPU-burnable workload, set a target, drive load, watch replicas climb then settle.

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
kubectl run -it load --rm --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
kubectl get hpa php-apache --watch
```

Gotcha: HPA needs metrics-server (or a custom/external metrics adapter) installed, and resource
`requests` set on the pods, or it reports `<unknown>` and never scales.

**KEDA** (CNCF graduated, v2.21.x) — event-driven scaling (queues, Kafka, Prometheus, cron). Validate
by pushing events to the source and watching the `ScaledObject` drive the Deployment, including
scale-to-zero (`kubectl get scaledobject,hpa -w`; KEDA creates a managed HPA under the hood).

**Cluster Autoscaler / Karpenter** (Karpenter now `kubernetes-sigs/karpenter`, v1.15.x) — validate
node scale-up by creating pending pods that exceed current capacity:

```bash
kubectl create deployment inflate --image=public.ecr.aws/eks-distro/kubernetes/pause:3.9
kubectl set resources deployment inflate --requests=cpu=1
kubectl scale deployment inflate --replicas=50
kubectl get nodes -w          # watch new nodes join (Karpenter: kubectl get nodeclaims)
```

Gotcha: scale-up only triggers on `Pending` pods that are actually unschedulable; set real
`requests`. Consolidation/scale-down is slow by design — don't assert on it with short timeouts.

**VPA** — install VPA, create a `VerticalPodAutoscaler` (`updateMode: "Off"` for recommendations-only),
then `kubectl describe vpa my-vpa`. Gotcha: `Auto`/`Recreate` mode evicts pods to resize them; never
combine VPA-on-CPU with HPA-on-CPU for the same workload.

### A4. Resource & node benchmarking

**Networking throughput** — iperf3 between two pods (or across nodes with anti-affinity /
`hostNetwork` for node-to-node):

```bash
kubectl run iperf-server --image=networkstatic/iperf3 -- -s
kubectl run iperf-client --rm -it --image=networkstatic/iperf3 -- \
  -c $(kubectl get pod iperf-server -o jsonpath='{.status.podIP}')
```

**Storage benchmarking — kubestr + fio** (`kastenhq/kubestr`) — benchmark a StorageClass and validate
CSI snapshot/restore:

```bash
kubestr fio -s <storageclass> -z 10Gi            # throughput/IOPS/latency via fio
kubestr csicheck -s <storageclass> -v <volumesnapshotclass>
```

Gotchas: results depend on PV zone/node placement — pin the test pod to the node under test;
`csicheck` requires a working VolumeSnapshotClass and the external-snapshotter CRDs.

Node/kubelet capacity: validate `requests`/`limits` realism with CL2 `node-density` / kube-burner
`node-density-heavy`; check allocatable headroom with `kubectl describe node` and `kubectl top nodes`
(needs metrics-server).

## Domain B — Policy & configuration validation

### B5. Manifest / schema validation (pre-deploy, CI)

**kubeconform** (v0.8.0) — fast, offline, parallel schema validation. This is the maintained successor
to **kubeval**, which is **deprecated and unmaintained — do not use it; migrate to kubeconform.**

```bash
brew install kubeconform   # or: go install github.com/yannh/kubeconform/cmd/kubeconform@latest
kubeconform -summary -strict manifests/
# with CRD schemas (Datree CRDs-catalog):
kubeconform -summary -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  manifests/
```

Gotchas: default schemas only cover built-in APIs — add a `-schema-location` for every CRD or it skips
them; pin `-kubernetes-version` to match the target cluster.

**Server-side dry-run / validation** — catch admission + schema errors against the live API without
persisting:

```bash
kubectl apply -f manifest.yaml --dry-run=server --validate=true
```

Gotcha: `--dry-run=server` runs mutating/validating webhooks, so it needs cluster access and real
RBAC; `--dry-run=client` only checks local schema.

### B6. Policy-as-code admission & testing

**OPA/Gatekeeper** — Rego policies as `ConstraintTemplate` + `Constraint` admission objects.
Unit-test them offline with **gator** (`gator verify` for Suite unit tests — renamed from the old
`gator test`; `gator test` now just evaluates manifests against templates/constraints).

```bash
helm install gatekeeper gatekeeper/gatekeeper -n gatekeeper-system --create-namespace
brew install gator
gator test --filename=templates-and-constraints/     # eval manifests, exit code = violations
gator verify ./...                                   # run Suite unit tests
```

Gotcha: suite/case paths are relative to the suite file (absolute paths rejected); `gator` needs
Gatekeeper v3.16.0+ for `go install`.

**Kyverno** — YAML-native policies (no Rego). Unit-test with `kyverno test` and dry-run with
`kyverno apply` in CI.

```bash
helm install kyverno kyverno/kyverno -n kyverno --create-namespace   # in-cluster
kubectl krew install kyverno                                          # the CLI plugin
kyverno test .                                        # runs kyverno-test.yaml cases
kyverno apply policy.yaml --resource resource.yaml    # dry-run a policy off-cluster
```

Gotcha: some CLI subcommands need `KYVERNO_EXPERIMENTAL=true`; `kyverno apply` evaluates policies
locally and won't catch cluster-context lookups (API calls/config maps) the same as live admission.

**Chainsaw** (Kyverno) — declarative YAML end-to-end tests: apply resources, assert the
controller/policy reconciles or rejects them (supports negative tests via `error`/expected-failure
steps). Reach for it for policy/operator e2e against a real (or kind) cluster. Gotcha: it mutates a
real cluster — point it at a throwaway/kind cluster. (Same tool as the functional-testing reference.)

**Conftest** (OPA, Rego against any config) — reach for it to gate arbitrary YAML/JSON (incl. non-k8s:
Helm output, Terraform, Dockerfiles) in CI.

```bash
brew install conftest
conftest test deployment.yaml          # looks for deny/warn rules in ./policy
helm template . | conftest test -      # gate rendered Helm charts
```

### B7. Best-practice / linting scanners

- **kube-score** (v1.20.0) — reliability/security recommendations: `kube-score score deployment.yaml`.
- **Polaris** (Fairwinds, v10.2.0) — 30+ best-practice checks; CLI, dashboard, or admission controller:
  `polaris audit --audit-path ./manifests --format pretty`. Gotcha: images moved to
  `us-docker.pkg.dev/fairwinds-ops/oss/polaris` as of v10.2.0 — pin a full version (floating tags on
  `quay.io` no longer published).
- **KubeLinter** (StackRox, v0.7.6) — static checks on manifests and Helm charts:
  `kube-linter lint ./manifests/`.
- **datree — DEPRECATED / SUNSET.** Repo marked `[DEPRECATED]`, hosted service shut down (acquired by
  Sysdig). **Do not adopt it.** Use kubeconform (schema), Polaris/kube-score/KubeLinter
  (best-practice), or Kyverno/Gatekeeper (policy). Its CRDs-catalog JSON-schema repo lives on and is
  still a handy `-schema-location` source for kubeconform.
- **config-lint** (Stelligent) — niche, Terraform-oriented; prefer the above for Kubernetes.

### B8. Drift & GitOps testing

```bash
kubectl diff -f manifest.yaml     # compare local vs live (server-side; nonzero exit = drift)
argocd app diff <app>             # Argo CD: UI shows OutOfSync status
flux diff kustomization <name> --path ./clusters/prod   # Flux dry-run/diff before reconcile
```

Gotcha: `diff` exit code is nonzero on any difference — useful as a CI drift gate; configure
ignore-differences for fields other controllers mutate. `flux diff` renders locally and needs the
same build inputs (overlays, substitutions) Flux uses in-cluster.

## References

- ClusterLoader2: https://github.com/kubernetes/perf-tests/tree/master/clusterloader2 · SLIs/SLOs: https://github.com/kubernetes/community/blob/master/sig-scalability/slos/slos.md
- kube-burner: https://kube-burner.github.io/kube-burner/ · k6: https://grafana.com/docs/k6/latest/ · k6-operator: https://github.com/grafana/k6-operator
- Fortio: https://github.com/fortio/fortio · vegeta: https://github.com/tsenart/vegeta · Locust: https://locust.io · hey: https://github.com/rakyll/hey
- KEDA: https://keda.sh · Karpenter: https://karpenter.sh · kubestr: https://github.com/kastenhq/kubestr
- kubeconform: https://github.com/yannh/kubeconform (kubeval DEPRECATED: https://github.com/instrumenta/kubeval)
- Gatekeeper gator: https://open-policy-agent.github.io/gatekeeper/website/docs/gator/ · Kyverno CLI: https://kyverno.io/docs/kyverno-cli/ · Conftest: https://www.conftest.dev/
- kube-score: https://github.com/zegl/kube-score · Polaris: https://github.com/FairwindsOps/polaris · KubeLinter: https://github.com/stackrox/kube-linter · datree (DEPRECATED): https://github.com/datreeio/datree
- Argo CD: https://argo-cd.readthedocs.io/ · Flux diff: https://fluxcd.io/flux/cmd/flux_diff_kustomization/
