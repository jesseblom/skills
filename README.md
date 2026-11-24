# Skills

A collection of [Claude Code](https://claude.com/claude-code) skills.

## Skills in this repo

### [kubernetes-cluster-testing](kubernetes-cluster-testing/)

Pick the right way to test a Kubernetes cluster and get runnable, command-level workflows for it.
Covers five domains — does it work (conformance, functional/e2e, health & smoke), is it secure
(CIS, RBAC, runtime, supply chain, secrets), does it survive failure (chaos, backup/DR, HA),
does it scale (scale/load/autoscaling), and is its config valid (schema, policy-as-code, drift) —
with a decision table that routes each question to the right tool (Sonobuoy, kube-bench, Trivy,
kubescape, Falco, Chaos Mesh, Velero, k6, ClusterLoader2, kubeconform, Kyverno, Gatekeeper, and more).

> Tool names, versions, and commands were verified **2026-11-24** against official docs and release
> pages (Kubernetes **v1.36** line). Versions drift — pin the current release of each tool before
> use. Deprecations are flagged inline.

## Installing a skill

**Personal (all your projects):**

```bash
git clone https://github.com/jesseblom/skills.git /tmp/jesseblom-skills
cp -R /tmp/jesseblom-skills/kubernetes-cluster-testing ~/.claude/skills/
```

**Team (shared via a project repo):** copy the skill folder into that project's
`.claude/skills/` instead of `~/.claude/skills/`.

Restart Claude Code; the skill is discovered at startup. Claude loads it automatically when a
request matches its description, or you can invoke it explicitly with `/kubernetes-cluster-testing`.

## License

[MIT](LICENSE)
