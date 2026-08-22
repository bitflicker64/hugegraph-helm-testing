# The fix loop: report → audit → fix → review → re-test → ship

Use when a test campaign has produced findings and the goal is a chart change
maintainers can merge. Each stage exists because skipping it shipped (or
nearly shipped) a defect in a real run.

## 1. Audit the report with parallel read-only agents

Two agents, launched together, both grounded in real files (never in the
report's prose alone):

- **Upstream agent**: locate the defect in the component source, verify any
  claims about which PR added/removed a fix against actual git history (PR
  web pages hallucinate; `git log --all -- <file>` and commit diffs are the
  authority), and produce a concrete image-side change list mapped to
  file/method.
- **Chart agent**: read the actual chart and answer, in this exact shape:
  *Does the chart need to change? yes/no → MUST items (correctness /
  install-reliability) → SHOULD items (hardening/docs) → image-only items
  the chart can merely document.* Each item with file + line anchor and a
  one-line rationale.

Keep the chart/image boundary honest in everything downstream: a chart must
not silently work around an image bug; it may make its own behavior
deterministic and document the residual.

## 2. Implement as one reviewable commit per branch

- Have an agent write a **commit-ready spec** first (exact helper text,
  exact insertion anchors, values + schema additions, version bumps),
  verified against the real target files — then apply it yourself with
  direct edits (don't delegate the write; the spec-author lacks your
  session's ground truth).
- Fold every later change in with `git commit --amend` — the deliverable is
  ONE reviewable commit.
- **If the fix must land on more than one branch** (common pattern: a PR
  branch and a faster-moving test/integration branch of the same chart):
  implement fully on one branch, verify by render (see chart-engineering),
  then port — `cp` only files proven byte-identical pre-edit (`diff -q`
  each), re-apply edits to diverged files at their own anchors, and prove
  **hunk parity** with a diff-of-diffs (compare both branches' `git diff`
  outputs line-by-line, excluding version-only files). With a single target
  branch, skip all of this porting machinery.
- Scope discipline: the user/maintainer decides bundling. Defaults that
  worked: MUST + SHOULD in one commit with a bullet-per-concern message;
  follow-ups that grow the API (new value patterns, digest pinning) recorded
  but deferred.

## 3. Three-lens independent review panel

Launch three READ-ONLY reviewers in parallel over the actual commits
(`git show <sha>`), each with: full context of what the commit does, the
test evidence so far, and a demand for severity-ranked findings with
file:line and a one-line concrete fix, ending SHIP or FIX-FIRST.

- **Lens 1 — correctness + tests**: shell-script edge cases (set -eu traps,
  parse guards, quoting), template semantics (`with` on `{}`/null, nindent
  depths), helper argument order, schema-vs-preset compatibility, and
  explicitly: which workloads roll on upgrade, and what test is still
  missing before shipping.
- **Lens 2 — design + boundaries**: naming/shape consistency with the
  chart's conventions, default choices with justification, escape hatches,
  alpha-field/kubeVersion interactions, upgrade semantics, whether the
  change is the chart's business at all, branch-divergence management.
- **Lens 3 — security + maintainability**: information leaks (anything
  derived from secrets in annotations/labels/logs), injection surfaces in
  values interpolated into scripts, supply-chain posture of added images,
  removal criteria for workarounds, overclaims in the commit message.

Real catches this panel made, as calibration: an unsalted hash of secret
data in a pod annotation (lens 3, FIX-FIRST); two undocumented
upgrade-induced pod rolls, one under the chart's own documented failure mode
(lens 2, HIGH); a missing gate off-switch; free-form schema objects; stale
version prose. Expect the panel to find real things — if all three come back
empty, suspect the prompts.

**Wait for all three, then apply the union in ONE amend pass.** Editing after
each reviewer multiplies re-verification. Re-render, re-verify, amend.

## 4. Re-test the minimum live battery

Static checks never prove a distributed fix. The floor:

1. Two fresh installs — determinism oracle (zero unresolved / zero blocked
   on every PD, clean election).
2. One churn probe — confirm the documented residual still behaves as
   documented (and capture its exact lines for the report).
3. The live upgrade from the previous chart version with marker data —
   uid-diff roll oracles, durability, and **execute the documented
   remediation** to prove it converges.
4. The rotation path the checksum annotation exists for.
5. The smallest preset (values-single) once.

An isolated sandbox can independently re-run the static battery on the
patched chart: ship the change as `git format-patch` output applied onto the
public base ref inside the sandbox.

If the re-test finds a defect in the fix itself (it did: the
nslookup/search-domain issue) — fix, amend the same commit, and restart the
battery from step 1. Evidence gathered before the amend is void.

## 5. Report and ship

- Update the campaign's report/artifact in place with: commit SHAs, an
  evidence table (one row per verification, each backed by session logs),
  review outcomes including what was changed in response, and an honest
  residual list (image-side items, deferred follow-ups).
- Audit the final report row-by-row against session logs before publishing —
  remove or re-run anything unbacked.
- Push only on explicit approval from the person who owns the branches, then
  confirm the pushed ref ranges. Verify commit hygiene first: intended
  author identity, no unintended references (word-boundary grep), no stray
  files in the diff.
