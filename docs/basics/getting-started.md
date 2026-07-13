# Getting Started: Cluster, kubectl Context, Setup

This is the path from "empty machine" to "Open Service Portal running against a
local cluster" — and the concepts you need so the scripts never surprise you.

**The one thing to internalize:** our setup scripts have no `--cluster` flag.
They always operate on whatever `kubectl config current-context` points at.
Selecting the context IS selecting the target cluster.

## 1. What a kubectl context is

`kubectl` reads `~/.kube/config`, which contains three kinds of entries:

- **cluster** — an API endpoint plus its CA certificate ("where").
- **user** — credentials or an auth method: client certificate, token,
  exec-plugin such as OIDC login ("who/how").
- **context** — a named pair of cluster + user (optionally a default
  namespace). This is what you switch between.

One cluster can have **several contexts** — same "where", different "how".
Our shared cluster is the canonical example:

```
NAME              CLUSTER           AUTHINFO       # three ways into ONE cluster
osp               osp               osp            # client certificate
osp-oidc          osp               entra-oidc     # Entra ID OIDC
osp-proxy-oidc    osp               proxy-oidc     # OIDC via proxy
rancher-desktop   rancher-desktop   rancher-desktop
```

This distinction matters for configuration: `cluster-config.sh` keys
everything off the **cluster name** (the `CLUSTER` column), not the context
name — so all three `osp*` contexts share one `.env.osp` and one
`app-config.osp.local.yaml`.

## 2. How contexts come into existence

You rarely write `~/.kube/config` by hand:

- **Rancher Desktop** — enable Kubernetes in the app; it creates and maintains
  the `rancher-desktop` context automatically.
- **kind / k3d / minikube** — `kind create cluster --name foo` adds a
  `kind-foo` context on creation.
- **Remote clusters (like osp)** — you receive or generate a kubeconfig file
  and import it. Use the helper, which merges ALL contexts/clusters/users from
  a `<name>.kubeconfig` file into `~/.kube/config` and preserves every auth
  type (token, exec/OIDC, client certs):

  ```bash
  ./scripts/cluster-kubeconfig.sh osp   # imports from osp.kubeconfig
  ```

- **Fully manual** (rarely needed, but demystifies the file):

  ```bash
  kubectl config set-cluster my-cluster --server=https://1.2.3.4:6443 --certificate-authority=ca.crt
  kubectl config set-credentials me --client-certificate=me.crt --client-key=me.key
  kubectl config set-context my-cluster --cluster=my-cluster --user=me
  ```

## 3. Selecting a context

```bash
kubectl config get-contexts        # list everything; * marks the active one
kubectl config current-context     # print the active one
kubectl config use-context rancher-desktop   # switch
```

**Always check `current-context` before running any setup script.** If your
active context is some unrelated kind cluster, the script will happily install
the whole platform stack into it.

## 4. The setup flow, in order

```
cluster aufsetzen → context einrichten → context auswählen → envs setzen → setup → config → start
```

### Step 1 — Bring up the cluster

Rancher Desktop: start the app, enable Kubernetes, wait until it reports
running. (Any other cluster works the same way — what matters is that a
context for it exists afterwards.)

### Step 2 — Make sure the context exists

Rancher Desktop and kind do this for you (see §2). For remote clusters import
the kubeconfig with `./scripts/cluster-kubeconfig.sh <name>`.

### Step 3 — Select it

```bash
kubectl config use-context rancher-desktop
kubectl config current-context   # verify — the scripts trust this blindly
```

### Step 4 — Set the environment file

Environment files are named after the **cluster name** (see §1):

```bash
cp .env.rancher-desktop.example .env.rancher-desktop
vim .env.rancher-desktop         # GITHUB_TOKEN, optional DNS credentials, …
```

### Step 5 — `./scripts/cluster-setup.sh` (the platform)

Installs the platform components into the **currently selected** cluster, in
this order: NGINX Ingress → Flux (with the catalog watcher) → Crossplane v2 →
provider-kubernetes → cert-manager → External-DNS → provider-helm →
composition functions → platform EnvironmentConfigs → service accounts with
persistent tokens (`backstage`, `gha-app-portal-deploy`).

It is idempotent (helm upgrade --install semantics) — re-running repairs
rather than breaks. At the end it automatically runs `cluster-config.sh` if it
finds the matching `.env.<cluster>` file.

### Step 6 — `./scripts/cluster-config.sh` (your configuration)

Runs automatically after setup (or standalone at any time). It:

- derives the **cluster name** from the active context's cluster field,
- loads `.env.<cluster>`,
- writes `app-config.<cluster>.local.yaml` for Backstage, including a
  generated API token for programmatic catalog access,
- configures External-DNS credentials (if provided),
- creates a demo namespace,
- patches Flux to watch the cluster-specific paths in `catalog-orders`.

### Step 7 — Start Backstage

```bash
cd app-portal
yarn start        # start.js loads app-config modules + your cluster-local yaml
```

## 5. Verifying / troubleshooting

```bash
kubectl config current-context            # am I where I think I am?
kubectl get pods -A                       # platform components healthy?
flux get kustomizations                   # GitOps reconciling?
kubectl get providers.pkg.crossplane.io   # Crossplane providers installed?
./scripts/template-status.sh              # template releases visible?
```

Deeper dives: [local-kubernetes-setup](../cluster/) docs and
[troubleshooting/](../troubleshooting/).

## Why it is done this way

- **Scripts follow the context** instead of taking a flag: one mental model
  (`current-context` = target), no drift between what kubectl shows and what
  scripts do. The cost is that YOU own the check in step 3.
- **Config keyed by cluster name, not context name:** several auth methods
  (certificates, OIDC) can coexist for the same cluster without duplicating
  `.env` files and app-configs.
- **setup vs. config split:** `cluster-setup.sh` is the shared, team-identical
  platform layer; `cluster-config.sh` is the per-cluster/per-person layer
  (credentials, tokens, DNS). You re-run the latter freely without touching
  the platform.
