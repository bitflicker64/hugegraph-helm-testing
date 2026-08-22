# Case study: the campaign this skill was distilled from

A complete end-to-end application of this skill against chart `hugegraph`
0.1.1 (helm-dev @ `a5088526d`), run on a 16 GB / 12-CPU Ubuntu host with
Kind (k8s v1.35.1), August 2026. Real numbers and log lines — use them to
calibrate what "normal" and "broken" look like.

## Original campaign (session 20260821-204109)

**Static layer**: lint clean; renders default 18 / cluster 19 / single 16
resources; 10/10 validateValues fail-paths failed as designed; kubeconform
`-strict` at k8s 1.30 + 1.23 all valid; pluto clean; ct lint pass;
helm-unittest 18/18 (suite written in a chart copy). kube-score: 48 CRITICAL
lines; polaris 82/100 (`runAsRootAllowed` ×5, `tagNotSpecified` ×1) — the
standing image-limitation findings.

**Install (the headline)**: FAIL, nondeterministic.
- Attempt 1: 20 m timeout, release `failed`. PD/Store/Hubble Ready but all 3
  Servers CrashLoop (exit 143, startup probe exhausted, JVM stuck at
  `o.a.h.p.c.KvClient - wait for client starting....`). Root chain: pd-0
  logged `Could not resolve allowlist entry '<pd-1 and pd-2 DNS>'` at boot
  (Parallel bring-up raced DNS publication), froze a partial allowlist, then
  blocked the elected leader — `Blocked connection from 10.244.1.6`
  **×31,635 in ~30 min** — and sat at "Leader is not ready" forever. Server's
  KvClient round-robins the pd-client Service including the zombie.
- Attempt 2 (fresh ns): deployed in ~71 s — the race fired AGAIN (pd-0
  couldn't resolve pd-1) but pd-0 *won the election*, and a leader initiates
  replication outbound, so bring-up survived, degraded.
- Conclusion: install outcome = f(who holds the partial allowlist, who wins
  the election). Race 2/2, brick 1/2.

**Scenarios** (fault → verdict): follower crash PASS-hardening; leader crash
PASS-hardening (note: `kill -9 1` was inert — dumb-init; killed the java
child instead); PD majority loss (negative) PASS-hardening — 7/8
quorum-dependent ops correctly failed during the window; store crash +
durability PASS-hardening; store majority loss INCONCLUSIVE (in-place JVM
restarts outran the fault window); partition of leader INCONCLUSIVE
(landed — iptables counters 21/23 pkts — but ran after churn had poisoned
allowlists); **pod-IP churn FAIL-reproducible (image)** — delete pd-1,
hostname stable, IP 10.244.1.23→.24, peers logged
`Blocked connection from 10.244.1.24` (39 and 63 lines); PVC reattach
PASS-hardening (same volumeName); rolling restart PASS-hardening for
durability (10/10 markers) with a ~25 s write gap during the PD roll; auth
matrix PASS-hardening on all three Server pod IPs (`/versions` is public —
re-tested on protected endpoints). PD management REST rejected every
credential (`invalid service name`) and returned HTTP 200 +
`{"error":"Unauthorized!"}` unauthenticated.

**User journey**: INCONCLUSIVE-env (aborted partway) — the zero-override
install ran while the scenario cluster still occupied the host and starved
(Server 0/3, PD clean that time); environment blame, honestly recorded.
Lesson: one topology per host.

## The fix (one commit per branch)

`fix(helm): gate PD startup on peer DNS and harden chart defaults` —
9 files, ~246 insertions: the `wait-for-pd-dns` init container +
`hugegraph.pd.peerHostsList` helper + `pd.waitEnabled/waitImage/
waitTimeoutSeconds/waitResources` (schema-enforced); explicit
`updateStrategy` + `persistentVolumeClaimRetentionPolicy` (Retain/Retain) on
PD and Store; `checksum/auth` (resourceVersion-based) on the Server
Deployment; NOTES.txt password echo removed; README release-name note,
port-forward troubleshooting, Upgrading roll notes, Limitations entries for
the image-side items. Chart version bumped per branch; the test branch's
packaged `.tgz` repackaged in the same commit.

Three-lens review returned FIX-FIRST ×3; applied in one amend pass:
data-hash → resourceVersion checksum (security), Upgrading-section roll
documentation + stale version prose (design), resolv.conf guard + typed
`rollingUpdate` schema + waitEnabled hatch (correctness/design).

## Re-test (session 20260822-175403)

- First run of the gate caught a defect in the fix itself: all PDs sat at
  `Init:0/1` because busybox nslookup NXDOMAINed the short `.svc` names
  while the EndpointSlice was already published. Fixed (cluster-domain
  derivation + FQDN retry), amended, battery restarted.
- After the fix: install #1 deployed 10/10, `unresolved=0 blocked=0` on all
  three PDs, clean single election; CRUD 201/200; helm test Succeeded.
  Install #2 (fresh ns): same clean oracle. **2/2 deterministic** vs the
  pre-fix 2/2-race / 1/2-brick.
- Churn probe: pd-1 IP 10.244.2.18→.27 — peers still block (image residual,
  as documented).
- Independent sandbox re-ran the static battery on the patched chart via
  `git format-patch` applied to the public branch: all pass; kubeconform
  accepted `persistentVolumeClaimRetentionPolicy` even at the 1.23 schema;
  three apparent fail-path "passes" traced to the PR branch's
  hubble-disabled default (guards fire with hubble on) — branch defaults
  matter when interpreting guard tests.

## Upgrade suite (session upg-20260822-182300)

- U1: old chart installed clean (won the coin flip).
- U2 upgrade old→new with marker data: revision 2 deployed; **PD 3/3 and
  Server 3/3 rolled, Store 0/3 untouched** (exactly the documented set);
  marker durable, writes 201. Transient allowlist blocks during the roll
  (68 / 446 lines) and ongoing after (30 / 231 per minute) —
  degraded-but-serving, precisely what the new Upgrading note warns.
- U2b: executed the documented remediation — delete all PD pods at once →
  gate-controlled parallel cold start → 0 unresolved, 0 blocked, writes 201.
  Documentation proven, not just written.
- U3 rotation: switch to existingSecret → checksum changed, Server rolled;
  patch the secret + upgrade with no value change → resourceVersion moved,
  checksum changed, Server rolled again. A "1 uid unchanged" scare was a
  Terminating-pod capture artifact (4 superseded ReplicaSets proved full
  rolls). Original password stayed active (image applies it at first init
  only) — documented behavior, not a bug.
- U4 values-single on a 1-node Kind: 4/4 deployed, the single-PD gate
  resolved its own record ("All PD peer DNS resolved."), smoke 200/200.
  Honesty note: an earlier report draft carried a U4 "PASS" row before the
  test had actually run; the row was caught by auditing rows against session
  logs, and the test was then run for real.

## Verbatim shipped code

### wait-for-pd-dns (pd-statefulset.yaml)

```yaml
      {{- if .Values.pd.waitEnabled }}
      initContainers:
        # Gate PD start on all peer DNS names resolving. PD builds its raft RPC
        # allowlist by resolving peer hostnames once at boot; with a Parallel
        # podManagementPolicy a PD can start before its peers' A records are
        # published and freeze a partial allowlist that blocks the leader. The
        # PD headless Service sets publishNotReadyAddresses: true, so peer
        # records appear as soon as pods get IPs (before readiness), keeping
        # this gate free of a circular wait.
        - name: wait-for-pd-dns
          image: {{ .Values.pd.waitImage | quote }}
          {{- with .Values.pd.securityContext }}
          securityContext:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          command:
            - sh
            - -c
            - |
              set -eu
              PEERS=$(echo "{{ include "hugegraph.pd.peerHostsList" . }}" | tr ',' ' ')
              TIMEOUT={{ .Values.pd.waitTimeoutSeconds | default 300 }}
              DEADLINE=$(( $(date +%s) + TIMEOUT ))
              # busybox nslookup ignores resolv.conf search domains, so the
              # short <pod>.<svc>.<ns>.svc names NXDOMAIN even when published.
              # Derive the cluster domain and retry against the FQDN.
              CLUSTER_DOMAIN=$(awk '/^search/ {for(i=2;i<=NF;i++) if ($i ~ /^svc\./) {sub(/^svc\./,"",$i); print $i; exit}}' /etc/resolv.conf 2>/dev/null || true)
              resolves() {
                nslookup "$1" >/dev/null 2>&1 && return 0
                if [ -n "${CLUSTER_DOMAIN}" ]; then
                  nslookup "$1.${CLUSTER_DOMAIN}" >/dev/null 2>&1 && return 0
                fi
                nslookup "$1.cluster.local" >/dev/null 2>&1
              }
              echo "Waiting for all PD peer DNS to resolve: ${PEERS}"
              for host in ${PEERS}; do
                until resolves "${host}"; do
                  if [ "$(date +%s)" -ge "${DEADLINE}" ]; then
                    echo "Timed out after ${TIMEOUT}s waiting for PD peer DNS: ${host}" >&2
                    echo "Check PD Pods: kubectl get pods -l app.kubernetes.io/component=pd" >&2
                    exit 1
                  fi
                  echo "Waiting for DNS: ${host}"
                  sleep 3
                done
                echo "Resolved: ${host}"
              done
              echo "All PD peer DNS resolved."
          {{- with .Values.pd.waitResources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- end }}
```

### Helpers (_helpers.tpl)

```
{{- define "hugegraph.pd.peerHostsList" -}}
{{- regexReplaceAll ":[0-9]+" (include "hugegraph.pd.raftPeersList" .) "" -}}
{{- end }}

{{/*
Checksum for the Server pod template so rotating the referenced auth Secrets
rolls Server pods. Hashes Secret names, keys, and metadata.resourceVersion -
never Secret data - so the annotation carries no credential-derived material.
Lookup-based and therefore best-effort: plain `helm template` (and
template-only GitOps renderers) see no live Secrets and emit a constant; the
first upgrade after a fresh install rolls Server once as the checksum picks
up the Secrets created by that install; out-of-band rotation of an
existingSecret applies on the next `helm upgrade`.
*/}}
{{- define "hugegraph.server.authChecksum" -}}
{{- $parts := list (include "hugegraph.server.authSecretName" .) (include "hugegraph.server.authSecretKey" .) (include "hugegraph.server.authTokenSecretName" .) (include "hugegraph.server.authTokenSecretKey" .) -}}
{{- $admin := lookup "v1" "Secret" .Release.Namespace (include "hugegraph.server.authSecretName" .) -}}
{{- if $admin -}}{{- $parts = append $parts (dig "metadata" "resourceVersion" "" $admin) -}}{{- end -}}
{{- $token := lookup "v1" "Secret" .Release.Namespace (include "hugegraph.server.authTokenSecretName" .) -}}
{{- if $token -}}{{- $parts = append $parts (dig "metadata" "resourceVersion" "" $token) -}}{{- end -}}
{{- join "|" $parts | sha256sum -}}
{{- end }}
```

### Values (pd block)

```yaml
  # Escape hatch for the peer-DNS gate below. Disable only when pod DNS
  # cannot resolve the headless Service names in any form the gate tries;
  # disabling re-exposes the first-boot allowlist race.
  waitEnabled: true
  # The gate needs an nslookup whose exit code reflects NXDOMAIN; busybox
  # older than 1.30 exits 0 on failure and would make the gate a no-op.
  waitImage: curlimages/curl:8.5.0
  # Bound the peer-DNS wait so a cluster whose PD DNS never publishes fails
  # visibly instead of sitting in Init:0/1 forever.
  waitTimeoutSeconds: 300
  # Optional bounds for the PD-start DNS-gate init container.
  waitResources: {}
```

### README Upgrading addition

```markdown
Upgrading to <new-version> from an earlier revision rolls two workloads once:

- **PD** restarts one pod at a time because the Pod template gains the
  `wait-for-pd-dns` init container. On current images a restarted PD returns
  with a new Pod IP that peers holding older allowlists may reject (see
  Limitations). If PDs log `Blocked connection` after the roll, delete all PD
  pods at once — the DNS gate makes the parallel cold start deterministic.
  For a maintenance-window upgrade, set `pd.updateStrategy.type=OnDelete`
  and restart the pods yourself.
- **Server** rolls because the Pod template gains the `checksum/auth`
  annotation, and once more on the first upgrade after a fresh install, when
  the checksum first observes the install-created Secrets. Template-only
  pipelines (`helm template`, GitOps renderers) never see live Secrets, so
  there the annotation is a constant and Secret rotation does not roll pods.
```
