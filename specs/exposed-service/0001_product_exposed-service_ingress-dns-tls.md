# Product-Spec: ExposedService (Ingress + TLS + DNS as one composable order)

Spec-ID: `SPEC-exposed-service` · Status: Draft (awaiting PO review) · Datum: 2026-07-01 · Autor: michaelstingl (mit Claude)

Builds on: [`SPEC-valkey-admin-webui`](../valkey-admin-webui/0001_product_valkey-admin-webui.md) (first consumer) ·
Reuses: `CloudflareDNSRecord` (`template-cloudflare-dnsrecord`) · External-DNS ·
Pattern: composition-of-compositions (`template-whoami-service`).

> Language note: this is a **public** repo, so the spec is in English (osp deviation from PD's German house style — see `specs/README.md`).

---

## 1. Theme

A service author who has an in-cluster `Service` can publish it at a real HTTPS URL by ordering **one** thing: an `ExposedService`. That single XR bundles the three legs of "make it reachable, safely": an **Ingress** (route), a **TLS certificate** (trust), and a **DNS record** (name). Product framing: *"my Service → my public HTTPS endpoint."*

Restaurant analogy: `ExposedService` is the **table setting** — plate (Ingress), clean cutlery (TLS), and the reservation card with your name on it (DNS). You order the dish; this makes it servable to guests.

It is a **composable building block**: other service templates embed it the same way `whoami-service` embeds `CloudflareDNSRecord` today. Its first consumer is the Valkey Admin web UI ([`SPEC-valkey-admin-webui`](../valkey-admin-webui/0001_product_valkey-admin-webui.md)), which today is only reachable via `kubectl port-forward`.

## 2. Why

- **Exposure is repeated, TLS is missing.** The platform already composes Ingress + DNS (`whoami-service`), but every Ingress is plain HTTP. There is no reusable "with a valid certificate" story.
- **The Ingress-NGINX controller is retiring.** ingress-nginx (what osp installs today via `cluster-setup.sh`) stops receiving releases, bugfixes, and **security fixes in March 2026** ([k8s blog, 2025-11-11](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)); the official replacement is **Gateway API**. Standardising a *new* capability on the dying controller would be wrong — so `ExposedService` is designed on **Gateway API** (`HTTPRoute` + a shared `Gateway`), which is controller-portable (Traefik, nginx-gateway-fabric, Istio, …).
- **It must be portable, not laptop-specific.** The capability has to work identically on `openportal` and any real cluster — not a Rancher-Desktop-only mkcert/dnsmasq hack. The cluster-specific bits (the `Gateway` ref, cluster-issuer, base domain) belong in per-cluster infra, not in each service's composition.
- **DNS is already a bookable service.** `CloudflareDNSRecord` exists; TLS is the one missing leg. Adding it as a composable block completes the "combine services" concept the platform is built on.
- **No public inbound required.** TLS is issued via ACME **DNS-01** (Cloudflare), so certificates are real (Let's Encrypt, browser-trusted) without exposing the cluster to the internet — the same trust model on a laptop and in prod.

## 3. User stories & acceptance criteria

### US-exp-1 — Publish my Service over HTTPS with one order

> As a service author, I compose (or order) an `ExposedService` for my in-cluster Service and get a working `https://<hostname>` with a trusted certificate and a DNS record — no per-service Ingress/cert/DNS wiring.

- **AC-exp-1-1** — Given `spec.serviceName`, `spec.port`, `spec.hostname`, the composition renders an **`HTTPRoute`** (Gateway API) for that Service/port with hostname = `hostname`, attached to the platform `Gateway` (from EnvironmentConfig). · *Test: Integration — `crossplane render` (`tests/render/httproute`)*
- **AC-exp-1-2** — TLS is terminated at the shared `Gateway` listener (wildcard cert, §5); the `HTTPRoute` requires no per-route cert. The composition authors **no** `Certificate` or cert Secret. · *Test: Integration — render assertion (HTTPRoute has no TLS/secret refs; attaches to the TLS listener) (`tests/render/tls`)*
- **AC-exp-1-3** — DNS follows automatically: **External-DNS** (`--source=gateway-httproute`, platform infra §5) creates the record from the `HTTPRoute` hostname; the composition renders **no** separate DNS resource. `spec.dns.proxied` is emitted as the `external-dns.alpha.kubernetes.io/cloudflare-proxied` annotation on the `HTTPRoute`. · *Test: Integration — `crossplane render` asserts the annotation + that no DNS CR is authored (`tests/render/dns`)*
- **AC-exp-1-4** — End to end on a cluster with the infra (§5): applying an `ExposedService` yields a browser-trusted `https://<hostname>` serving the Service. · *Test: E2E — manual validation (§6); not automated*

### US-exp-2 — Certificates without public inbound (platform infra)

> As a platform operator, I want TLS issued centrally so no service composition carries ACME/secret logic and nothing needs to be reachable from the internet.

- **AC-exp-2-1** — TLS is issued by **cert-manager** via a **Cloudflare DNS-01 `ClusterIssuer`** installed once per cluster (like External-DNS), reusing the **same Cloudflare API token** External-DNS already uses (no second credential). The cert is a **wildcard `*.baseDomain`** provisioned onto the shared `Gateway` listener via cert-manager's Gateway-API support (gateway-shim). · *Test: Integration — infra manifest present + `ClusterIssuer` `Ready` + Gateway listener cert `Ready` on a live cluster (`tests/infra/issuer-ready`)*
- **AC-exp-2-2** — No `ExposedService` composition contains a Cloudflare token, ACME account key, or `Certificate` spec; it only attaches to the shared `Gateway` by reference. · *Test: Integration — render assertion (no secret/Certificate refs) (`tests/render/no-secrets`)*

### US-exp-3 — Portable across clusters

> As a platform operator, I want the same service compositions to work on every cluster, with only per-cluster values differing.

- **AC-exp-3-1** — Cluster-specific values (`Gateway` name/namespace, base domain) are supplied via **EnvironmentConfig**, not hardcoded in the `ExposedService` composition. Swapping the Gateway API implementation (Traefik on RD ↔ another on osp) needs **no** change to `ExposedService` or its consumers. · *Test: Integration — render with two EnvironmentConfigs yields the two Gateway/domain values (`tests/render/env-portability`)*

### US-exp-4 — First consumer: Valkey Admin web UI

> As a developer ordering Valkey with its web UI, the UI is reachable at a real HTTPS URL, not only via port-forward.

- **AC-exp-4-1** — `template-valkey`'s composition composes an `ExposedService` for the `<name>-admin` Service when web-UI exposure is requested; disabling it renders no `ExposedService`. · *Test: Integration — `crossplane render` in `template-valkey` (`tests/render/admin-exposed`)*

## 4. User-facing API (XRD, draft)

```yaml
apiVersion: openportal.dev/v1alpha1
kind: ExposedService
metadata: { name: my-app, namespace: my-team }
spec:
  serviceName: my-app-admin      # in-cluster Service to expose
  port: 8080
  hostname: my-app.openportal.dev   # or just the subdomain, base domain from EnvironmentConfig
  tls:
    enabled: true                # default true
    # issuer: <name>             # optional override; default from EnvironmentConfig
  dns:
    proxied: false               # Cloudflare proxy on/off (default false)
    # type: A | CNAME            # default derived from ingress target
```

Namespaced (Crossplane v2). The composition emits **essentially one resource — a Gateway API `HTTPRoute`** (plus a `ReferenceGrant` if the shared `Gateway` is in another namespace). TLS is served by the shared `Gateway`'s wildcard listener and DNS is created by External-DNS from the route's hostname, so both are **automatic platform behaviours** — the XR carries no cert or DNS-record logic. `spec.dns.proxied` becomes an annotation on the `HTTPRoute`. Cluster-specific defaults (`Gateway` ref, base domain) come from **EnvironmentConfig**, so the XR stays cluster- and controller-agnostic.

## 5. Platform infra (prerequisite, per cluster)

Added to `scripts/cluster-setup.sh` alongside External-DNS (replacing the ingress-nginx step — see §2):
- **Gateway API CRDs** + **Traefik v3** as the Gateway API implementation (§8; lowest ops for a small team, fully conformant v1.5.1, already present on RD).
- A shared **`Gateway`** in a platform namespace (`gateway-system`) with a wildcard HTTPS listener `*.baseDomain`; apps' `HTTPRoute`s in other namespaces attach via **`ReferenceGrant`**.
- **cert-manager** + a **Cloudflare DNS-01 `ClusterIssuer`** (`openportal-cloudflare`, cert-manager's native Cloudflare provider); the Cloudflare token comes from the **existing** External-DNS secret / `.env.${cluster}` — no new credential. cert-manager provisions the **wildcard cert onto the Gateway listener** (gateway-shim; stable for a platform-owned Gateway + shared wildcard — the multi-tenant self-service caveat does not apply here).
- **External-DNS** with **`--source=gateway-httproute`** so DNS records follow `HTTPRoute` hostnames automatically.
- EnvironmentConfig entries: `gatewayName`, `gatewayNamespace`, `baseDomain`.

On Rancher Desktop the same infra applies (Traefik implements Gateway API); only local name resolution differs (dnsmasq wildcard `*.openportal.dev → 127.0.0.1`, or a real record). Local-TLS note → `docs/decisions/` (to add).

## 6. Validation (E2E, manual)

On rancher-desktop with the infra installed + a Cloudflare token: apply an `ExposedService` for a demo Service, resolve the hostname locally, open `https://<hostname>` and confirm a browser-trusted certificate and the Service response. Then repeat via `template-valkey` (admin UI).

## 7. Non-goals / deferred

- Not a general API-gateway / auth proxy — just Ingress + TLS + DNS.
- Non-Cloudflare DNS providers: out of scope for v1 (the issuer/DNS block is provider-shaped so it can extend later).
- mkcert / self-signed local certs: explicitly rejected in favor of real DNS-01 certs (portable, browser-trusted everywhere).

## 8. Open questions & decisions

- **[DECIDED] Ingress vs Gateway API:** **Gateway API** (`HTTPRoute` + shared `Gateway`) — ingress-nginx is retiring (§2), Gateway API is the official replacement and controller-portable.
- **[DECIDED] Cert scope:** **wildcard `*.baseDomain` on the shared Gateway listener** (natural with Gateway API — cert lives with the Gateway, not per-namespace; avoids cross-namespace secret copies).
- **[DECIDED] DNS leg:** **External-DNS `--source=gateway-httproute`** — 2026 best practice: DNS follows the `HTTPRoute` hostname automatically, no per-route CR. (`CloudflareDNSRecord` stays for *standalone* DNS not tied to a route.)
- **[DECIDED] Gateway API implementation:** **Traefik v3** — for a small team the lowest-ops conformant choice (single binary, Gateway API v1.5.1 conformant), already on RD. (Cilium would be the pick only if it becomes the CNI.)
- **[DECIDED] Naming:** **`ExposedService`** — `Route` collides with Gateway API's own route kinds; `ExposedService` states the intent.

All resolved via 2026 web research (see §10) since the project had ~9 months low activity and current cloud-native best practice outweighs stale in-repo convention.

## 9. Platform implication (bigger than this spec)

Moving off ingress-nginx to **Gateway API** affects `scripts/cluster-setup.sh` (controller swap → Traefik v3 + Gateway API CRDs), `template-whoami` / `template-whoami-service` (Ingress → HTTPRoute), and External-DNS config (`--source=gateway-httproute`). This warrants a **decision record** in `docs/decisions/` (osp ingress → Gateway API migration) that `ExposedService` then follows; `ExposedService` is the **pilot**. Migration aid: **`ingress2gateway` 1.0** (2026) converts existing ingress-nginx objects to Gateway API.

## 10. Sources (2026 research)

- ingress-nginx retirement (k8s blog, 2025-11-11) + Ingress2Gateway 1.0 (k8s blog, 2026-03-20) — Gateway API is the consensus successor.
- Gateway API implementations/conformance list (2026) — Traefik v3, Cilium, Istio, NGINX Gateway Fabric, kgateway conformant; Envoy Gateway partial.
- cert-manager Gateway-API guidance (2025-11) — gateway-shim stable for platform-owned Gateway + shared wildcard; XListenerSet (per-team self-service) experimental in 1.20, stable expected late 2026.
- External-DNS Gateway-API source docs (2026-04) — `gateway-httproute` source stable (v1 API).
- Wildcard-on-shared-Gateway + ReferenceGrant + ACME DNS-01 = established small-platform pattern (2026).
