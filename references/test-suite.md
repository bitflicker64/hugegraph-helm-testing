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
helm lint "$CHART" --strict -f "$CHART/values-single.yaml"
helm template t "$CHART" > render-default.yaml
helm template t "$CHART" -f "$CHART/values-cluster.yaml" > render-cluster.yaml
helm template t "$CHART" -f "$CHART/values-single.yaml"  > render-single.yaml
helm template t "$CHART" --set hubble.enabled=true      > render-hubble.yaml
for f in render-*.yaml; do printf '%-22s %s objects\n' "$f" "$(grep -c '^kind:' "$f")"; done
```

Write each invocation out as above. Do not iterate presets with a shell
variable that holds several flags (`for v in "" "-f values-cluster.yaml
--strict"; do helm lint . $v`): the string expands as one argument, helm
looks for a file named ` values-cluster.yaml --strict`, and the preset
renders report 0 objects, which reads exactly like a chart regression. This
has happened twice. A 0-object render, or a lint error naming a file with a
leading space, is a harness fault: fix the invocation and re-run before any
conclusion about the chart. Record the object counts (current branches:
15 default, 15 cluster, 13 single, 18 with Hubble) so a drift is visible.

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
unset — report them, they gate restricted-PSS namespaces. Do not propose
ephemeral-storage defaults as a fix: the TiDB Operator and CockroachDB charts
set none either (checked 2026-09-02), the `resources` blocks are passthroughs,
and a limit would add an eviction mode nobody has measured.

**helm-unittest**: current branches ship `tests/` (ten suites); run
`helm unittest "$CHART"` and record the count. Older charts ship none; then
write a suite in a COPY of the chart (never modify the chart under test). High-value assertions: peer
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

Kind cluster: 1 control-plane + 3 workers for the full topology (the
cluster preset's `required` anti-affinity needs three schedulable nodes).
Install `kind` from the GitHub releases/latest asset; the
`kind.sigs.k8s.io/dl/latest` redirect has served alpha builds. Preload every
component image, then check the nodes actually hold them:

```bash
docker pull hugegraph/pd:latest   # and store, server, hubble
# Docker 29 with the containerd image store: `kind load docker-image` fails
# with "ctr: content digest sha256:...: not found" on multi-platform
# manifests. Export one platform and load the archive instead.
for i in pd store server hubble; do
  docker save --platform linux/amd64 -o "/tmp/$i.tar" "hugegraph/$i:latest" \
    && kind load image-archive "/tmp/$i.tar" --name "$CLUSTER" && rm -f "/tmp/$i.tar"
done
for n in $(kind get nodes --name "$CLUSTER"); do
  echo "$n: $(docker exec "$n" crictl images 2>/dev/null | grep -c hugegraph) hugegraph images"
done
```

### Building the images yourself

When the tree under test has no published images (an unmerged fix, a testing
branch), build them from that tree on the test host and stamp the revision in.
The bake file in `docker/bake.hcl` defaults to two platforms, which needs
emulation; override it to the host's platform:

```bash
SHA=$(git rev-parse HEAD)
IMAGE_TAG=<tag> SOURCE_REVISION=$SHA docker buildx bake -f docker/bake.hcl pd store server-hstore \
  --set '*.platform=linux/amd64' \
  --set "*.labels.org.opencontainers.image.revision=$SHA" --progress=plain
```

About eight minutes for the three images on 24 cores. Use a tag that exists
nowhere else (`hd-a`, `hd-b`, never the chart default), load with the
`docker save --platform` + `kind load image-archive` pair above, and then
**pin that tag in the session values** for every component: the chart default
plus `IfNotPresent` would make each node pull the registry's copy of the
default tag and the campaign would test a different build without a single
error. Read the identity back from the running pod, through the node that
runs it:

```bash
POD=$(kubectl -n "$NS" get pods -l app.kubernetes.io/component=pd -o jsonpath='{.items[0].metadata.name}')
REF=$(kubectl -n "$NS" get pod "$POD" -o jsonpath='{.status.containerStatuses[0].image}')
NODE=$(kubectl -n "$NS" get pod "$POD" -o jsonpath='{.spec.nodeName}')
docker exec "$NODE" crictl inspecti --output json "$REF" \
  | jq -r '.status.spec // .info.imageSpec | .config.Labels["org.opencontainers.image.revision"] // "NONE"'
```

Push a tag on the source repository for the built tree (`helm-dev-YYYYMMDD`
worked) so PR comments and reports can cite an immutable ref.

`kind load docker-image` **aborts entirely if any one image is missing
locally**, so pull first and check every load's exit status; a failed load
means slow in-node pulls and possibly `Always`-policy surprises. Set
`pullPolicy: IfNotPresent` on all four images in the session values so the
preloaded copies are used. (Non-Kind distros: use the distro's import —
`minikube image load`, `k3s ctr images import` — or a registry the cluster
can reach.) Record the images' `org.opencontainers.image.revision` labels
(`docker inspect --format '{{index .Config.Labels "org.opencontainers.image.revision"}}'`);
Hubble carries none on current builds, note that rather than leaving the
cell blank.

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
| PD majority loss (negative) | `kubectl delete pod` TWO PDs (delete holds the window; in-place JVM restart recovers too fast) | PD-dependent ops (schema create) FAIL during window — failures are the pass; full recovery ≤600s. With `pd.readinessPath` and `store.waitPath` on `/v1/ready`: the survivor answers 503 with `hg_raft_has_leader 0`, `/v1/health` stays 200, and a Store deleted inside the window holds in its init container until two PDs answer | 7/8 failures observed = correct quorum protection. Name every write oracle per run (`d3q_$(date +%s)`): a reused name gets a fast 400 "has existed" from the Server's own state and says nothing about PD. Sample `/v1/ready` with `curl -m 10 -w '%{time_total}'`: on 1.8.0 images the handler stalled 9.8 s during the election before its 503, and a 2 s timeout showed only blanks. The window is 12 to 20 s, under the 30 s readiness failure budget, so the PD Pod itself stays Ready |
| Store crash + durability | kill java in one store | writes/reads continue (2/3 shards); pre- and mid-fault writes readable after rejoin | store Up-mark lags the 60s patrol |
| Store majority loss | kill java in two stores | honest answer: in-place restart is faster than the window — expect INCONCLUSIVE unless you block rescheduling | don't fake this one |
| Partition of leader | Preferred: Chaos Mesh `NetworkChaos` (action partition, `target` selector scoped to the peer PDs), which acts inside the pod's network namespace and reports landing in `status.experiment.containerRecords[].injectedCount` / `recoveredCount`. Fallback: `docker exec <kind-node> iptables -I FORWARD 1 -s/-d <podIP> -j DROP` (Kind nodes are containers; on other distros get a node shell via `kubectl debug node/<n>`); landing = packet counters | survivors keep serving; converge ≤120s after heal; no split-brain | Node-level filtering sits a layer above the pod, so the pod's own probes (which originate on the node) never see the cut: pod stays Ready, NotReady is NOT evidence. Chaos Mesh injects only and supplies no oracle. Run EARLY — after churn scenarios the allowlists are poisoned and results are confounded. |
| **Pod-IP churn (mandatory)** | `kubectl delete pod <pd>`; landing = same hostname, NEW podIP, new uid | peers must accept the new IP. Expect blocking on images WITHOUT the allowlist refresh — the verdict comes from the captured `Blocked connection from <new-ip>` lines, never pre-declared. If peers accept the new IP, the image has the refresh: record PASS and note the image version. | if the IP didn't change, the fault didn't land — retry |
| PVC reattach | graceful delete a store pod | `.spec.volumeName` unchanged; pre-fault data readable | local-path PVs are node-pinned on Kind |
| Rolling restart under load | `kubectl rollout restart` store→pd→server with a write sampler | zero LOST acked writes; bounded error window; PD roll causes a block window (bug family Face 2) | run the sampler in the SAME foreground session (see pitfalls) |
| Auth matrix | none | on EVERY Server pod IP: protected read/write = 401 no-cred, 401 wrong-cred, 200/201 admin | `/versions` is PUBLIC — useless as an auth oracle |
| Server discovery lease | `kubectl delete pod <server>` with a 5s poller on PD `/v1/allInfo` running | three `data.other` rows = three Server Pod IPs before; new IP within one heartbeat; old IP gone within 45s; never more than replicas+1 rows; Hubble `operations/nodes` lists three SERVER | measured 30 to 35s expiry, 5s new registration; see §Discovery lease |

### Claim taxonomy: generating scenarios beyond the table

The table above is the default battery, not the boundary of coverage. Each row
is an instance of the same eight-field claim block (claim, budget tier, fault,
landing evidence, oracle, negative control, ambiguity, blame rule); the block is
reusable, so new claims are generated, not invented. Enumerate the space:

- **Invariant**: availability (writes keep landing), durability (acked data
  survives), consistency (every replica returns the same answer), isolation
  (data stays in its own graph and graphspace), blast radius (one component's
  fault stays in that component), identity (schema and ID namespaces never
  collide), attribution (the image and chart under test are the ones you
  think).
- **Fault**: process crash, pod replacement (new IP and uid), majority loss,
  partition, disk or PVC reattach, clock skew, rolling restart, upgrade,
  secret rotation, slow start.
- **Topology and boundary**: single node vs. one-per-node, replicas 1/3/5,
  two releases in one cluster, two namespaces on one node, first boot vs.
  steady state, first upgrade after install.

For each run, pick at most three new cells the battery does not cover (more
than that and the evidence quality drops), write them into the eight-field
block before the fault, and report coverage as cells exercised over cells
enumerated, not as rows passed. Data-correctness claims need data-shaped
oracles: a written expectation of each graph's contents and a differ, so
"my data survived" becomes "my data stayed only where it belongs".

### Discovery lease (four claims)

Added when the chart switched Servers from announcing the shared Service
URL to announcing their own Pod IP. Read the registry through a port-forward
to the PD Service; every read carries `-u hg:` (PD REST checks the service
name only, see §Endpoints).

```bash
kubectl -n "$NS" port-forward svc/"$REL"-hugegraph-pd 18620:8620 > pf-pd.log 2>&1 < /dev/null &
curl -s -u hg: -H 'Content-Type: application/json' http://127.0.0.1:18620/v1/allInfo \
  | jq -r '.data.other[] | "\(.appName)\t\(.address)\t\(.interval)\t\(.labels.cores)"'
```

1. **Three rows.** `data.other` has one entry per Server replica and the
   addresses equal `kubectl get pod -o wide` Pod IPs on port 8080; each row
   carries `interval 15000` and its own labels. One row with the Service
   name means the announcement env never reached the process (chart); one
   row with a Pod IP means PD keys on something other than the address
   (upstream, and the registry reading is wrong); zero rows is registration
   itself (upstream).
2. **Hubble shows three.** Log in and list nodes (recipe in §Journey);
   three `SERVER` items with distinct uptimes. PD three, Hubble one means the
   Server-side `getServiceUrls` label filter, not the chart.
3. **Expiry within the lease.** Start the poller, delete one Server Pod at
   `T`, record: first sample with the new IP, last sample with the old IP,
   first sample without it, peak row count. Pass: old IP gone within 45s
   (15s heartbeat x 3 misses), new IP within ~one heartbeat, peak = replicas
   + 1. Old IP past 60s: the lease formula does not match the image
   (upstream; correct the Limitations number). Never leaving: PD does not
   expire entries (upstream, serious).

   ```bash
   # poll.sh <pd-host:port> <outfile>
   PD=$1; OUT=$2
   while :; do
     NOW=$(date +%s)
     BODY=$(curl -s -m 4 -u hg: -H 'Content-Type: application/json' "http://$PD/v1/allInfo")
     ADDRS=$(printf '%s' "$BODY" | jq -r '[.. | objects | select(has("address")) | .address] | unique | join(" ")' 2>/dev/null)
     echo "$NOW $ADDRS" | tee -a "$OUT"; sleep 5
   done
   ```

4. **Override still collapses.** `helm upgrade --set-string
   server.advertiseUrl=http://<svc>.<ns>.svc:8080`, wait one lease: exactly
   one `data.other` row with that URL. This proves the outside-Hubble path.

Run the poller from a script file on the host, never inline over ssh, and
stop it by pid: a `pkill -f poll.sh` issued over ssh matches the ssh shell
itself and kills your session.

## Endpoints (read from the handlers, not guessed)

PD REST classes live under `hugegraph-pd/hg-pd-service/.../pd/rest/`; the
auth interceptor's exclusion list is in `AuthenticationConfigurer.java`.
Everything under `/v1` that is not excluded takes Basic auth, and on current
images the check is the **service name only** (`hg`, `store`, `hubble`,
`vermeer`; any password, including empty); every outcome answers HTTP 200
with the result in the body, so status codes carry no signal there.

| Endpoint | Auth | What it is | Source |
|---|---|---|---|
| `GET /v1/health` | none | liveness: listener is up, never consults raft | `StoreAPI.checkHealthy` |
| `GET /v1/ready` | none | readiness: 200 only with an active raft node and a leader, else 503; PD images from 1.8.0 (apache/hugegraph#3185) | `StoreAPI.checkReady` |
| `GET /v1/members` | `-u hg:` | PD raft members, `pdLeader` with role and state | `MemberAPI` |
| `GET /v1/stores` | `-u hg:` | registered Stores and state counts; lags the 60s patrol | `StoreAPI` |
| `GET /v1/allInfo` | `-u hg:` | everything registered: `data.PD`, `data.STORE`, `data.other` (Servers) | `RegistryAPI.getAllInfo` |
| `POST /v1/registryInfo` | `-u hg:` | registry query by `appName` / labels / version, JSON body | `RegistryAPI.getInfo` |
| `/actuator/prometheus` | none | `hg_up`, `hg_stores`, `hg_terms`, `hg_graphs`; from 1.8.0 also `hg_raft_leader`, `hg_raft_has_leader`, `hg_raft_alive_peers`. `alive_peers` is NaN on every non-leader by design (the registration returns NaN when `getAlivePeerCount` is -1), so a node that lost leadership reads NaN, not a defect. Labels are emitted as `hg_raft_has_leader{hg="pd",} 1.0`; strip them (`sed -E 's/\{[^}]*\}//'`) before counting | `PDMetrics` |

Server registrations are **not** in `/v1/members`; that endpoint lists PD's
own raft group. The registry key is app name / version / address
(`DiscoveryMetaStore.toKey`), which is why the announced address decides
whether replicas collapse into one row.

## Upgrade path + rotation

### Image-swap variant (two-stage campaign)

To measure an image-side fix, build two image sets from adjacent tree states
(without and with the fix), install on the first, run the battery, then
`helm upgrade` changing only the image tags. One cluster then yields the
negative control (the old behaviour reproduced on stage A), the fix (the
behaviour flipped on stage B) and an image-roll upgrade path. Oracle for the
roll: uid diff per component (exactly the components whose image changed),
the revision label read from every pod afterwards, a marker written on stage
A readable on stage B, and the stage A control matrix repeated on stage B.
Workloads carrying a lookup-based checksum annotation also roll on the
**first** upgrade after a fresh install (the lookup first sees the
install-created Secret then), so do not make stage B the first upgrade if
the roll set is part of the oracle, and expect any port-forward to those pods
to die during it.


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

**Hubble through its API** (faster and stricter than driving the UI, and
the only way to assert counts): the backend is `/api/v1.3`; the login body
is the hugegraph-client `Login` shape, `user_name` and `user_password`, and
the session is cookie based. Four wrong attempts lock the account for 5s
(HTTP 429), so get the password from the chart's admin Secret first.
`/actuator/mappings` returns the single-page frontend, not the route table;
routes come from `hubble-be/.../controller/` in hugegraph-toolchain.

```bash
PW=$(kubectl -n "$NS" get secret "$REL"-admin -o jsonpath='{.data.password}' | base64 -d)
kubectl -n "$NS" port-forward svc/"$REL"-hugegraph-hubble 18088:8088 > pf-hubble.log 2>&1 < /dev/null &
curl -s -c hubble.cookies -H 'Content-Type: application/json' \
  -d "{\"user_name\":\"admin\",\"user_password\":\"$PW\"}" http://127.0.0.1:18088/api/v1.3/auth/login
curl -s -b hubble.cookies http://127.0.0.1:18088/api/v1.3/operations/capabilities
curl -s -b hubble.cookies 'http://127.0.0.1:18088/api/v1.3/operations/nodes?page_size=100' \
  | jq -r '.data.items[] | "\(.type)\t\(.status)\t\(.name)\t\(.metrics.system.basic.uptime // "-")"' | sort
```

Expected on the full topology: three `PD`, three `STORE`, three `SERVER`,
all `UP`, Servers with distinct uptimes. `operations/overview` gives the
per-source availability. The Server items carry no address, so the count
and the uptimes are the evidence; the addresses come from PD `/v1/allInfo`.
