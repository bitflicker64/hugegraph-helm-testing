# The test suite

Phases in order. `$CHART` = path to `helm/hugegraph`. Every command that
touches a cluster carries an explicit `--context "$CTX"` / `--kube-context
"$CTX"` — an empty context variable silently falls back to the user's
current-context, which is how test suites destroy the wrong cluster.

## Static / manifest layer (no cluster)

Run all of it; a missing tool is `NOT-RUN` with a reason, never a silent skip.

```bash
helm lint "$CHART"
helm lint "$CHART" --strict -f "$CHART/values-cluster.yaml"
helm template t "$CHART" > render-default.yaml
helm template t "$CHART" -f "$CHART/values-cluster.yaml" > render-cluster.yaml
helm template t "$CHART" -f "$CHART/values-single.yaml"  > render-single.yaml
```

**validateValues fail-paths** — the chart guards invalid configs with
`fail`; each of these MUST exit non-zero, and a pass is itself a bug:
`pd.pdb.minAvailable=1` and `=3` (quorum math), `pd.partition.defaultShardCount=2`
and `=5`, reserved `server.extraEnv[0].name=JAVA_OPTIONS`,
`networkPolicy.enabled=true`, `pd.replicas=notanumber` (schema), plus — only
meaningful when hubble is enabled (add `--set hubble.enabled=true` on
branches where it defaults off) — `server.auth.enabled=false`,
`hubble.ingress.enabled=true` (no TLS), `hubble.image.tag=`.
Branch defaults change which guards fire: an "unexpected pass" must be
re-checked with the guard's precondition enabled before calling it a bug.

**kubeconform** at the target k8s version AND the chart's `kubeVersion`
floor, `-strict -summary`, on all three renders. **pluto** for deprecated
APIs. **kube-score / polaris** against the CLUSTER render (the default preset
deliberately has empty resources; scoring it is noise). Expected standing
findings: root-by-default containers, liveness==readiness endpoints on
pd/store/server, no NetworkPolicy (explicitly unsupported), ephemeral-storage
unset — report them, they gate restricted-PSS namespaces.

**helm-unittest**: the chart may ship no `tests/`; write a suite in a COPY of
the chart (never modify the chart under test). High-value assertions: peer
lists are DNS-based and track `pd.replicas`; `serviceName` equals the
headless Service name; wait-container quorum math (`REQUIRED=` 1/2/3 at
replicas 1/3/5); PDB gating off at 1 replica; derived
`-Dpartition.default-shard-count` (3 when store.replicas>=3, else 1); Server
startup-probe floor `ceil(450/period)`; auth `secretKeyRef` wiring incl.
`existingSecret` override; ≤63-char names at a 53-char release name (Helm's
own cap). Pin `server.auth.admin.password` and `server.auth.token.value` in
test values — the chart's `lookup`+`randAlphaNum` Secrets regenerate every
render, so never snapshot Secret data.

**Manual render review checklist**: `publishNotReadyAddresses: true` on both
headless Services (load-bearing, see the bug family reference);
`podManagementPolicy: Parallel`; explicit `updateStrategy` and
`persistentVolumeClaimRetentionPolicy` present (older chart versions omit
them); `checksum/auth` on the Server Deployment; no hardcoded IPs/localhost
in rendered config; image tags and pull policies vs what you actually loaded.

## Install + smoke

Kind cluster: 1 control-plane + 3 workers for the full topology.
`kind load docker-image` every component image first — **the load aborts
entirely if any one image is missing locally** (`docker pull` them first),
and check its exit status; a failed load means slow in-node pulls and
possibly `Always`-policy surprises. (Non-Kind distros: use the distro's
import — `minikube image load`, `k3s ctr images import` — or a registry the
cluster can reach.)

**session-values.yaml** (mode 600, in the session dir — it carries a
credential) must at minimum pin the admin credentials, because the chart's
`lookup`+`randAlphaNum` Secrets are otherwise random-per-install and the
smoke/scenario oracles need a known password for admin basic auth:

```yaml
server:
  auth:
    admin:
      password: "<random-you-generated-and-saved-to-a-600-file>"
```

Add image tag/pullPolicy overrides here when they must match what you
loaded. Auth is on by default; if a preset turned it off, force it back on —
the suite tests the auth topology.

Install detached (a foreground `--wait 15m+` outlives most shell-tool
timeouts and a killed helm leaves `pending-install`, which blocks retries
with "cannot re-use a name" while `helm list` shows nothing — use
`helm list -a`):

```bash
setsid nohup helm --kube-context "$CTX" install "$REL" "$CHART" -n "$NS" \
  -f "$CHART/values-cluster.yaml" -f session-values.yaml \
  --wait --timeout 18m > install.log 2>&1 < /dev/null &
```

**Install oracle** (the part naive suites miss): after `deployed`, for every
PD pod grep total counts of `Could not resolve allowlist entry` and
`Blocked connection` — all must be **zero** — and confirm a single current
leader (multiple `Raft becomes leader` lines during a slow bring-up are a
recorded finding, classified by whether the allowlist counts are zero, not
an automatic failure). Then smoke: `/versions` (public), `/graphs`,
schema + vertex write/read via the Server Service with admin basic auth
(`POST /graphs/hugegraph/graph/vertices`, read back
`/graphs/hugegraph/graph/vertices/"1:<name>"`), `helm test`, Hubble `/` 200.
Smoke is `PASS-smoke`, never a suite verdict.

Expected bring-up order: PD quorum → Store leaves its wait-for-pd init →
Server (startup floor 450s) → all Ready. Server crash-looping while PD still
elects is normal early; judge by PD logs, not by waiting longer.

## Distributed fault scenarios

Declare oracle + landing evidence BEFORE the fault. Landing evidence is
mandatory: restartCount bump, uid change, leader identity change, iptables
packet counters — no evidence, no verdict beyond INCONCLUSIVE.

| Scenario | Fault | Oracle | Budget / gotcha |
|---|---|---|---|
| Follower crash | `kubectl exec <pd> -c pd -- sh -c 'kill -9 $(pgrep -f "[j]ava"|head -1)'` | writes 201 during outage; rejoin Ready | ≤300s rejoin. `kill -9 1` is INERT — PID 1 is dumb-init and the kernel drops the signal; kill the java child. |
| Leader crash | same, on the pod whose log says `becomes leader` | new leader elected; pre-fault write durable; write path ≤120s | election is seconds; clients reconnect lazily |
| PD majority loss (negative) | `kubectl delete pod` TWO PDs (delete holds the window; in-place JVM restart recovers too fast) | PD-dependent ops (schema create) FAIL during window — failures are the pass; full recovery ≤600s | 7/8 failures observed = correct quorum protection |
| Store crash + durability | kill java in one store | writes/reads continue (2/3 shards); pre- and mid-fault writes readable after rejoin | store Up-mark lags the 60s patrol |
| Store majority loss | kill java in two stores | honest answer: in-place restart is faster than the window — expect INCONCLUSIVE unless you block rescheduling | don't fake this one |
| Partition of leader | `docker exec <kind-node> iptables -I FORWARD 1 -s/-d <podIP> -j DROP` (Kind nodes are containers; on other distros get a node shell via `kubectl debug node/<n>` and apply the same rules); landing = packet counters | survivors keep serving; converge ≤120s after heal; no split-brain | kubelet probes bypass FORWARD → pod stays Ready, NotReady is NOT evidence. Run EARLY — after churn scenarios the allowlists are poisoned and results are confounded. |
| **Pod-IP churn (mandatory)** | `kubectl delete pod <pd>`; landing = same hostname, NEW podIP, new uid | peers must accept the new IP. Expect blocking on images WITHOUT the allowlist refresh — the verdict comes from the captured `Blocked connection from <new-ip>` lines, never pre-declared. If peers accept the new IP, the image has the refresh: record PASS and note the image version. | if the IP didn't change, the fault didn't land — retry |
| PVC reattach | graceful delete a store pod | `.spec.volumeName` unchanged; pre-fault data readable | local-path PVs are node-pinned on Kind |
| Rolling restart under load | `kubectl rollout restart` store→pd→server with a write sampler | zero LOST acked writes; bounded error window; PD roll causes a block window (bug family Face 2) | run the sampler in the SAME foreground session (see pitfalls) |
| Auth matrix | none | on EVERY Server pod IP: protected read/write = 401 no-cred, 401 wrong-cred, 200/201 admin | `/versions` is PUBLIC — useless as an auth oracle |

## Upgrade path + rotation

1. Install the PREVIOUS chart version, write marker data. Pick the baseline
   as the most recent ref where `Chart.yaml`'s version differs from the
   checkout's (`git log -p -- helm/hugegraph/Chart.yaml` shows the bumps;
   prefer a release tag when one exists), then extract it:
   `git archive <old-ref> helm/hugegraph | tar -x`. On a standalone chart
   checkout with no history, use the previous packaged `.tgz` from wherever
   releases are published; if no prior version is obtainable, mark this
   phase NOT-RUN with that reason rather than skipping silently.
2. Record every pod's uid. `helm upgrade` to the new chart, `--wait`.
3. Judge by **uid diff per component**: pod-template changes must roll
   exactly the components they touch (e.g. PD + Server) and must NOT roll
   the rest (Store). Capture uid lists sorted per component; beware
   terminating old pods polluting the capture (see pitfalls).
4. Marker readable, new writes 201. Expect the PD roll's transient block
   window; then **execute the documented remediation** (delete all PD pods
   at once) and verify 0 unresolved / 0 blocked afterward — a remediation
   that is documented but never executed is not documentation, it's hope.
5. Rotation: create an `existingSecret`, upgrade to it (Server must roll);
   `kubectl patch` the secret's data, upgrade with NO value change — the
   `checksum/auth` annotation must change (resourceVersion moved) and Server
   must roll again. Auth continuity is unchanged by design: the admin
   password is applied at first init only (rotated value gets 401) — that's
   image behavior, not a chart bug.
6. `values-single` (1+1+1): proves the gate self-resolves at one replica.

## User journey

Follow README/NOTES.txt LITERALLY as a first-time user: zero-override
install, run every NOTES command verbatim, fetch the password the documented
way, port-forward, first CRUD, `helm upgrade --reuse-values` with one change,
rollback, uninstall (inspect what's kept: STS PVCs and `resource-policy:
keep` Secrets survive), reinstall over leftovers. Any documented command that
fails as written is a finding (release-literal names, curl assumed in
images, password echoed to terminal). Never run the journey while another
full topology occupies the same host — resource starvation mimics chart
bugs and voids the result.
