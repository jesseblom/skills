# Resilience, chaos & disaster-recovery testing

Resilience testing proves the cluster keeps its steady state when components fail. Work in three
layers: **chaos engineering** (inject faults), **backup/DR** (prove you can rebuild), and
**HA/failover** (prove redundancy actually redundant). Always define a measurable steady-state
hypothesis first, control blast radius, and run in staging before production.

## Chaos engineering platforms

**Which to reach for:**

- **Chaos Mesh** — the default for Kubernetes-native fault injection. Richest fault set (pod,
  network, IO, stress, kernel, DNS, time, HTTP), everything is a CRD, has a dashboard and a built-in
  `Workflow` engine. Pick this for serious in-cluster experiments.
- **LitmusChaos** — pick when you want a managed experiment hub (ChaosCenter UI), a reusable
  experiment catalog (ChaosHub), GitOps/CI-driven experiments, and declarative resilience probes as
  pass/fail gates.
- **Chaos Toolkit** — pick when experiments must span beyond Kubernetes (cloud APIs, app endpoints)
  and you want a vendor-neutral JSON/YAML experiment format with an automated steady-state check +
  rollback, ideal in CI.
- **kube-monkey** — pick only for a dead-simple, opt-in, Chaos-Monkey-style random pod killer on a
  schedule. Not a platform; no network/IO/stress faults.

**Maintenance status (verified Nov 2026):** Chaos Mesh CNCF **incubating**, very active, **v2.8.8**.
LitmusChaos CNCF **incubating**, monthly releases, **v3.33.0** (Oct 2026; **3.x is not
backward-compatible with 2.x**). Chaos Toolkit **v1.21.4**. kube-monkey **recently revived** (was
dormant ~2 years, now active again, **v0.7.1**, Nov 2026 — stale Docker Hub tags make it *look*
abandoned; the repo is not).

### Chaos Mesh

```bash
helm repo add chaos-mesh https://charts.chaos-mesh.org
kubectl create ns chaos-mesh
helm install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --version 2.8.8
kubectl port-forward -n chaos-mesh svc/chaos-dashboard 2333:2333   # dashboard
```

Pod kill (`PodChaos`):

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-example
  namespace: chaos-mesh
spec:
  action: pod-kill        # also: pod-failure, container-kill
  mode: one               # one | all | fixed | fixed-percent | random-max-percent
  selector:
    namespaces: [payments]
    labelSelectors:
      app: checkout
```

Chain experiments with `Workflow` (serial/parallel steps, deadlines, conditional branches) — see the
Chaos Mesh docs for the full `Workflow` schema. Gotchas: match `chaosDaemon.runtime`/`socketPath` to
your runtime (`containerd`/`crio`/`k3s`) at install or every injection silently no-ops. `DNSChaos`
needs `--set dnsServer.create=true` at install. Namespace-scoped mode is recommended to cap blast
radius.

### LitmusChaos

`ChaosExperiment` = reusable fault definition; `ChaosEngine` = binds an experiment to a target app
and runs it; ChaosHub = catalog of prebuilt experiments; probes = declarative steady-state checks
that make an experiment pass/fail.

```bash
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
kubectl create ns litmus
helm install chaos litmuschaos/litmus -n litmus --set portal.frontend.service.type=NodePort
```

`ChaosEngine` running pod-delete:

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata: {name: checkout-chaos, namespace: payments}
spec:
  appinfo: {appns: payments, applabel: 'app=checkout', appkind: deployment}
  chaosServiceAccount: pod-delete-sa
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - {name: TOTAL_CHAOS_DURATION, value: '30'}
            - {name: CHAOS_INTERVAL, value: '10'}
            - {name: FORCE, value: 'false'}
```

`litmusctl` drives experiments from manifests in CI. Gotchas: the `chaosServiceAccount` needs RBAC
for the targeted verbs/resources or the experiment errors out. On 3.x the ChaosCenter architecture
replaced 2.x — don't mix docs/manifest versions.

### Chaos Toolkit

Vendor-neutral experiments in JSON/YAML with an automatic steady-state check before and after, plus
rollbacks.

```bash
pip install chaostoolkit chaostoolkit-kubernetes
chaos run experiment.json     # verifies steady state -> injects -> re-verifies -> rolls back
```

`experiment.json` has a `steady-state-hypothesis` (probes with tolerances) and a `method` (actions).
Gotcha: if the steady-state hypothesis fails *before* injection, the run aborts — good, it means the
system was already unhealthy.

### kube-monkey

Opt-in random pod killer, Chaos-Monkey style. Dry-run by default.

```bash
helm repo add kubemonkey https://asobti.github.io/kube-monkey/charts/repo
helm install kube-monkey kubemonkey/kube-monkey -n kube-system
```

Opt a workload in via labels (`kube-monkey/enabled: "enabled"`, `kube-monkey/mtbf`,
`kube-monkey/kill-mode`, `kube-monkey/kill-value`); nothing is touched otherwise. Gotcha: it only
runs during its configured schedule window and defaults to dry-run (`dry_run: true`) — set it false
to actually kill. Scope is pods only.

## Concrete failure experiments

| Failure | How to inject |
|---|---|
| Kill a pod | `kubectl delete pod -l app=checkout` or Chaos Mesh `PodChaos pod-kill` |
| Graceful node removal | `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` (respects PDBs) |
| Cordon only (no evict) | `kubectl cordon <node>` |
| Hard node failure | `systemctl stop kubelet` on the node, or power-off/terminate the VM/instance |
| Network latency | Chaos Mesh `NetworkChaos action: delay` |
| Network partition | Chaos Mesh `NetworkChaos action: partition` |
| DNS failure | Chaos Mesh `DNSChaos` (needs DNS server enabled) |
| CPU / memory stress | Chaos Mesh `StressChaos` |
| Disk pressure | fill a volume with `dd`/`fallocate`, or Chaos Mesh `IOChaos` for I/O latency/faults |
| Control-plane kill | move static-pod manifest aside on a control-plane node |

Latency injection:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata: {name: delay}
spec:
  action: delay
  mode: all
  selector: {namespaces: [payments], labelSelectors: {app: checkout}}
  delay: {latency: '100ms', jitter: '20ms', correlation: '50'}
```

Zone / multi-node loss (simulate a zone going down):

```bash
for n in $(kubectl get nodes -l topology.kubernetes.io/zone=us-east-1a -o name); do
  kubectl cordon "$n"; kubectl drain "$n" --ignore-daemonsets --delete-emptydir-data
done
```

Control-plane component kill / API-server unavailability (kubeadm; components are static pods). This
validates that the **data plane keeps serving while the control plane is down**:

```bash
# On a control-plane node:
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/   # kubelet stops the static pod
# verify running workloads still serve traffic; `kubectl` calls will fail (expected)
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/   # restore
```

Gotchas: node drain blocks on PodDisruptionBudgets and un-evictable pods — that is the point; don't
`--force`/`--disable-eviction` or you defeat the test. Never run control-plane or `kube-system` chaos
in production without a tight blast radius and a tested recovery path; losing the API server also
blinds your chaos controller.

## Backup & disaster-recovery testing

### Velero

Standard for backup/restore/migration of cluster resources + persistent volumes. Latest **v1.19.0**
(Nov 2026); now a CNCF project.

```bash
brew install velero      # or download the CLI release
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.14.0 \
  --bucket velero-backups \
  --secret-file ./credentials-velero \
  --use-node-agent \
  --backup-location-config region=us-east-1
```

```bash
velero backup create full-cluster --wait
velero schedule create nightly --schedule="0 1 * * *" --ttl 168h
# Restore drill — restore into a DIFFERENT namespace to test non-destructively
velero restore create drill-$(date +%s) --from-backup full-cluster \
  --namespace-mappings payments:payments-dr
velero backup get && velero restore describe <restore-name>
```

Gotchas: resource backup alone does **not** capture PV data — you need either CSI volume snapshots or
node-agent (Kopia) file-system backup (`--use-node-agent` + opt-in annotation or
`--default-volumes-to-fs-backup`). A backup you never restore is unverified; schedule a recurring
restore drill into a throwaway namespace and diff the result.

### etcd snapshot & restore

The authoritative state of the whole cluster. Take snapshots from a single healthy member.

```bash
# Backup (kubeadm cert layout; ETCDCTL_API=3 is default on etcd 3.4+)
ETCDCTL_API=3 etcdctl snapshot save /var/lib/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

etcdutl snapshot status /var/lib/etcd-snapshot.db --write-out=table   # inspect
etcdutl snapshot restore /var/lib/etcd-snapshot.db --data-dir=/var/lib/etcd-restored   # offline restore
```

Gotchas: restore is **offline** — stop the API server (and etcd) first, restore, then update the etcd
static pod manifest `hostPath` to the new data dir and restart. Modern etcd moved snapshot
`status`/`restore` to the separate `etcdutl` binary; `etcdctl snapshot restore` is deprecated. On
multi-member clusters, restore each member with matching `--initial-cluster`/
`--initial-advertise-peer-urls`.

### Full-restore drill & RTO/RPO validation

A DR plan is unproven until a timed restore succeeds on a clean cluster. Drill: (1) provision a fresh
cluster; (2) `etcdutl snapshot restore` (or stand up a clean control plane) and/or
`velero restore create --from-backup`; (3) verify steady state (apps Ready, data intact, endpoints
serving); (4) **time it.** Elapsed restore time vs target = RTO; gap between the restored
snapshot/backup timestamp and the failure time = RPO. If measured RTO/RPO exceed targets, shrink
backup interval (RPO) or pre-stage infra/automation (RTO).

## Failover & HA validation

- **Redundancy actually survives node loss:** deploy ≥2 replicas with
  `podAntiAffinity`/`topologySpreadConstraints` across `kubernetes.io/hostname` and
  `topology.kubernetes.io/zone`, then drain a node (or a whole zone) and assert zero/near-zero error
  rate and that replicas rescheduled onto surviving nodes.
- **PodDisruptionBudget:** define a PDB and confirm voluntary disruptions respect it.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: {name: checkout-pdb}
spec:
  minAvailable: 2
  selector: {matchLabels: {app: checkout}}
```

```bash
kubectl drain <node> --ignore-daemonsets   # should BLOCK rather than drop below minAvailable
kubectl get pdb checkout-pdb                # ALLOWED DISRUPTIONS column shows headroom
```

- **Leader election:** controllers/operators elect via Lease objects. Find the holder, kill it, time
  re-election:

```bash
kubectl get leases -A
kubectl -n <ns> get lease <name> -o jsonpath='{.spec.holderIdentity}{"\n"}'
kubectl delete pod <current-leader>        # watch holderIdentity change; failover should be seconds
```

Gotcha: a PDB with `minAvailable` equal to replica count makes nodes undrainable (deadlocks
maintenance); size budgets to allow at least one disruption.

## Resilience validation patterns

- **Steady-state hypothesis first.** Define a measurable "normal" and assert it before and after
  every experiment. No hypothesis = no experiment, just vandalism.
- **Blast radius control.** Start in staging; scope to one namespace/label; use `mode: one` or
  `fixed-percent` before `all`; set experiment `duration`/`deadline` so faults auto-revert; always
  know the manual kill switch (`kubectl delete <chaos-crd>`).
- **Game days.** Scheduled, hypothesis-driven group exercises: announce scope, run an escalating
  `Workflow`, watch dashboards/alerts, capture findings as action items. Verify that monitoring and
  on-call actually detect the fault.
- **Chaos in CI.** Gate releases with small, fast automated experiments (single fault, tight blast
  radius, automatic rollback: `chaos run experiment.json` or a Litmus experiment with probes); reserve
  broad multi-fault workflows for scheduled game days.

## References

- Chaos Mesh: https://chaos-mesh.org/docs/ · https://github.com/chaos-mesh/chaos-mesh/releases
- LitmusChaos: https://docs.litmuschaos.io/ · https://github.com/litmuschaos/litmus/releases
- Chaos Toolkit: https://chaostoolkit.org/ · kube-monkey: https://github.com/asobti/kube-monkey
- Velero: https://velero.io/docs/ · https://github.com/vmware-tanzu/velero/releases
- etcd backup/restore: https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/ · https://etcd.io/docs/v3.5/op-guide/recovery/
- PodDisruptionBudget: https://kubernetes.io/docs/tasks/run-application/configure-pdb/ · Principles of Chaos: https://principlesofchaos.org/
