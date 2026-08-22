# hugegraph-helm-testing

An agent skill for testing the [Apache HugeGraph](https://hugegraph.apache.org/)
Helm chart the way it fails in production — and for landing chart fixes that
survive independent review.

Distilled from a complete real-world campaign: static/manifest checks, Kind
installs with auth on, claim-driven distributed fault scenarios, the
discovery of an install-bricking PD allowlist race, the `wait-for-pd-dns`
chart fix, a three-lens review panel, and a live upgrade-path re-test.

## What's inside

```
hugegraph-helm-testing/
├── SKILL.md                          # entry point: phases, ground truth, hard rules
└── references/
    ├── test-suite.md                 # the full battery: static → install → faults → upgrade → journey
    ├── ipauth-bug-family.md          # the PD allowlist bug: mechanics, signatures, remediation
    ├── chart-engineering.md          # fix patterns: DNS gate, resourceVersion checksums, schema discipline
    ├── fix-loop.md                   # report → audit → fix → review → re-test → ship
    ├── pitfalls.md                   # tooling behaviors that silently invalidate results
    └── evidence.md                   # the worked case study, with real log lines and shipped code
```

## Install

**Claude Code** (or any Agent Skills runtime) — clone into your skills
directory:

```bash
git clone https://github.com/bitflicker64/hugegraph-helm-testing ~/.claude/skills/hugegraph-helm-testing
```

**Codex** (or any AGENTS.md-reading agent) — clone it into your workspace
(e.g. next to your hugegraph checkout); the bundled `AGENTS.md` routes the
agent into `SKILL.md`:

```bash
git clone https://github.com/bitflicker64/hugegraph-helm-testing
```

You can also point your project's own `AGENTS.md` at
`hugegraph-helm-testing/SKILL.md` ("for HugeGraph Helm testing, follow this
file").

Then ask your agent to "test the HugeGraph helm chart" (or anything close);
the skill self-describes when it should trigger. On invocation it starts
with a short intake (push allowed? local or remote? which branch? how deep?)
and keeps a session state file so long campaigns stay grounded in recorded
fact rather than model memory.

## Requirements

- `kind`, `helm` v3, `kubectl`, `curl`, Docker with ~12 GB free memory for
  the full 3 PD + 3 Store + 3 Server + Hubble topology (a 1+1+1 run via
  `values-single.yaml` fits far smaller hosts)
- A checkout of the chart (`helm/hugegraph` in a hugegraph clone, or a
  standalone chart directory)

## Principles

- Verdicts carry evidence and blame (chart / image / harness / environment);
  a fault with no landing evidence is INCONCLUSIVE, never a PASS.
- Pods-Ready, `helm test`, and one CRUD are smoke, never a suite pass — the
  install oracle is zero allowlist errors on every PD plus a clean election.
- Tests never modify the chart; fixes are their own phase, one reviewable
  commit, independently reviewed, re-tested live before shipping.
- Report what happened, not what should have happened.

## License

Apache-2.0, same as HugeGraph.
