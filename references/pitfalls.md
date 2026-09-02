# Pitfalls that silently invalidate results

Every entry here produced a wrong conclusion or a dead hour in a real
campaign. Skim before writing any test script; re-read when a result looks
too clean or too broken.

## DNS / images / containers

- **busybox `nslookup` ignores resolv.conf search domains.** Short names like
  `pod-0.svc-name.ns.svc` return NXDOMAIN even while the FQDN
  (`...svc.cluster.local`) resolves and the EndpointSlice is published. Any
  DNS-gating script must derive the cluster domain (parse the `search` line
  for the `svc.<domain>` token) and retry the FQDN — or it spins to timeout
  against a perfectly healthy cluster. Also: busybox older than 1.30 exits 0
  on NXDOMAIN, turning any exit-code-based gate into a no-op.
- **`kind load docker-image` aborts the ENTIRE load if one image is missing
  locally.** It does not load the others. `docker pull` everything first and
  check the load's exit status; otherwise pods quietly pull from the network
  inside nodes (slow, and wrong under `imagePullPolicy: Never`).
- **`kind load docker-image` fails outright under Docker 29's containerd
  image store** (`ctr: content digest sha256:...: not found`): the export is
  a multi-platform manifest kind's importer cannot resolve. Use
  `docker save --platform linux/amd64` and `kind load image-archive`, then
  count images on each node with `crictl images`. And install kind from the
  GitHub releases/latest asset; the `dl/latest` redirect has served alphas.
- **`kill -9 1` inside a container is inert** when PID 1 is an init shim
  (dumb-init): the kernel drops signals to PID 1 from its own namespace. To
  crash the app, kill the java child: `kill -9 $(pgrep -f "[j]ava" | head -1)`.
  Conversely `kubectl delete pod` is the graceful path (SIGTERM + grace) —
  choose per what the scenario claims to test, and note in-place JVM
  restarts recover in seconds, often faster than your fault window.
- **Exit code 143 = SIGTERM** (usually kubelet probe-kill), not OOM. Check
  `lastState.terminated.reason` before blaming memory.

## Helm

- **`helm list` hides `pending-install` releases.** A detached install that
  is still running is invisible without `-a`, while a retry fails with
  "cannot re-use a name that is still in use" — which reads like a ghost.
  Always check `helm list -a` before concluding a release doesn't exist.
- **A foreground `helm install --wait --timeout 15m+` outlives shell-tool
  timeouts**; killed mid-flight it leaves `pending-install`, which blocks
  both `upgrade` and `uninstall`. Run installs detached and poll status.
- **`--wait` timing out is a finding, not a retry loop**: capture pods, the
  stuck pod's describe + logs, THEN classify.
- **`lookup` + `randAlphaNum` Secrets make renders nondeterministic**: two
  consecutive `helm template` runs differ. Pin credentials in test values;
  never diff or snapshot rendered Secret data; expect template-only (GitOps)
  renders to differ from live-cluster renders (lookup returns nothing).
- **A shell variable holding several flags is one argument.** `for v in ""
  "-f values-cluster.yaml --strict"; do helm lint . $v` makes helm look for
  a file named ` values-cluster.yaml --strict`; the preset renders come back
  with 0 objects and the lint error looks like a chart failure. Write every
  preset invocation out, or use a `case` table. A 0-object render is a
  harness fault until the invocation has been re-run by hand.
- **`helm template --show-only templates/NOTES.txt` errors** ("could not
  find template") — NOTES.txt is not a manifest. Verify NOTES content at the
  source file, or via `helm install --dry-run` output.
- **Values-file resource mode-bits**: session values files carrying
  passwords must be chmod 600 in a chmod 700 dir — but containers running
  as non-root (polaris, etc.) then can't read files you bind-mount from
  there; copy to a world-readable temp for container-based scanners.

## Remote execution (ssh + long jobs)

- **`nohup ... &` inside an `ssh host 'script'` session often dies with the
  session.** Use `setsid nohup ... < /dev/null &` and verify the process
  registered (e.g. `helm list -a`) before disconnecting.
- **Processes spawned via `kubectl exec` are reaped when the exec session
  closes** — an in-pod `nohup` writer does not survive. For write-samplers
  during rollouts, drive the loop from your own foreground session, or run a
  dedicated probe pod whose main command is the loop.
- **Flaky links mimic host death.** A mesh/VPN peer can show `active` while
  the data path drops rx entirely. Retry with backoff, require N consecutive
  successes before declaring the path stable, and remember detached work on
  the remote host continues regardless — resume, don't restart (timestamped
  session dirs make this safe).
- Always create the k8s resources with a session label and delete only by
  label match. Never `docker system prune`, never touch containers you did
  not create (the host may run unrelated workloads).

## Oracles and captures

- **Pod-uid captures are polluted by Terminating pods**: a label-selector
  listing taken right after `--wait` returns can include an old pod still
  terminating, making a full roll look partial ("1 uid unchanged"). Confirm
  with ReplicaSet generations / `pod-template-hash` before concluding a pod
  dodged a roll.
- **Never write a result row for a test that did not run.** Before any
  report ships, verify every claimed row against the session's own logs —
  a plausible-sounding PASS with no log behind it is the most dangerous
  artifact a campaign can produce.
- **Ordering poisons later scenarios**: pod-IP churn tests poison PD
  allowlists for everything after them. Run network-partition and other
  clean-cluster scenarios BEFORE churn scenarios, or on a fresh install.
- **Resource contention mimics chart bugs.** Ten JVMs need ~10-12 GB; two
  topologies on a 16 GB host starve each other and produce
  CrashLoop/pending states indistinguishable from real failures. One
  topology per host at a time; blame `environment` honestly.
- **`grep -ci ai`-style substring checks lie** (matches "dom**ai**n"). Use
  word-boundary patterns when auditing text for forbidden references.
- **kubelet probes bypass iptables FORWARD drops** (they originate in the
  host netns): a partitioned pod stays Ready. Use packet counters as
  landing evidence, not pod status.
- **PD store-state oracles lag**: the offline patrol runs on a 60s cadence —
  polling `/v1/stores` sooner than ~70s after a store fault reads stale Up.
- **A crippled PD passes `/v1/health`.** It answers 200 as soon as the
  listener is up; with two of three PDs deleted the survivor logged `Raft
  lost leader` within a second and still answered 200 on 26 of 26 samples.
  Membership signals on such images are PD logs (`becomes leader`, block
  counts), `/v1/members` with `-u hg:`, and end-to-end writes. PD images
  from 1.8.0 add `/v1/ready` (503 without a leader); the chart's
  `pd.readinessPath` and `store.waitPath` switch to it.
- **An oracle named from a URL is a guess.** `/v1/members` sounds like "who
  is registered" and lists only PD raft members; Server registrations are
  under `/v1/allInfo` (`data.other`). Read the REST class that serves an
  endpoint before a claim block cites it, and record path, auth and response
  shape next to the claim.

## Reporting

- **Label every claim with its evidence class** (`measured`, `derived`,
  `carried`) and never let a negative claim be anything but measured.
  "PD REST rejects every credential" was derived from source, carried
  through three reports, and used to declare a quorum gate impossible and to
  ship recovery commands without a credential; five curls with the real
  username set and an empty password disproved it. A derived number (a
  lease, a timeout) stays labelled derived until a campaign measures it.

- When the campaign report lives in a shared/concurrently-edited document,
  never blind-overwrite on a conflict: fetch the live version, prove your
  copy is a strict superset (or merge), then write. Shared links sometimes
  pin an older snapshot — re-share/re-pin after meaningful updates.
- To re-test a chart change in any isolated environment, transfer it as
  `git format-patch` output applied onto the public base ref — a
  base64-embedded chart tarball inflates to many times the information it
  carries and wastes the receiving agent's context.
