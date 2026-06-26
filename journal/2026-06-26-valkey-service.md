# Valkey service (Crossplane) — 2026-06-26

Multi-repo session. PRs: template-valkey #1, #2, #3, #4, #5 · portal-workspace
#127 (merged), #128, #129 (#126 closed) · catalog #43 · ai-plugins-internal #17 (issue).

**What** — Shipped a self-service **Valkey** offering: a `ValkeyInstance` XRD +
Composition (`template-valkey`) that renders a password Secret + a `ValkeyCluster`
(valkey-io operator) via provider-kubernetes; the operator install in
`cluster-setup.sh`; the finalized spec & spike findings; a release workflow →
`ghcr.io/open-service-portal/configuration-valkey:v0.1.0`; catalog registration;
and an end-user ordering guide. Validated end-to-end from the published package
on rancher-desktop, including enforced auth.

**How / decisions**
- Operator: official `valkey-io/valkey-operator` (latest+greatest, accepted alpha
  risk). MVP = single instance (`shards:1, replicas:0`), two knobs (`size`,
  `persistence`), auth **on** (the built-in `default` user gets an auto-generated
  password). TLS/HA/sharding/advanced-ACL deferred; XRD designed to extend later.
- Operator install: **build from a pinned upstream commit** (`5ac4d51`) in
  `cluster-setup.sh`, deployed via the pinned chart `0.2.7` with an image
  override — because the released v0.2.0 cannot password-protect `default` (see
  Learnings). Not in the Composition (operator = platform "supplier").
- Password stability via observed-reuse, not regeneration (see Learnings).

**Learnings (the bits the diff won't tell you)**
- **Securing `default` needs operator ≥ #235.** A *separate* app user is useless —
  `default` stays `nopass`, so the instance is wide open. But password-protecting
  `default` on v0.2.0 makes the server's **startup probe fail with `NOAUTH`**
  (the probe runs `valkey-cli PING` as `default`, no password) → the container is
  killed in a loop → cluster never forms (`cluster_state:fail`, `slots:0`). The
  fix (#235, commit `d184606`) makes the probe auth as the `_operator` system
  user — but it's **only on `main`, not in any release** (latest v0.2.0, tagged
  8 days before #235). Found via pod events + `git tag --contains d184606`. →
  build from source.
- **Don't generate the password every reconcile.** `randAlphaNum` in
  go-templating churns the Secret Object forever (XR never goes Ready). Reuse the
  observed value: `dig "resource" "status" "atProvider" "manifest" "data"
  "password" "" (index .observed.resources "<name>-auth")` then `b64dec`.
- **Test gotcha:** the `server` container ships `VALKEYCLI_AUTH` (the `_operator`
  password) in its env for the probe. A manual `valkey-cli` exec'd into it
  inherits that → `WRONGPASS` on the `default` user. Use `env -u VALKEYCLI_AUTH
  valkey-cli -a "$PW"`.
- **`crossplane xpkg build` rejects non-package objects.** `rbac.yaml` (a
  ClusterRole) in `configuration/` breaks the build → `--ignore="rbac.yaml"`.
- **rancher-desktop uses the docker runtime** (`docker://…` on the node), so a
  locally-built image is usable directly with `imagePullPolicy: Never` — no
  registry push needed for the dev cluster.
- Operator is **cluster-mode only** (no standalone); smallest unit `shards:1,
  replicas:0` = 1 pod owning all 16384 slots. Clients must be cluster-mode aware.
- Readiness: key on the `Ready` condition / `status.state==Ready`, **not**
  `ValkeyNode.status.role` (reports `primary` even for replicas).
- **Public-repo hygiene:** a personal/internal note (a "…not yet pushed…
  (someone's machine)" reference) slipped into the merged spec. Scrub before
  public commits — grep the diff for personal names, local paths (`/Users/`,
  `_work/`), "not yet pushed" notes, and non-English text.

**Follow-ups**
- Switch operator install back to a **chart-version pin** once a release > v0.2.0
  ships #235; then drop the build-from-source step.
- `status.ready` (bool) isn't populated — only the `Ready` condition is. Could
  derive it from the observed `ValkeyCluster` state in the go-template.
- A scoped `provider-kubernetes` RBAC ships in the template (`rbac.yaml`); verify
  it's applied wherever the provider isn't cluster-admin.
- The minimal dev cluster has no Flux/catalog-sync (we installed only Crossplane +
  providers + functions + operator); catalog delivery is exercised by full setups.
- MCP feature request open: configurable channel-message rendering (ai-plugins-internal #17).
