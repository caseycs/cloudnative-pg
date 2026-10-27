<!-- bug.yml form fields:
Version: 1.30.1 (also reproduced on main @ af3f2c5d8, testing image sha256:dff2d998…)
Kubernetes: EKS 1.xx (prod) / kind (repro)
Environment: Cloud: Amazon EKS
Install: Helm (cloudnative-pg chart 0.29.1)
-->

**Title:** [Bug]: primary SIGHUPs PostgreSQL several times per second while waiting for a supervised switchover

### What happened?

With `primaryUpdateStrategy: supervised`, after changing `imageName` the replicas are updated and the
cluster goes to `Waiting for user action`. Until the switchover is issued, the primary (still on the
old image) reloads its PostgreSQL configuration several times per second. Each cycle logs, in the
instance manager:

```
Detected drift between the bootstrap images and the configuration. Skipping configuration reload
Requesting configuration reload
```

followed by `received SIGHUP, reloading configuration files` from postgres. On kind that is 196
SIGHUPs per minute on the primary (0 before the image change, 0 on the replica). In production it
cost about 0.4 CPU core per primary for as long as the switchover was pending.

I'd expect the primary to sit idle until the switchover. The log says the reload is skipped, but it
happens anyway.

### Steps to reproduce

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: repro
spec:
  instances: 2
  imageName: ghcr.io/cloudnative-pg/postgresql:18.0-minimal-trixie
  primaryUpdateStrategy: supervised
  primaryUpdateMethod: switchover
  storage:
    size: 1Gi
```

```sh
kubectl patch cluster repro --type merge \
  -p '{"spec":{"imageName":"ghcr.io/cloudnative-pg/postgresql:18.1-minimal-trixie"}}'
# wait for "Waiting for user action"
kubectl logs repro-1 -c postgres --since=60s | grep -c 'received SIGHUP'   # 196
```

### Where it comes from

Two things combine.

The Cluster status flaps, so the instance manager reconciles constantly. Watching the Cluster for 5s
while waiting: 52 updates, 51 of them only flip `status.phaseReason` between `""` and
`"User must issue a supervised switchover"`. The reconcile starts with the admission guard
(`internal/controller/cluster_controller.go:213`), which calls `SetAdmissionError("")` on every
successful validation (`internal/webhook/guard/guard.go:155`). For a Cluster that clears
`Status.PhaseReason` whatever the current phase is (`api/v1/cluster_funcs.go:1689`).
`updateResourceStatus` takes its "before" snapshot after that
(`internal/controller/cluster_status.go:267`), so whenever something else in the status changed its
`Status().Update` persists the blank reason, and `rolloutRequiredInstances` then writes
`RegisterPhase(PhaseWaitingForUser, "User must issue a supervised switchover")` back
(`internal/controller/cluster_upgrade.go:135`). This came in with #10874 (v1.30.0). I haven't pinned
down which status field changes on each loop; the operator logs ~40 "Conflict error while reconciling
resource status" per 10s, so the instance managers' own status writes probably keep it moving.

Each instance-manager reconcile on the drifted primary reloads. In `refreshConfigurationFiles` the
result of `requiresImagesRollout()` is OR-ed into `reloadNeeded`
(`internal/management/controller/instance_controller.go:447`), and it stays true until the pod is
restarted on the new image. From #8115 (v1.27.0).

### Local fix

I changed `SetAdmissionError("")` to clear `PhaseReason` only when the phase is
`PhaseDefinitionInvalid`, i.e. only what the guard itself set. With an operator built from that and
the same repro, while waiting for the switchover: Cluster updates go from ~10/s to 0, operator
reconciles from ~9/s to 0, SIGHUPs on the primary from 196/min to 0. Annotating the Cluster once
still gives one drift message and one SIGHUP, so the second part is still there, it just no longer
runs in a loop. For that one I'd either leave image drift out of `reloadNeeded` or reload once per
new image set.

Happy to send a PR for either part once you say which direction you prefer.

### Relevant log output

```
2026-10-27T11:47:02.169 instance-manager Detected drift between the bootstrap images and the configuration. Skipping configuration reload
2026-10-27T11:47:02.190 instance-manager Requesting configuration reload
2026-10-27T11:47:02.195 postgres received SIGHUP, reloading configuration files
2026-10-27T11:47:02.380 instance-manager Detected drift between the bootstrap images and the configuration. Skipping configuration reload
2026-10-27T11:47:02.413 instance-manager Requesting configuration reload
2026-10-27T11:47:02.414 postgres received SIGHUP, reloading configuration files
```
