# Product Spec: Valkey Admin Web UI (per-instance observability)

| | |
|---|---|
| **Spec ID** | VAW (valkey-admin-webui) |
| **Status** | Design approved (PO, 2026-06-30) — implementing |
| **PO / Owner** | Michael (michaelstingl) |
| **Implements** | [template-valkey#9](https://github.com/open-service-portal/template-valkey/issues/9) |
| **Builds on** | [Valkey Service](../valkey-service/product-spec.md) |
| **Tech spec** | in the implementing repo: `template-valkey` (composition) |

## Theme

A developer who orders a **Valkey as a Service** gets, with that order, a web UI to observe *their* Valkey — dashboard, cluster topology, key browser, command logs — without manual setup. Product framing: *"my Valkey → my UI."* One UI per instance (order three, get three independent UIs), naturally tenant-isolated.

Restaurant analogy: the UI is a **side dish plated with the dish** — same namespace as the order, pointed only at that order's Valkey.

## User stories & acceptance criteria

### US-VAW-1 — Bundled web UI for my Valkey

> As a developer ordering Valkey, I get a web UI alongside my instance so I can observe it without any setup.

- **AC-VAW-1-1** — When `spec.observability.webUI` is `true` (the default), the composition renders a `<name>-admin` **Deployment** and **Service** in the order's namespace. · *Test: Integration — `crossplane render` (`tests/render/webui-enabled`)*
- **AC-VAW-1-2** — The UI Deployment's env is wired to the instance: `VALKEY_HOST`/`VALKEY_PORT` to the operator's client Service, `VALKEY_USERNAME`/`VALKEY_PASSWORD` from the instance's observed auth Secret, `VALKEY_TLS` matching the instance. · *Test: Integration — render assertion on env (`tests/render/webui-env`)*
- **AC-VAW-1-3** — When `spec.observability.webUI` is `false`, the composition renders **no** UI Deployment or Service. · *Test: Integration — `crossplane render` (`tests/render/webui-disabled`)*
- **AC-VAW-1-4** — The UI is reachable via `kubectl port-forward svc/<name>-admin 8080:8080`, authenticates to the password-protected instance, and shows dashboard, topology, and key browser. · *Test: E2E — manual validation on rancher-desktop (§V); not automated*

### US-VAW-2 — No operator ownership conflict

> As a platform operator, I want the UI to add observability without fighting the Valkey operator's reconcile loop.

- **AC-VAW-2-1** — The composition does **not** patch the operator-owned StatefulSet/pods; the UI is a standalone Deployment that connects over the network. (Approach A — app-only; the metrics-sidecar path is deferred, see Follow-up.) · *Test: Integration — render assertion that no `StatefulSet`/`ValkeyCluster.spec.containers` patch is emitted (`tests/render/no-sts-patch`)*

## User-facing API (XRD)

```yaml
spec:
  observability:
    webUI: true   # default: true — ship a web UI with the order; set false to opt out
```

`spec.observability.webUI` (boolean, **default `true`**). Default-on is a deliberate product choice: the UI is part of the service; the toggle lets a lean/headless order opt out.

## Behaviour (composition)

When `webUI: true`, in the order's namespace:

1. **Deployment** `<name>-admin` — image `valkey/valkey-admin:1.0.2` (official, pinned), container port `8080`, env per AC-VAW-1-2 plus `DEPLOYMENT_MODE` (the value that makes the app self-collect metrics for one preset connection — confirmed in validation §V), readiness/liveness `GET /` on `8080`.
2. **Service** `<name>-admin` `:8080` → the Deployment.

No extra RBAC — the UI is a plain network client of Valkey.

### Data flow

```
browser → (port-forward) → Service <name>-admin:8080
                              → valkey-admin app  (auth from observed Secret)
                                  → Valkey client Service → discovers topology, polls INFO/stats
```

## Image

- **App image is official:** `valkey/valkey-admin` on Docker Hub (tags through `1.0.2`, maintained by the Valkey community, built from `docker/Dockerfile.app`). Pinned to `1.0.2`.
- No official **metrics** image is published — the reason Approach B (sidecars) is deferred.

## Access

- **MVP: `kubectl port-forward`** to `svc/<name>-admin`, documented in the template README. Zero ingress dependency, no exposure.
- **Later (out of scope):** an optional Ingress/route behind its own toggle, once the platform has an auth/ingress story.

## Follow-up (out of scope)

- **Approach B — operator-managed metrics sidecars.** Advanced metrics (hot-keys via MONITOR, command-log retention) by injecting the upstream metrics sidecar through the operator's native `ValkeyCluster.spec.containers[]` strategic-merge (no StatefulSet ownership conflict). Requires osp to **build and publish a `valkey-admin-metrics` image** (none is published upstream). The operator merges containers but **not** pod volumes — the sidecar must rely on the image's baked default config and ephemeral `/app/data` (both present in `Dockerfile.metrics`). Track as a separate issue; pursue only if advanced metrics are requested. Sidecars are an upstream optimisation for *large* clusters; osp instances are single-node (`shards:1, replicas:0`).
- **Ingress / external exposure** (see Access).

## Validation (§V — do first, before the composition change)

Deploy `valkey/valkey-admin:1.0.2` manually against the live rancher-desktop instance and confirm, before baking into the composition:

1. **`DEPLOYMENT_MODE`** — which value makes the app self-collect metrics for one preset env connection (vs. the K8s registry mode that expects sidecars).
2. **Metrics coverage without sidecars** — which panels populate from the app alone (expected: dashboard, topology, key browser, send-command, command logs) vs. the deferred-B set.
3. **Auth** — the app connects using credentials from the observed Secret against the password-protected default user.
4. **Service DNS** — the exact operator client Service name for `VALKEY_HOST`.

Outcome fills the two confirmed values (`DEPLOYMENT_MODE`, Service name) used in AC-VAW-1-2 and Behaviour.

## Decisions (PO, 2026-06-30)

- Per-instance UI, bundled with the order (the web app is single-target → tenant isolation is free).
- Approach **A** (app-only, official image) now; **B** (sidecars) as a follow-up issue.
- `spec.observability.webUI` defaults to **`true`**.
- Access via `port-forward` for the MVP.

## References

- Issue: [template-valkey#9](https://github.com/open-service-portal/template-valkey/issues/9)
- [Valkey Service spec](../valkey-service/product-spec.md) · [spike findings](../valkey-service/spike-findings.md)
- Upstream: [valkey-io/valkey-admin](https://github.com/valkey-io/valkey-admin) (v1.0.1) · image `valkey/valkey-admin:1.0.2`
- Operator container-merge: `valkey-operator` `api/v1alpha1` — `ValkeyCluster.spec.containers[]` (strategic merge, containers only)
