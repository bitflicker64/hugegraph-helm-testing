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

**Created by Himanshu Verma / https://github.com/bitflicker64**

An open-source skill for testing the HugeGraph Helm chart the way it fails in
real life, and for landing chart fixes that survive review. Distilled from six
test campaigns against the chart in apache/hugegraph#3132 between August and
September 2026.

**Licence:** Apache License 2.0, the same as HugeGraph; share and adapt with
attribution (see LICENSE).

**Feedback & Support:** if the method is wrong or missing a case, open an
issue at https://github.com/bitflicker64/hugegraph-helm-testing so every
user benefits. If the agent did not follow a rule written here, acknowledge
and correct rather than filing.

It was distilled from a full real-world cycle: a test campaign that caught an
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
  When the branch under test has no published images, build them yourself
  from its tree with the revision label stamped in (recipe in
  `references/test-suite.md` §Install, "Building the images yourself") and
  tag the source tree so a report or PR comment can cite it.

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

Every claim in a finding or report carries an **evidence class**:
`measured` (the command and its output are in the session logs), `derived`
(reasoned from named source lines, cite them), or `carried` (copied from an
earlier report, with that report's class). A negative claim — "rejects
every credential", "cannot be built", "never expires" — is only ever
`measured`: run it against the real inputs, including the empty password
and the missing header, before it can appear as a finding. A `derived`
number (a timeout, a lease, a budget) stays labelled derived until a
campaign measures it. One campaign carried "PD REST rejects every
credential" through three reports on source reading alone; five curls
showed any password works for four fixed usernames, and two recommendations
had been built on the false claim.

(The `ktest` in the session-dir path and label is just this skill's session
naming convention, kept short for label values.)

## The phases

Read the reference for a phase when you enter it, not before.

1. **Static / manifest layer** — lint (default + `--strict` with
   `values-cluster.yaml`), render all presets, run the chart's
   validateValues fail-paths (each MUST fail; a pass is a bug), kubeconform
   `-strict` at the target k8s version AND the chart's `kubeVersion` floor,
   pluto, kube-score/polaris on the CLUSTER render, helm-unittest, ct lint,
   and a manual render review. Iterate presets with explicit invocations,
   never a shell variable holding several flags; a render of 0 objects or a
   lint error naming a file with a leading space is a harness fault, re-run
   before concluding anything about the chart. → `references/test-suite.md`
   §Static.
2. **Install + smoke** — 3+3+3+Hubble, auth on, images pre-loaded with
   `kind load`. The install oracle is not "Ready": grep every PD's logs for
   `Could not resolve allowlist entry` and `Blocked connection` — zero of
   each, plus a clean leader election, or the install only *looks* healthy.
   → `references/test-suite.md` §Install.
3. **Distributed fault scenarios** — follower/leader crash, majority loss
   (negative tests), store durability, network partition, pod-IP churn
   (mandatory — it exercises the known allowlist bug), PVC reattach, rolling
   restart under load, auth on every replica, and the Server discovery lease
   (three registry rows, Hubble shows three, a replaced Pod's row expires
   within the lease). Each with fault command, landing evidence, oracle,
   timing budget. The table is a starting set, not the definition of
   coverage: generate further claims from the taxonomy in
   `references/test-suite.md` §Claim taxonomy and report coverage against
   the taxonomy, not against the list. → `references/test-suite.md`
   §Scenarios, and `references/ipauth-bug-family.md` for what the allowlist
   bug looks like when you hit it.
4. **Upgrade path + rotation** — install the previous chart version with
   marker data, upgrade, and judge by **pod-uid diff per component** (who
   rolled, who must not have), marker durability, and block-window behavior;
   then validate the documented remediation actually converges. Rotate an
   `existingSecret` and prove the checksum annotation rolls Server. When the
   question is an image-side fix, run the two-stage variant: install on
   images without the fix, `helm upgrade` to images with it, and take the
   negative control, the fix and the image-roll path from one cluster.
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
  waits until a **majority of PD peers answer `store.waitPath`** (default
  `/v1/health`; minutes-scale, 900s budget); the PD's `wait-for-pd-dns` init
  waits only for peer **DNS publication** (seconds-scale, 300s budget).
  Don't conflate them, and don't call the first one a quorum wait:
  `/v1/health` answers 200 as soon as PD's listener is up and never consults
  raft, so the count is of listeners. PD images from 1.8.0 add `/v1/ready`
  (503 without a raft leader); on those, `pd.readinessPath` and
  `store.waitPath` set to `/v1/ready` make readiness and the gate mean quorum.
- Server discovery is a lease: each Server re-registers its announced URL
  with PD every 15s and PD drops the entry after three missed heartbeats
  (45s; measured 30 to 35s). The registry key is app name / version /
  address, so per-Pod-IP announcement yields one row per replica and a
  shared Service URL collapses all replicas into one row.
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
  logs before publishing, and confirm every claim row carries its evidence
  class. Blame environment honestly (resource contention and network drops
  mimic chart bugs).
- An oracle endpoint is named from the handler that serves it, not from the
  shape of its URL: before a claim block is written, read the REST class
  (path prefix, auth exclusion list, response shape) and cite it. The PD
  endpoint table in `references/test-suite.md` §Endpoints was built that
  way; extend it the same way. One report told readers to count Server
  registrations via `/v1/members`, which lists PD raft members. The same
  rule covers metrics: a gauge is interpreted from its registration (the
  lambda and description that emit it), not from its value. NaN, -1 and 0
  are usually documented sentinels; `hg_raft_alive_peers` is NaN on every
  non-leader by design, and one campaign nearly reported that as a defect.
- Pin in the session values exactly the image tags (or digests) you loaded
  onto the nodes. The chart default plus `IfNotPresent` pulls whatever the
  registry holds under that tag on every node that lacks it, and the run
  silently tests a different build. Read the identity back from the running
  pod, never from the build host.
- Before injecting a fault, run every jsonpath, jq and grep the scenario
  script relies on against the live cluster and against one real sampler
  line, and check they return a value. A parser that cannot match returns
  emptiness, and emptiness reads like a quiet cluster.
- Image-side bugs get documented in the chart's README Limitations, never
  silently worked around in templates.
- A worked case study of this entire skill applied end-to-end, with real
  log lines and numbers: `references/evidence.md`.
