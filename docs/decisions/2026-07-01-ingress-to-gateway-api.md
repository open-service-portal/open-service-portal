# Adopt Gateway API (Traefik v3) as the osp ingress path; retire ingress-nginx

Status: **Proposed** · Date: 2026-07-01 · Author: michaelstingl (mit Claude)

Related: [`SPEC-exposed-service`](../../specs/exposed-service/0001_product_exposed-service_ingress-dns-tls.md) (the pilot).

## Context

- **ingress-nginx is retiring.** The Kubernetes ingress-nginx controller — which osp installs today via `scripts/cluster-setup.sh` — stops receiving releases, bugfixes, and **security fixes in March 2026** ([k8s blog, 2025-11-11](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)). The `Ingress` API itself is not deprecated, but staying on an unmaintained, security-frozen controller is not acceptable for a platform.
- **Gateway API is the consensus successor.** Kubernetes SIG Network's official replacement is the **Gateway API**; the `ingress2gateway` 1.0 migration tool shipped 2026-03-20. Every major migration guide points to Gateway API.
- **osp has been low-activity for ~9 months**, so this decision is grounded in current (mid-2026) cloud-native best practice rather than older in-repo convention. Research synthesis is recorded in `SPEC-exposed-service` §10.
- osp already models exposure as composition (`whoami-service` = app + Ingress + DNS) and DNS as a service (`CloudflareDNSRecord` + External-DNS). What was missing was TLS and a maintained controller.

## Decision

Adopt **Gateway API** as the osp routing layer, with this stack:

1. **Controller: Traefik v3** (Gateway API implementation). Rationale: lowest ops for a small team — single binary, no separate data plane or mesh, fully conformant against Gateway API v1.5.1, and already the ingress on Rancher Desktop. (Cilium would be preferred only if/when it becomes the CNI.)
2. **TLS: cert-manager + a shared wildcard cert via ACME DNS-01 (Cloudflare).** A single `Gateway` in `gateway-system` holds a `*.<baseDomain>` HTTPS listener; cert-manager (gateway-shim) provisions the wildcard cert using the **existing** External-DNS Cloudflare token — no new credential, no public inbound. Apps attach `HTTPRoute`s from their own namespaces via `ReferenceGrant`. This platform-owned-wildcard model sidesteps cert-manager's per-team self-service caveat (XListenerSet, experimental until ~late 2026).
3. **DNS: External-DNS with `--source=gateway-httproute`.** DNS records follow `HTTPRoute` hostnames automatically; no per-route DNS resource. `CloudflareDNSRecord` remains for standalone DNS not tied to a route.
4. **Developer surface: a composable `ExposedService` XR** (see the spec) that emits just an `HTTPRoute` (+ `ReferenceGrant`); TLS and DNS are automatic platform behaviours.

## Consequences

- `scripts/cluster-setup.sh`: replace the ingress-nginx step with Gateway API CRDs + Traefik v3 + the shared `Gateway` + cert-manager + the Cloudflare DNS-01 `ClusterIssuer`; add External-DNS `--source=gateway-httproute`. New EnvironmentConfig keys: `gatewayName`, `gatewayNamespace`, `baseDomain`.
- `template-whoami` / `template-whoami-service`: migrate `Ingress` → `HTTPRoute` (use `ingress2gateway` to generate a baseline).
- `ExposedService` is the **pilot** for the new pattern; the Valkey Admin web UI ([`SPEC-valkey-admin-webui`](../../specs/valkey-admin-webui/0001_product_valkey-admin-webui.md)) is its first consumer (today only reachable via `kubectl port-forward`).
- Existing ingress-nginx deployments keep working during a parallel-run transition; cut over per host, then remove ingress-nginx.
- On Rancher Desktop the same stack applies (Traefik implements Gateway API); only local name resolution differs (dnsmasq wildcard `*.<baseDomain> → 127.0.0.1`, or a real record).

## Alternatives considered

- **Stay on Ingress with a maintained controller (Traefik Ingress).** Smaller step, but Ingress is the legacy model and Gateway API is where the ecosystem (cert-manager, External-DNS, tooling) is investing; deferring only repeats the migration later.
- **Envoy Gateway / kgateway / Istio / NGINX Gateway Fabric.** All viable Gateway API implementations; rejected for a small team on ops-simplicity grounds (Envoy Gateway was only partially conformant on the official list at last check; Istio adds mesh complexity we don't need). Traefik v3 is abstracted behind the `Gateway` ref, so a later swap needs no change to `ExposedService` or its consumers.
- **mkcert / self-signed local certs for demos.** Rejected: not portable and not browser-trusted everywhere; DNS-01 wildcard gives real certs identically on laptop and prod.

## Open follow-ups

- Pick the exact Traefik v3 install method (Helm values / k3s HelmChartConfig) and pin versions.
- cert-manager XListenerSet (per-team cert self-service) — revisit when it reaches stable (~late 2026) if per-team certs are ever needed.
