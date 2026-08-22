---
name: hugegraph-helm-testing
description: >-
  Test the Apache HugeGraph Helm chart (PD + Store + Server + Hubble) like it
  will actually be operated: static/manifest checks, a real Kind install with
  auth on, claim-driven distributed-system fault scenarios with named oracles
  and evidence, upgrade-path and secret-rotation tests, and a user-journey
  pass — then, when bugs surface, drive them through a disciplined
  audit → fix → independent-review → re-test → ship loop. Use this whenever
  someone wants to test, validate, verify, CI-check, or harden the HugeGraph
  Helm chart or a HugeGraph deployment on Kubernetes/Kind; whenever PD raft,
  IpAuthHandler, allowlist, wait-for-pd-dns, or "install sometimes fails"
  comes up; whenever a chart change needs review or re-testing before a PR;
  or when someone asks "does the HugeGraph chart actually work?" — even if
  they don't say the word "test". Also trigger when someone pastes PD log
  lines like "Blocked connection from" or "Could not resolve allowlist
  entry", reports Server CrashLoopBackOff or a helm install --wait timeout
  on HugeGraph, mentions Hubble not coming up, or asks to smoke-test,
  upgrade-test, or verify secret rotation for a HugeGraph release.
---

# HugeGraph Helm chart testing

A community skill for testing the HugeGraph Helm chart the way it fails in
real life, and for landing chart fixes that survive review. It was distilled
from a full real-world cycle: a test campaign that caught an
install-bricking PD allowlist race, the chart fix for it (`wait-for-pd-dns`),
three independent reviews, and a live upgrade-path re-test. The rules below
each earned their place by catching a real bug — or by being the mistake that
hid one.

## What you need

- A checkout of the chart (in-tree at `helm/hugegraph` of a hugegraph clone,
  or standalone). Confirm `Chart.yaml` says `name: hugegraph` before testing
  anything — never test a guessed chart.
- A Docker host with ~12+ GB free memory for the full 3 PD + 3 Store +
  3 Server + Hubble topology (ten JVMs; the README's own warning about
  "silent Raft failures" under memory pressure is real). `kind`, `helm` v3,
  `kubectl`, `curl`. A 1+1+1 run via `values-single.yaml` fits far smaller
  hosts.
- Component images. Test against pinned digests and record them in every
  report — mutable tags (`helm-dev`, `latest`) make results unreproducible.

## Intake — ask before touching anything

On invocation, before any command runs, put a SHORT list of questions to the
user (compact, yes/no where possible, each with a default so "just use
defaults" is a valid answer). These answers prevent the two expensive failure
modes: acting on the wrong tree, and mutating something the user didn't
authorize.

1. **Scope** — test only, or test + fix loop if bugs are found?
   *(default: test only; fixes proposed, not applied)*
2. **Where** — this machine, or a remote host over SSH? If remote: the exact
   host alias, and confirmation nothing else critical runs there.
   *(default: local)*
3. **Chart source** — which checkout/branch? Pull/sync latest before testing,
   yes or no? *(default: test the tree exactly as it stands; record its SHA)*
4. **Push policy** — is pushing EVER allowed this session? *(default: no.
   Even "yes" means: only on an explicit "push now", never automatically)*
5. **Other remote mutations** — PR/issue comments, artifact publishing?
   *(default: none)*
6. **Cluster** — create a fresh Kind cluster (default), or use an existing
   one? Existing requires proof of ownership before any mutating command.
7. **Depth** — quick smoke, the default battery, or the full campaign
   including upgrade-path and user journey? *(default: default battery)*

Record the answers verbatim in the state file before proceeding. If the user
is unreachable mid-run, the defaults above are the standing answers — never
upgrade your own permissions because a step would be convenient.

## The state file (anti-hallucination anchor)

Maintain `<session-dir>/state.md` from the first action to the last. Its job
is to be the single source of truth the agent re-reads instead of trusting
its own memory — long campaigns outlive context windows, and a result that
is not written down did not happen.

```markdown
# Session <ts> state
## Intake answers        (verbatim, incl. push policy)
## Baseline              (chart path, branch@SHA, image digests, host, cluster name/context)
## Phase checklist       (- [ ] static  - [ ] install  - [ ] scenarios ... with status per item)
## Decisions log         (dated one-liners: what was decided and why)
## Findings index        (one line per findings/ file: id → verdict → blame)
## Next action           (exactly one)
```

Rules that make it work: update state **before** ending any work stretch;
on resume (new session, reconnect, context loss) read state FIRST and
continue from "Next action" — resume, never restart; before writing any
report, verify every claimed result against the findings index and the
session logs — if a row has no backing file, the test did not run (this
exact check once caught a phantom PASS row); when reality contradicts state
(a cluster vanished, a file changed), update state to reality and note it in
the decisions log rather than acting on the stale picture.

## Session discipline (applies to every phase)

Create a timestamped session dir (`~/.cache/hugegraph-ktest/<UTC-ts>/` with
`logs/`, `findings/`, `artifacts/`, and `state.md` above) and write findings
**as you go** — the per-scenario files are what make honest reporting
possible later. Use
timestamped, session-labelled namespaces/releases/clusters
(label `hugegraph-ktest/session=<ts>`), pin `--context`/`--kube-context` on
every command, and at teardown delete **only** what carries your session
label. Never let a test modify the chart to go green; fixes are a separate
phase with their own commit.

Verdict vocabulary: `PASS-static` / `PASS-smoke` / `PASS-hardening` /
`FAIL` (add `-reproducible` when the failure repeats on demand) /
`INCONCLUSIVE-fault-not-proven` / `INCONCLUSIVE-env` /
`INCONCLUSIVE-confounded` / `NOT-RUN`, each with a blame class
(chart / image / harness / environment). Pods-Ready, `helm test`, and one
CRUD are smoke, never a suite pass. A fault with no landing evidence is
INCONCLUSIVE, never a PASS. Declare each scenario's oracle **before**
injecting its fault — an oracle written afterwards gets fitted to whatever
happened, which is how wishful verdicts are born.

(The `ktest` in the session-dir path and label is just this skill's session
naming convention, kept short for label values.)

## The phases

Read the reference for a phase when you enter it, not before.

1. **Static / manifest layer** — lint (default + `--strict` with
   `values-cluster.yaml`), render all three presets, run the chart's
   validateValues fail-paths (each MUST fail; a pass is a bug), kubeconform
   `-strict` at the target k8s version AND the chart's `kubeVersion` floor,
   pluto, kube-score/polaris on the CLUSTER render, helm-unittest, ct lint,
   and a manual render review. → `references/test-suite.md` §Static.
2. **Install + smoke** — 3+3+3+Hubble, auth on, images pre-loaded with
   `kind load`. The install oracle is not "Ready": grep every PD's logs for
   `Could not resolve allowlist entry` and `Blocked connection` — zero of
   each, plus a clean leader election, or the install only *looks* healthy.
   → `references/test-suite.md` §Install.
3. **Distributed fault scenarios** — follower/leader crash, majority loss
   (negative tests), store durability, network partition, pod-IP churn
   (mandatory — it exercises the known allowlist bug), PVC reattach, rolling
   restart under load, auth on every replica. Each with fault command,
   landing evidence, oracle, timing budget. → `references/test-suite.md`
   §Scenarios, and `references/ipauth-bug-family.md` for what the allowlist
   bug looks like when you hit it.
4. **Upgrade path + rotation** — install the previous chart version with
   marker data, upgrade, and judge by **pod-uid diff per component** (who
   rolled, who must not have), marker durability, and block-window behavior;
   then validate the documented remediation actually converges. Rotate an
   `existingSecret` and prove the checksum annotation rolls Server.
   → `references/test-suite.md` §Upgrade.
5. **User journey** — follow the README/NOTES.txt literally as a new user
   (zero-override install, port-forwards, first CRUD, upgrade, rollback,
   uninstall, reinstall over kept PVCs). Doc commands that don't work as
   written are findings. → `references/test-suite.md` §Journey.
6. **Fix loop** (when the campaign finds chart bugs) — audit with parallel
   read-only agents (chart-fixable vs image-only; MUST vs SHOULD), implement
   one reviewable commit (amend, never stack), run a three-lens independent
   review panel, apply the union of findings in one amend pass, re-test the
   minimum live battery, and ship only on maintainer/owner approval.
   → `references/fix-loop.md` and `references/chart-engineering.md`.

## The chart's load-bearing facts

Carry these into every phase (full detail: `references/ipauth-bug-family.md`
and `references/chart-engineering.md`):

- PD resolves raft-peer DNS to IPs **once at boot** and freezes the result as
  its RPC allowlist (`IpAuthHandler`). With `podManagementPolicy: Parallel`,
  a PD can boot before peers' A-records publish → partial allowlist → if it
  blocks the elected leader, Servers CrashLoop and the install times out,
  nondeterministically. The chart-level cure is a DNS gate before PD start;
  the runtime-churn half (new pod IP blocked forever) is image-side only.
- The chart's headless Services set `publishNotReadyAddresses: true` — pod
  A-records appear at IP assignment, before readiness. This is what makes a
  peer-DNS gate deadlock-free, and it is load-bearing: verify it exists
  before trusting any bootstrap reasoning.
- Two similarly named gates, different jobs: the Store's `wait-for-pd` init
  waits for PD **quorum health** over `/v1/health` (minutes-scale, 900s
  budget); the PD's `wait-for-pd-dns` init waits only for peer **DNS
  publication** (seconds-scale, 300s budget). Don't conflate them.
- Timing budgets that separate real failures from false INCONCLUSIVEs:
  PD startup ≤300s, Store startup ≤400s (+ wait-for-pd init ≤900s), Server
  startup floor 450s, PD store-offline patrol 60s cadence, PD quorum
  = floor(replicas/2)+1. Any oracle deadline below these is self-inflicted.
- Secrets are chart-managed via `lookup` + `randAlphaNum`: plain
  `helm template` regenerates values every render — pin passwords in test
  values, never snapshot Secret data, and expect template-only (GitOps)
  renders to differ from live renders.

## Non-negotiables

- Read `references/pitfalls.md` before writing any test script — it is a
  list of tooling behaviors that silently invalidate results (busybox
  nslookup ignoring search domains, `helm list` hiding `pending-install`,
  ssh-orphaned processes, `kind load` aborting on one missing image,
  pod-uid captures polluted by terminating pods, and more).
- Report what happened, not what should have happened: never write a result
  row for a test that did not run; verify every claimed row against session
  logs before publishing. Blame environment honestly (resource contention
  and network drops mimic chart bugs).
- Image-side bugs get documented in the chart's README Limitations, never
  silently worked around in templates.
- A worked case study of this entire skill applied end-to-end, with real
  log lines and numbers: `references/evidence.md`.
