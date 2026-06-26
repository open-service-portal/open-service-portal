# Spec: Valkey Service (Crossplane)

| | |
|---|---|
| **Status** | Draft — for PO review |
| **PO / Owner** | Michael (michaelstingl) |
| **Date** | 2026-06-26 |
| **Scope** | MVP, KISS — single Valkey instance, self-service via Crossplane |
| **Backstage** | Out of scope for now (Crossplane-first) |

## 1. Goal

Let developers self-service a **Valkey** instance (Redis-compatible in-memory key/value store) through Crossplane: order a namespaced XR, get a running, connectable Valkey. Backstage form integration comes later — first make it work via Crossplane.

Restaurant analogy:

- **Menu** = `Valkey` XRD
- **Kitchen** = Composition
- **Supplier** = `valkey-operator`
- **Order** = the XR

## 2. Operator decision (confirmed)

Build on the **official `valkey-io/valkey-operator`** (`valkey.io/v1alpha1`, chart `valkey/valkey-operator`). Chosen deliberately per the platform's "latest + greatest" philosophy. It is **WIP** (`v1alpha1` may change, "not for production") — acceptable for our local/dev platform.

Key constraint (validated): the operator is **cluster-mode only** — there is no true standalone. The smallest unit is `ValkeyCluster{shards:1, replicas:0}` → a single pod running in cluster mode. For our MVP this *is* the "single instance".

> Alternatives (hyperspike, SAP) only if a real single-node / simple-password mode becomes a hard requirement. Not now.

## 3. User-facing API (XRD)

- **Kind:** `ValkeyInstance`
- **Group:** `openportal.dev`, **version** `v1alpha1`
- **Scope:** `Namespaced` (Crossplane v2 — direct XR, no claim)

### MVP parameters (KISS — two knobs)

| Param | Type | Default | Maps to |
|---|---|---|---|
| `size` | enum `small\|medium\|large` | `small` | `ValkeyCluster.spec.resources` (requests+limits) |
| `persistence.enabled` | bool | `false` | selects `workloadType` / adds `persistence` |
| `persistence.size` | quantity | `1Gi` | `ValkeyCluster.spec.persistence.size` |

Size mapping (starting point, tune later):

| size | cpu req | mem req | mem limit |
|---|---|---|---|
| small | 50m | 128Mi | 256Mi |
| medium | 250m | 512Mi | 1Gi |
| large | 500m | 1Gi | 2Gi |

Everything else (`shards:1`, `replicas:0`, exporter default) is fixed/hidden in the Composition. Auth, TLS, replicas, sharding are **not** exposed in the MVP.

### Status (surfaced on the XR)

- `ready` (bool) — derived from `ValkeyCluster status.state == Ready`
- `endpoint` — `valkey-<name>.<namespace>.svc.cluster.local:6379`

## 4. Behaviour (Composition)

Pipeline-mode Composition (matches existing templates):

1. `function-go-templating` renders a `ValkeyCluster` (valkey.io/v1alpha1) with `shards:1, replicas:0`, mapped `resources`, and optional `persistence`.
2. The `ValkeyCluster` is applied via **provider-kubernetes** `Object` (`kubernetes.m.crossplane.io/v1alpha1`, namespaced, like template-whoami).
3. Readiness keyed on the operator's `status.state == Ready` (`function-auto-ready` or explicit readiness check).

## 5. Operator installation (platform infra)

Install the operator **once per cluster** via `cluster-setup.sh`, exactly like cert-manager (idempotent Helm) — **not** inside the Composition (the operator is the "supplier", not part of an order):

```bash
helm repo add valkey https://valkey.io/valkey-helm
helm upgrade --install valkey-operator valkey/valkey-operator \
  -n valkey-operator-system --create-namespace
```

## 6. Out of scope (deferred)

Deferred for the MVP:

- Auth / ACL
- TLS
- Replicas / HA / failover
- Sharded cluster
- Backup / restore
- Backstage form / scaffolder integration
- Custom `config` tuning

The XRD is designed so these can be added as optional params later without breaking changes.

## 7. Validation — spike results (2026-06-26, rancher-desktop)

Already proven on a live cluster (Crossplane v2.0.0 + operator chart 0.2.7 / app v0.2.0):

- `ValkeyCluster{shards:1, replicas:0}` → **Ready / ClusterHealthy in ~15s**, 1 pod, `PING→PONG`, `SET/GET` ok.
- **Resource floor tiny** (real usage ~6m CPU / 18Mi RAM) — `small` is generous.
- **Persistence works end-to-end**: PVC `valkey-<name>-0-0-data` (local-path), data **survives pod restart**.
- **Default user `nopass +@all`** → clients connect without auth out of the box. ACL secrets only for internal `_operator`/`_exporter`. → auth/TLS safely deferred.
- Objects: StatefulSet `valkey-<name>-0-0`, Service `valkey-<name>:6379`, ConfigMap, ACL secrets. Containers: `server` + `metrics-exporter` (:9121).

## 8. Decisions (resolved by PO, 2026-06-26)

- **D5 — XRD kind:** `ValkeyInstance` (neutral; covers cache *and* persistent store).
- **D6 — sizing:** `size` enum (`small|medium|large`) for the MVP; a raw `resources` override may be added later (non-breaking).
- **D7 — Valkey version:** use the **operator default** image — no pin and no `image` param in the MVP. Revisit if reproducibility becomes a requirement.

## 9. Implementation outline

New repo **`template-valkey`** following [template-standards.md](../template-standards.md): `configuration/` package with `xrd.yaml`, `composition.yaml`, `rbac.yaml`, `examples/xr.yaml`, `crossplane.yaml`. Operator install added to `scripts/cluster-setup.sh` + `scripts/manifests/setup/`. Detailed steps go into a separate implementation plan (writing-plans).

## 10. References

- Operator source (local): `_work/reference/valkey-io/valkey-operator/` (`api/v1alpha1/*_types.go`, `config/samples/`)
- Helm chart (local): `_work/reference/valkey-io/valkey-helm/`
- Existing pattern: `template-whoami/configuration/` (XRD + Pipeline Composition)
- Felix strawman (not yet pushed): `main/docs/specs/valkey-service.md` (his machine)
