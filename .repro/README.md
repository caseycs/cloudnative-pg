# Repro: SIGHUP loop on old-image primary while supervised switchover is pending

Operator: trunk snapshot `ghcr.io/cloudnative-pg/cloudnative-pg-testing:main@sha256:dff2d998…` (2026-10-27).

```sh
kind create cluster --name cnpg-repro
kubectl --context kind-cnpg-repro apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/artifacts/main/manifests/operator-manifest.yaml
kubectl --context kind-cnpg-repro -n cnpg-system rollout status deploy/cnpg-controller-manager
kubectl --context kind-cnpg-repro apply -f .repro/cluster.yaml   # retry until the webhook answers
kubectl --context kind-cnpg-repro wait cluster/repro --for=condition=Ready --timeout=400s

# change the image: the replica is updated, the primary waits ("Waiting for user action")
kubectl --context kind-cnpg-repro patch cluster repro --type merge \
  -p '{"spec":{"imageName":"ghcr.io/cloudnative-pg/postgresql:18.1-minimal-trixie"}}'

# count over the last minute on the primary
kubectl --context kind-cnpg-repro logs repro-1 -c postgres --since=60s \
  | grep -cE 'Detected drift|Requesting configuration reload|received SIGHUP'
```

## Result

| 60s window on `repro-1` (primary, old image) | before patch | while waiting |
|---|---|---|
| `Detected drift between the bootstrap images…` | 0 | 196 |
| `Requesting configuration reload` | 0 | 196 |
| postgres `received SIGHUP, reloading configuration files` | 0 | 196 |

The replica (already on the new image) logs no SIGHUP. Each reconcile on the primary logs
"Skipping configuration reload", then reloads anyway: `requiresImagesRollout()` returns true and
is OR-ed into `reloadNeeded` (`internal/management/controller/instance_controller.go`).

## What drives the reconcile rate

Watching the Cluster for 5s while waiting: 52 updates, 51 of them only flip `status.phaseReason`
between `""` and `"User must issue a supervised switchover"`. Operator log over 10s: 93 reconcile
loops, 54 "Waiting for the user to request a switchover…", 40 "Conflict error while reconciling
resource status".

1. Operator reconcile starts with the admission guard (`internal/controller/cluster_controller.go:213`).
   On successful validation it calls `cluster.SetAdmissionError("")`
   (`internal/webhook/guard/guard.go:155`), which for a Cluster sets `Status.PhaseReason = ""`
   (`api/v1/cluster_funcs.go:1689`) in memory, whatever phase the cluster is in.
2. `updateResourceStatus` snapshots the status *after* that (`internal/controller/cluster_status.go:267`)
   and, if anything else in the status changed, writes the whole status with `Status().Update`:
   the blank reason is persisted.
3. The rollout path then sees a supervised primary and calls
   `RegisterPhase(PhaseWaitingForUser, "User must issue a supervised switchover")`
   (`internal/controller/cluster_upgrade.go:135`): the reason is written back.
4. Every Cluster write wakes the instance manager on the primary, and each of its reconciles sends
   a SIGHUP (see above).

Guard behaviour came with `aefb7a1b4` (#10874, first released in v1.30.0); the reload-on-drift with
`fc7d1e30b` (#8115, v1.27.0). We run chart 0.29.1 = operator 1.30.1 in all environments.

Not verified: which status change in step 2 triggers the full Update each time (not pinned down;
the 40 conflicts/10s suggest the instance manager's own status writes keep the status moving).

## With the guard fix (branch `dev/image-drift-reload-loop`)

`SetAdmissionError("")` only clears `PhaseReason` when the phase is `PhaseDefinitionInvalid`.
Patched operator built from the branch (`cnpg-local:guard-fix`: the trunk image with
`/operator/manager_arm64` replaced), cluster recreated from scratch, image bumped, 60s while waiting:

| | trunk | guard fix |
|---|---|---|
| Cluster updates | ~10/s | 0 in 10s |
| operator reconciles | ~9/s | 0 in 10s |
| SIGHUP on primary | 196/min | 0/min |

Annotating the Cluster once then gives exactly 1 drift message + 1 SIGHUP: the instance-manager
part (reload on every reconcile while drifted) is still there, it just no longer runs in a loop.

Side observations:
- In the first run the primary's container restarted (exit 5, after `Error retrieving lease lock`,
  "Primary lease about to expire with PostgreSQL still running, forcing a stop") 10 min into the loop.
  Not shown to be caused by the loop.
- After that restart the drift check stayed silent: `runningImages` is seeded from
  `cluster.Status.PGDataImageInfo` (already the new image), not from what postgres actually runs.

End the loop: `kubectl cnpg promote repro repro-2` (or `kind delete cluster --name cnpg-repro`).
