# The PD IpAuthHandler allowlist bug family

The single most load-bearing piece of HugeGraph-on-K8s knowledge. One root
cause, several faces. Source: `IpAuthHandler` in
`hugegraph-pd/hg-pd-core/src/main/java/org/apache/hugegraph/pd/raft/auth/`.

## Root cause

PD protects its raft RPC port with an IP allowlist built by resolving the
configured peer hostnames (StatefulSet headless DNS names) to IP addresses
**once, at process start**. There is no periodic or event-driven
re-resolution; the class comment itself admits DNS-only changes require a
process restart. Two consequences follow mechanically:

1. An entry that fails to resolve at boot is **silently skipped** — the PD
   runs with a partial allowlist and never retries.
2. Any peer whose pod IP later changes is **blocked forever** by every peer
   that resolved the old IP.

A dynamic-refresh implementation existed upstream (added and then removed
within the same PR before merge); until it lands in the image, only chart
mitigation and documentation are possible.

## The two log signatures (memorize these)

```
[WARN] o.a.h.p.r.a.IpAuthHandler - Could not resolve allowlist entry '<peer-dns>': ...
[WARN] o.a.h.p.r.a.IpAuthHandler - Blocked connection from <pod-ip>
```

`Could not resolve` at PD start = the first-boot race fired (partial
allowlist). `Blocked connection` = a frozen allowlist rejecting a live peer.
Both are grep-able oracles; counting them per PD is the core health check
after any install, upgrade, or churn event.

## Face 1 — first-install race (install bricks nondeterministically)

With `podManagementPolicy: Parallel` all PDs start simultaneously; a PD can
build its allowlist before a peer's headless-service A-record is published.
Whether the install survives is then a coin flip on two variables: *which* PD
holds the partial allowlist, and *which* PD wins the first raft election.

- If the crippled PD is a **follower**: the leader initiates replication
  outbound, the follower accepts inbound, the cluster limps up
  (degraded — that follower can never initiate RPC).
- If the crippled PD **blocks the elected leader**: it wedges at
  "Leader is not ready", the Server's `KvClient` (which round-robins the
  pd-client Service across all PDs including the zombie) never completes
  startup, all Server pods CrashLoop on their startup probe (exit 143), and
  `helm install --wait` times out with the release `failed`.

Observed rates on a real host: the race fired on 2/2 installs; 1/2 bricked.
A crippled-but-up cluster passes every naive health check — PD readiness
(`/v1/health`) stays green on the zombie member.

**Chart fix**: a `wait-for-pd-dns` init container that blocks each PD until
every peer hostname resolves. Sound because the PD headless Service sets
`publishNotReadyAddresses: true` (A-records appear at IP assignment, before
readiness), so there is no circular wait. After the fix: 2/2 installs with
zero `Could not resolve` / zero `Blocked connection` lines.

## Face 2 — pod-IP churn (rescheduled PD blocked forever)

Delete/reschedule a PD pod: hostname stays (StatefulSet identity), pod IP
changes, uid changes. Every surviving peer still holds the OLD IP in its
frozen allowlist and logs `Blocked connection from <new-ip>` indefinitely.
The restarted pod re-resolves everything fresh at ITS boot, producing an
asymmetric wedge: it can accept leader-initiated traffic but cannot initiate
(pre-vote/vote blocked).

Corollary: **rolling all PDs can never converge** — each pod's allowlist is
frozen at its own boot, so any later peer-IP change is blocked until that pod
also restarts, circularly. This is image-side only; no chart construct can
refresh a running peer's allowlist.

**Operational remediation (proven live)**: delete **all** PD pods at once.
The parallel cold start, gated by `wait-for-pd-dns`, re-resolves every
allowlist against the current IPs and converges clean (0 unresolved,
0 blocked). One-at-a-time restarts make it worse.

## Face 3 — log spam

One wedged PD emitted 31,635 `Blocked connection` WARN lines in ~30 minutes
(one per rejected RPC retry). Fills disks, drowns signal. Image-side; the
line is not rate-limited.

## Adjacent PD surface facts (bite during testing)

- PD management REST (`/v1/members`, `/v1/stores`) rejects **every**
  credential (`AccessDeniedException: invalid service name`) on current
  images, and unauthenticated GETs return **HTTP 200** with body
  `{"error":"Unauthorized!"}` — so membership oracles must come from PD
  *logs* (`becomes leader`, block counts) and the data plane, not REST, and
  status-code health checks are meaningless against PD REST.
- `/v1/health` is the unauthenticated endpoint that works — quorum-counting
  init containers use it.
- The upgrade roll of the PD StatefulSet (any pod-template change) triggers
  Face 2 transiently: expect a block window during and after the roll,
  degraded-but-serving, until the all-at-once remediation is applied.
  Document this in any chart version that changes the PD pod template.
