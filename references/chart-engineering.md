# Chart engineering patterns

The designs that shipped, why they have the shape they have, and the template
mechanics that keep chart changes reviewable. Verbatim final code for each is
in `evidence.md`.

## The wait-for-pd-dns gate (first-boot allowlist fix)

Design requirements that produced the final shape:

- **Gate on DNS publication, not readiness** — waiting for peer readiness
  would deadlock (peers wait on each other). Publication happens at pod-IP
  assignment because the headless Service sets
  `publishNotReadyAddresses: true`; verify that flag exists before trusting
  this design anywhere else.
- **Resolve ALL peers before releasing the pd container**, so the image's
  one-shot allowlist build can never see an unresolvable entry.
- **busybox-nslookup reality** (see pitfalls): derive the cluster domain
  from `/etc/resolv.conf`'s `search` line (`awk` for the `svc.<domain>`
  token), try short name → derived FQDN → `cluster.local` fallback. Guard
  the awk with `2>/dev/null || true` (a `set -eu` script dies on an
  unreadable resolv.conf otherwise).
- **Bounded and loud**: `waitTimeoutSeconds` (default 300 — DNS publication
  is seconds-scale; contrast the store's PD-quorum wait at 900, which waits
  on minutes-scale JVM bootstraps) with a named-peer error at timeout, so a
  broken-DNS cluster fails visibly instead of sitting `Init:0/1` forever.
- **Escape hatch**: `waitEnabled: true` — an unconditional init container in
  a community chart needs an off-switch for exotic DNS setups; disabling
  re-exposes the race, and the values comment says so.
- Mirror the sibling wait-container's conventions exactly (image value,
  securityContext inheritance, resources block, timeout naming) — symmetry
  is what makes the addition reviewable.
- Helper: derive the port-less host list from the existing peers helper
  (`regexReplaceAll ":[0-9]+" (include ...) ""` — note sprig's argument
  order is PATTERN, INPUT, REPLACEMENT; piping the input puts it in the
  wrong slot and silently yields an empty list. Render and grep the result;
  never trust a helper you haven't seen expand).

Limitation to state everywhere the gate is mentioned: it fixes first-boot
ordering only. Runtime pod-IP churn needs the image-side refresh.

## The checksum/auth annotation (secret-rotation restarts)

First design hashed the looked-up Secret **data** — review killed it: a
sha256 of secret material in a pod annotation is an unsalted credential
derivative readable by anyone with pod-read (and dictionary-attackable when
the co-hashed value is weak). The shipped design hashes Secret **names, keys,
and `metadata.resourceVersion`** — rotation still changes the annotation,
zero credential-derived material. Use sprig `dig "metadata"
"resourceVersion" "" $secret` for nil-safe access.

Honest semantics to document wherever this pattern is used (lookup-based
annotations are best-effort by construction):

- Fresh install renders a names-only hash (Secrets don't exist yet at render
  time) → the FIRST upgrade after install rolls the workload once, spuriously.
- Inline rotation (changing the value in the same upgrade) lands one upgrade
  late — render happens before apply.
- Template-only pipelines (`helm template`, ArgoCD/Flux) never see live
  Secrets: the annotation is a constant and rotation-detection is inert
  there.
- The reliable covered path: rotate an `existingSecret` out-of-band, then
  `helm upgrade` — annotation changes, workload rolls. Prove it live
  (annotation value + uid diff) before claiming it.

## Explicit StatefulSet lifecycle fields

`updateStrategy` (default `type: RollingUpdate`) and
`persistentVolumeClaimRetentionPolicy` (default Retain/Retain) as
value-driven fields wrapped in `{{- with }}` — an explicit `null` cleanly
omits the field, and `{}` coalesces to the chart default. Notes:

- `persistentVolumeClaimRetentionPolicy` is honored on K8s ≥1.27 (or the
  StatefulSetAutoDeletePVC gate); 1.23–1.26 API servers drop the field,
  which equals Retain anyway — say so in a values comment or GitOps users
  will chase phantom drift. kubeconform accepts the field even at the 1.23
  schema.
- `podManagementPolicy: Parallel` affects scale operations only — **updates
  still roll one pod at a time, reverse-ordinal**. Any pod-template change
  therefore restarts the StatefulSet's pods on upgrade; on PD this
  transiently triggers the allowlist churn face. The chart's Upgrading
  section must list every new roll a version causes, with the remediation
  and an `updateStrategy.type=OnDelete` option for maintenance windows.

## Schema discipline

`values.schema.json` uses `additionalProperties: false` per component —
every new value MUST be declared or installs break. Type properly: enums for
strategy types and Retain/Delete, `minimum` on timeouts, typed
`rollingUpdate` sub-objects (`partition` integer, `maxUnavailable`
int-or-string) rather than free-form objects — free-form is a review finding
and lets typos fail only at apply time. Name shared definitions after the
k8s field they validate so grep finds them.

## Change hygiene for a reviewable chart PR

- One commit per branch, everything folded by amend — reviewers get one
  diff, and the commit message maps bullet-per-concern to hunks.
- Verify by RENDER after every edit: `helm lint`, `helm template` piped
  through greps for each new field, schema negative tests
  (`--set x=Bogus` must fail), and determinism (two renders, identical
  annotation values).
- Version-bump semantics: patch bump per branch; a branch that ships a
  packaged `.tgz` must have it repackaged in the same commit (a stale
  archive diverging from its version number is a trap for chart consumers).
  Kill stale hardcoded version strings in prose (README "this chart is
  version X") — or better, remove the hardcode.
- Never carry branch-specific content across branches (image tags, pull
  policies, default toggles); verify per-file identity before any `cp`
  between checkouts, and afterwards prove hunk-parity by diffing the two
  branches' `git diff` outputs against each other.
