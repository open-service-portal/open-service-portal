# Product-Spec: Valkey Admin Web UI (per-instance observability)

Spec-ID: `SPEC-valkey-admin-webui` · Status: Approved (PO) · Datum: 2026-06-30 · Autor: michaelstingl (mit Claude)

Implements: [template-valkey#9](https://github.com/open-service-portal/template-valkey/issues/9) ·
Builds on: [`SPEC-valkey-service`](../valkey-service/0001_product_valkey-service.md) ·
Tech-Spec: in the implementing repo `template-valkey` (composition).

> Language note: this is a **public** repo, so the spec is in English (osp deviation from PD's German house style — see `specs/README.md`).

---

## 1. Theme

A developer who orders a **Valkey as a Service** gets, with that order, a web UI to observe *their* Valkey — dashboard, cluster topology, key browser, command logs — without manual setup. Product framing: *"my Valkey → my UI."* One UI per instance (order three, get three independent UIs), naturally tenant-isolated.

Restaurant analogy: the UI is a **side dish plated with the dish** — same namespace as the order, pointed only at that order's Valkey.

## 2. Why

- Observability belongs to the product: ordering Valkey should yield something you can *look at*, not just a connection string.
- The upstream web app is **single-target** in Kubernetes (one app ↔ one cluster via an env preset connection), so one UI per instance is both product-correct and the simplest shape — no multi-tenant connection manager to build.
- The Valkey operator **owns** the StatefulSet; observability must not fight its reconcile loop.

## 3. User stories & acceptance criteria

### US-vaw-1 — Bundled web UI for my Valkey

> As a developer ordering Valkey, I get a web UI alongside my instance so I can observe it without any setup.

- **AC-vaw-1-1** — When `spec.observability.webUI` is `true` (the default), the composition renders a `<name>-admin` **Deployment** and **Service** in the order's namespace. · *Test: Integration — `crossplane render` (`tests/render/webui-enabled`)*
- **AC-vaw-1-2** — The UI Deployment's env is wired to the instance: `VALKEY_HOST`/`VALKEY_PORT` to the operator's client Service, `VALKEY_USERNAME`/`VALKEY_PASSWORD` from the instance's observed auth Secret, `VALKEY_TLS` matching the instance. · *Test: Integration — render assertion on env (`tests/render/webui-env`)*
- **AC-vaw-1-3** — When `spec.observability.webUI` is `false`, the composition renders **no** UI Deployment or Service. · *Test: Integration — `crossplane render` (`tests/render/webui-disabled`)*
- **AC-vaw-1-4** — The UI authenticates to the password-protected instance and shows **cluster topology and the key browser**. Browser access requires HTTPS (the app sends HSTS, so a plain `http://` port-forward is unusable — expose via the platform's Gateway API `ExposedService`). The **dashboard CPU/memory metrics and Activity/Hot-Keys do NOT populate** without the metrics sidecar (§6). This is provider-agnostic: cluster-specific values (gateway, base domain) live in the `gateway-config` EnvironmentConfig, so the offering is portable to any managed Kubernetes. · *Test: E2E — manual validation on a valkey-operator cluster (openportal, 2026-07-07); not automated*

### US-vaw-2 — No operator ownership conflict

> As a platform operator, I want the UI to add observability without fighting the Valkey operator's reconcile loop.

- **AC-vaw-2-1** — The composition does **not** patch the operator-owned StatefulSet/pods; the UI is a standalone Deployment that connects over the network. (Approach A — app-only; the metrics-sidecar path is deferred, see §6.) · *Test: Integration — render assertion that no `StatefulSet`/`ValkeyCluster.spec.containers` patch is emitted (`tests/render/no-sts-patch`)*

## 4. User-facing API (XRD)

```yaml
spec:
  observability:
    webUI: true   # default: true — ship a web UI with the order; set false to opt out
```

`spec.observability.webUI` (boolean, **default `true`**). Default-on is a deliberate product choice: the UI is part of the service; the toggle lets a lean/headless order opt out.

## 5. Behaviour (composition)

When `webUI: true`, in the order's namespace:

1. **Deployment** `<name>-admin` — image `valkey/valkey-admin:1.0.2` (official, pinned), container port `8080`, env per AC-vaw-1-2 plus `DEPLOYMENT_MODE=K8` (the Kubernetes **registry mode** — it does NOT make the app self-collect metrics; it expects per-node metrics sidecars, see §6), readiness/liveness `GET /` on `8080`.
2. **Service** `<name>-admin` `:8080` → the Deployment.

No extra RBAC — the UI is a plain network client of Valkey.

Data flow:

```
browser → (port-forward) → Service <name>-admin:8080
                              → valkey-admin app  (auth from observed Secret)
                                  → Valkey client Service → discovers topology, polls INFO/stats
```

Image: `valkey/valkey-admin` is **official** on Docker Hub (tags through `1.0.2`, maintained by the Valkey community, `docker/Dockerfile.app`). No official **metrics** image is published — the reason §6 is deferred.

Access: **MVP `kubectl port-forward`** to `svc/<name>-admin` (documented in the template README); Ingress later, out of scope.

## 6. Out of scope / follow-up

- **Approach B — operator-managed metrics sidecars.** Advanced metrics (hot-keys via MONITOR, command-log retention) by injecting the upstream metrics sidecar through the operator's native `ValkeyCluster.spec.containers[]` strategic-merge (no StatefulSet ownership conflict). Requires osp to **build and publish a `valkey-admin-metrics` image** (none upstream). The operator merges containers but **not** pod volumes — the sidecar must rely on the image's baked default config and ephemeral `/app/data` (both present in `Dockerfile.metrics`). Track as a separate issue; pursue only if advanced metrics are requested. Sidecars are an upstream optimisation for *large* clusters; osp instances are single-node (`shards:1, replicas:0`).
- **Ingress / external exposure.**

## 7. Validation (do first, before the composition change)

Deploy `valkey/valkey-admin:1.0.2` manually against the live rancher-desktop instance and confirm:

1. **`DEPLOYMENT_MODE`** — **ANSWERED — the behaviour is generic to any valkey-operator-managed Kubernetes cluster (not provider-specific; validated on openportal 2026-07-07):** `DEPLOYMENT_MODE=K8` is the registry mode that **expects per-node metrics sidecars** — there is no "self-collect metrics" value. The main backend runs a metrics orchestrator (`POST /register`, `metricsServerMap[nodeId]`); with no sidecar registering, the dashboard reports "Could not register metrics server" / "Metrics server URI not found".
2. **Metrics coverage without sidecars** — **ANSWERED:** connect, cluster topology, key browser, and the interactive console populate from the app alone. The **dashboard CPU/memory metrics and Activity/Hot-Keys do NOT** — they are the sidecar (deferred-B) set. Blocker: the `valkey-admin-metrics` image is not yet published upstream ([valkey-admin#382](https://github.com/valkey-io/valkey-admin/issues/382)).
3. **Auth** — the app connects using credentials from the observed Secret against the password-protected default user.
4. **Service DNS** — the exact operator client Service name for `VALKEY_HOST`.

Outcome fills the two confirmed values (`DEPLOYMENT_MODE`, Service name) used in AC-vaw-1-2 and §5.

## 8. Decisions (PO, 2026-06-30)

- Per-instance UI, bundled with the order (the web app is single-target → tenant isolation is free).
- Approach **A** (app-only, official image) now; **B** (sidecars) as a follow-up issue.
- `spec.observability.webUI` defaults to **`true`**.
- Access via `port-forward` for the MVP.

## 9. References

- Issue: [template-valkey#9](https://github.com/open-service-portal/template-valkey/issues/9)
- [`SPEC-valkey-service`](../valkey-service/0001_product_valkey-service.md) · [spike findings](../valkey-service/spike-findings.md)
- Upstream: [valkey-io/valkey-admin](https://github.com/valkey-io/valkey-admin) (v1.0.1) · image `valkey/valkey-admin:1.0.2`
- Operator container-merge: `valkey-operator` `api/v1alpha1` — `ValkeyCluster.spec.containers[]` (strategic merge, containers only)
