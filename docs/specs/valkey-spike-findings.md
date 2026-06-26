# Valkey Operator Spike — Findings

**Date:** 2026-06-26
**Cluster:** `rancher-desktop` (k3s v1.34.3, 8 CPU / 16 GB)
**Goal:** Install the Valkey operator, run a minimal spike, assess MVP feasibility.
**Non-goal:** the final `template-valkey` (comes after spec approval).

## 1. Actual install method

Official operator: **github.com/valkey-io/valkey-operator** (WIP, `v1alpha1`).
Helm repo and chart name match expectations — the chart version differs from the assumption:

```bash
helm repo add valkey https://valkey.io/valkey-helm
helm repo update
helm upgrade --install valkey-operator valkey/valkey-operator \
  -n valkey-operator-system --create-namespace
```

- **Chart:** `valkey-operator` **0.2.7**, **app version v0.2.0** (not v0.10 or similar).
- Install is idempotent (`helm upgrade --install`).
- Operator deployment: 1 pod `valkey-operator-*`, becomes `1/1 Ready` with no special configuration.
- The same repo also ships a chart `valkey/valkey` (0.10.0 / app 9.1.0) — that is **not** the operator but a direct server chart. Ignore it for the operator path.

## 2. CRD details

Two CRDs are installed, both in group `valkey.io`, version **`v1alpha1`**:

| CRD | Kind | Purpose |
|-----|------|---------|
| `valkeyclusters.valkey.io` | `ValkeyCluster` | high-level API, user-managed |
| `valkeynodes.valkey.io` | `ValkeyNode` | created by the operator per pod (internal) |

`ValkeyCluster.spec` (excerpt of the relevant fields):

- `shards` (int) — number of shard groups (each 1 primary + N replicas).
- `replicas` (int) — replicas **per shard** (0 = primary only).
- `resources` — resource requirements for the `server` container.
- `persistence` (`size` *required*, `storageClassName`, `reclaimPolicy` Retain/Delete) — PVC per node; **optional**, default ephemeral.
- `users[]` — ACL/auth (see D4-b).
- `config` (map) — additional Valkey config parameters.
- `workloadType` — `StatefulSet` (default) | `Deployment`.
- `tls`, `exporter`, `affinity`, `tolerations`, `topologySpreadConstraints`, `podDisruptionBudget` (Managed/Disabled).

> Important: the operator always runs Valkey in **cluster mode** (16384 hash slots), even for a single node. There is no true standalone — the smallest form is `shards:1, replicas:0` = 1 pod forming a single-node cluster that owns all slots.

## 3. Minimal working ValkeyCluster YAML

Ran immediately, `Ready` after **~24 s**:

```yaml
apiVersion: valkey.io/v1alpha1
kind: ValkeyCluster
metadata:
  name: spike-test
  namespace: valkey-spike-test
spec:
  shards: 1
  replicas: 0
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 250m
      memory: 256Mi
```

**Verification (pod `valkey-spike-test-0-0-0`, container `server`):**

- `valkey-cli ping` → `PONG`
- `valkey-cli cluster info` → `cluster_state:ok`, `cluster_slots_assigned:16384`, `cluster_known_nodes:1`
- `set/get` → OK
- Pod is **2/2**: `server` (`valkey/valkey:9.0.0`) + `metrics-exporter` (`oliver006/redis_exporter:v1.80.0`). The `resources` from the spec apply only to `server`; the exporter runs without requests/limits.

## 4. Gotchas (D4)

**(a) CRD version / stability**
`valkey.io/v1alpha1`, operator app **v0.2.0**, marked **WIP** by the project itself. Alpha API → expect breaking changes between minor versions; pin the version (chart 0.2.7). Functionally stable in the spike, but declared not production-ready.

**(b) ACL / secret format for auth**
Auth is ACL-based. Without `spec.users`, client access is **password-less** (PING works without auth). The operator automatically creates two secrets (owner = ValkeyCluster, garbage-collected with it):
- `internal-<name>-acl` — type `valkey.io/acl`, key `users.acl` = full Valkey ACL file for the internal system users `_operator` and `_exporter`.
- `internal-<name>-system-passwords` — type `valkey.io/acl`, keys `_operator`, `_exporter` (auto-generated passwords).

Custom users: `spec.users[]` with `name` (required) and either `passwordSecret` (secret reference) or `nopass: true`; plus `commands`/`keys`/`channels`/`permissions` (raw ACL). **MVP recommendation:** define at least one app user with `passwordSecret` — the default is auth-less.

**(c) Minimal resource floor**
`64Mi` / `50m` CPU (requests) ran fine for `server`. Note: the **exporter sidecar** comes on top (without its own requests). The effective pod footprint is therefore a bit higher than the `server` requests suggest. For MVP defaults, budget around `128Mi` to cover the exporter plus headroom.

**(d) Readiness conditions of the ValkeyCluster**
Clean condition set. When `Ready`:
- `Ready=True (ClusterHealthy)`
- `Progressing=False (ReconcileComplete)`
- `ClusterFormed=True (TopologyComplete)`
- `SlotsAssigned=True (AllSlotsAssigned)`
- `status.state=Ready`, `status.shards=1`, `status.readyShards=1`
`state` enum: Initializing / Reconciling / Ready / Degraded. Well suited for `auto-ready` / composition readiness checks (check the `Ready` condition or `state=Ready`).

**(e) Upgrade / scale behaviour**
Scaling `replicas: 0 → 1` via `kubectl patch`: the second node `spike-test-0-1` came up in **~25 s**, the cluster stayed `Ready` throughout (no disruption). Valkey-side replication confirmed: primary reports `connected_slaves:1`, `slave0 ... state=online`.
Limitation: the **CRD `ValkeyNode.status.role`** showed `primary` for both nodes, even though the real Valkey `INFO replication` correctly reports `master` + 1 `replica` — the CRD role display is coarser/lagging behind the actual topology. For monitoring, rely on Valkey `INFO` / cluster state rather than the CRD `role` field.
Delete cascades cleanly: `kubectl delete valkeycluster` removes nodes, pods (Terminating) and the internal secrets via ownerReferences.

## 5. Recommendation

**MVP feasible: YES** — with conditions.

The operator installs cleanly, the minimal resource comes up fast and fully functional (`Ready`, PING/SET/GET, 16384 slots), scale and cleanup behave well, and the condition set is a good fit for Crossplane composition readiness logic.

**Conditions for the MVP:**
1. **Accept the alpha risk** — app v0.2.0 / `v1alpha1` is WIP. Pin the chart version (0.2.7), test upgrades, do not run unverified on prod.
2. **Don't forget auth** — the default is password-less. The MVP should set an app user via `spec.users[].passwordSecret`.
3. **Cluster mode is mandatory** — no true standalone; smallest form `shards:1, replicas:0`. Clients/libraries must be cluster-mode capable.
4. **Calculate resource defaults** including the exporter sidecar (~`128Mi`+ per pod, not just the `server` requests).
5. **Choose persistence deliberately** — default is ephemeral; for "real" data set `persistence.size` + a StorageClass.
6. **Readiness** in the composition via the `Ready` condition / `state=Ready`, **not** via the CRD `role` field.

## 6. Auth follow-up: securing the `default` user requires operator #235

A second round (after the template was built) chased real auth — making the
instance actually *require* a password, not just adding a side user.

- **Password-protecting a separate `app` user is useless on its own:** the
  built-in `default` user stays `nopass`, so anything can connect unauthenticated
  as `default`. To enforce auth you must password-protect the **`default`** user.
- **On v0.2.0 that breaks the cluster.** Setting a password on `default` makes the
  server's startup/readiness probe fail with `NOAUTH Authentication required`
  (the probe script runs `valkey-cli PING` as `default`, no password) → the
  `server` container is killed in a loop → cluster never forms (`cluster_state:fail`,
  `slots:0`). Confirmed via pod events.
- **Upstream already fixed this** in PR **#235** (commit `d184606`, 2026-06-17):
  the probe now authenticates as the `_operator` system user (`VALKEY_USER=_operator`
  + `VALKEYCLI_AUTH` from the system secret). **But #235 is NOT in any release** —
  latest is **v0.2.0** (2026-06-09); the fix only lives on `main`.
- **Validated on a from-source build** at commit `5ac4d51` (v0.2.0-12, includes
  #235): password-protecting `default` works end-to-end — `NOAUTH` without the
  password, `PONG`/`SET`/`GET` with it, cluster healthy (`cluster_state:ok`,
  16384 slots), XR `Ready`. The probe env `VALKEY_USER=_operator` is present.
- **Decision:** `cluster-setup.sh` builds the operator from the pinned commit
  `5ac4d51` and runs that image (chart `0.2.7` for everything else). Drop the
  build-from-source step once a release > v0.2.0 ships #235.

**Testing gotcha:** the `server` container has `VALKEYCLI_AUTH` (the `_operator`
password) in its env for the probe. A manual `valkey-cli` exec'd into that
container inherits it → tests as the wrong user. Use `env -u VALKEYCLI_AUTH
valkey-cli -a <password>` (or run from a clean pod) when verifying.
