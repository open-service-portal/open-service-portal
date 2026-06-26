# Valkey Service Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a `ValkeyInstance` Crossplane template so developers can self-service a single Valkey instance, plus the operator install that backs it.

**Architecture:** A new `template-valkey` repo provides an `apiextensions.crossplane.io/v2` XRD (`ValkeyInstance`, group `openportal.dev`, Namespaced) and a Pipeline Composition that renders a `valkey.io/v1alpha1` `ValkeyCluster{shards:1, replicas:0}` applied through provider-kubernetes. The official `valkey-io/valkey-operator` is installed once per cluster via `cluster-setup.sh` (Helm), mirroring cert-manager.

**Tech Stack:** Crossplane v2.0.0, provider-kubernetes, function-go-templating, function-auto-ready, valkey-io/valkey-operator (chart `valkey/valkey-operator`), Helm, `crossplane` CLI v2.4.0, kubectl.

**Spec:** [`docs/specs/valkey-service.md`](../docs/specs/valkey-service.md)

## Global Constraints

- Operator CR: `apiVersion: valkey.io/v1alpha1`, `kind: ValkeyCluster`. MVP always uses `shards: 1`, `replicas: 0`.
- XRD: `kind: ValkeyInstance`, `group: openportal.dev`, version `v1alpha1`, `scope: Namespaced`, `apiVersion: apiextensions.crossplane.io/v2`.
- MVP user params only: `size` (enum `small|medium|large`, default `small`) and `persistence` (`enabled` bool default `false`, `size` quantity default `1Gi`). No auth, TLS, replicas, sharding, or image param.
- Size → resources mapping: small=`{cpu:50m, mem:128Mi/256Mi}`, medium=`{cpu:250m, mem:512Mi/1Gi}`, large=`{cpu:500m, mem:1Gi/2Gi}`.
- Operator install namespace: `valkey-operator-system`. Helm repo: `https://valkey.io/valkey-helm`, chart `valkey/valkey-operator`.
- Composed resources applied via provider-kubernetes `Object` (`kubernetes.m.crossplane.io/v1alpha1`) using `ClusterProviderConfig` named `kubernetes-provider` (the managed/namespaced config already installed by cluster-setup).
- Connection endpoint surfaced on the XR: `valkey-<name>.<namespace>.svc.cluster.local:6379`.
- Never push to `main`; each task commits to a feature branch and lands via PR (self-approval + squash merge is the team norm).
- Repos use the bare+worktree layout; new work happens in a `<repo>/main` (or `<repo>/feat-*`) worktree.

---

### Task 1: Install the Valkey operator via cluster-setup.sh

**Files:**
- Modify: `scripts/cluster-setup.sh` (add `install_valkey_operator` function + call in `main`)

**Interfaces:**
- Produces: a running operator in `valkey-operator-system` and the CRDs `valkeyclusters.valkey.io`, `valkeynodes.valkey.io`. Later tasks rely on these CRDs existing.

- [ ] **Step 1: Add the install function**

In `scripts/cluster-setup.sh`, after `install_provider_helm()`, add:

```bash
# Install the Valkey operator (supplier for ValkeyInstance templates)
install_valkey_operator() {
    echo -e "${YELLOW}Installing Valkey operator...${NC}"
    if helm status valkey-operator -n valkey-operator-system >/dev/null 2>&1; then
        echo -e "${GREEN}✓ Valkey operator already installed${NC}"
        return
    fi
    helm repo add valkey https://valkey.io/valkey-helm
    helm repo update valkey
    helm upgrade --install valkey-operator valkey/valkey-operator \
        -n valkey-operator-system --create-namespace \
        --wait --timeout=5m
    echo -e "${GREEN}✓ Valkey operator installed${NC}"
}
```

- [ ] **Step 2: Call it from `main`**

In `main()`, add the call right after `install_provider_helm`:

```bash
    install_provider_helm  # Install provider-helm for Helm chart deployments
    install_valkey_operator  # Install Valkey operator (ValkeyInstance supplier)
```

- [ ] **Step 3: Run it against the running cluster (idempotent check)**

Run: `kubectl config use-context rancher-desktop && bash -c 'source <(sed -n "/^install_valkey_operator()/,/^}/p" scripts/cluster-setup.sh); install_valkey_operator'`
Expected: prints either "already installed" or installs and ends with "✓ Valkey operator installed".

- [ ] **Step 4: Verify CRDs + pod**

Run: `kubectl get crd valkeyclusters.valkey.io valkeynodes.valkey.io && kubectl get pods -n valkey-operator-system`
Expected: both CRDs listed; operator pod `Running` `1/1`.

- [ ] **Step 5: Commit**

```bash
git add scripts/cluster-setup.sh
git commit -m "feat(cluster-setup): install valkey operator"
```

---

### Task 2: Scaffold the template-valkey repo

**Files:**
- Create repo `open-service-portal/template-valkey` with: `configuration/crossplane.yaml`, `README.md`, `CLAUDE.md`, `.gitignore`, `examples/` (empty for now)

**Interfaces:**
- Produces: the `configuration/` package directory that Tasks 3–5 add `xrd.yaml`, `composition.yaml`, `rbac.yaml` into.

- [ ] **Step 1: Create the GitHub repo and clone as a worktree**

```bash
cd /Users/michaelstingl/Developer/github.com/open-service-portal
gh repo create open-service-portal/template-valkey --private --description "Crossplane template: self-service Valkey instance"
mkdir template-valkey && cd template-valkey
git clone --bare git@github.com:open-service-portal/template-valkey.git .bare
printf 'gitdir: ./.bare\n' > .git
git -C .bare config remote.origin.fetch '+refs/heads/*:refs/remotes/origin/*'
git -C .bare fetch origin 2>/dev/null || true
git worktree add -b main main 2>/dev/null || git worktree add main main
cd main
```

- [ ] **Step 2: Write `configuration/crossplane.yaml`**

```yaml
---
apiVersion: meta.pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: configuration-valkey
  labels:
    provider: kubernetes
    type: database
    openportal.dev/template: "true"
spec:
  crossplane:
    version: ">=v2.0.0"
  dependsOn:
    - provider: xpkg.upbound.io/crossplane-contrib/provider-kubernetes
      version: ">=v0.14.0"
```

- [ ] **Step 3: Write `README.md` and a minimal `.gitignore`**

`README.md`:

```markdown
# template-valkey

Crossplane template for a self-service **Valkey** instance (`ValkeyInstance`,
group `openportal.dev`). Backed by the official `valkey-io/valkey-operator`.
See the spec in portal-workspace: `docs/specs/valkey-service.md`.
```

`.gitignore`:

```gitignore
.DS_Store
```

- [ ] **Step 4: Commit**

```bash
git add configuration/crossplane.yaml README.md .gitignore
git commit -m "chore: scaffold template-valkey configuration package"
git push -u origin main
```

---

### Task 3: Define the ValkeyInstance XRD

**Files:**
- Create: `template-valkey/main/configuration/xrd.yaml`

**Interfaces:**
- Produces: CRD `valkeyinstances.openportal.dev` with `spec.size`, `spec.persistence.{enabled,size}`, `status.ready`, `status.endpoint`. Task 4's Composition targets `compositeTypeRef` `openportal.dev/v1alpha1` kind `ValkeyInstance`.

- [ ] **Step 1: Write `configuration/xrd.yaml`**

```yaml
# XRD: ValkeyInstance — what a developer orders (the "menu item")
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: valkeyinstances.openportal.dev
  labels:
    terasky.backstage.io/generate-form: "true"
    openportal.dev/version: "dev"
  annotations:
    backstage.io/source-location: "url:https://github.com/open-service-portal/template-valkey"
    terasky.backstage.io/add-to-catalog: 'true'
    openportal.dev/tags: "valkey,cache,database"
    terasky.backstage.io/owner: 'platform-team'
    terasky.backstage.io/system: 'infrastructure-templates'
    terasky.backstage.io/component-type: 'crossplane-template'
    terasky.backstage.io/lifecycle: 'production'
spec:
  scope: Namespaced
  group: openportal.dev
  names:
    kind: ValkeyInstance
    plural: valkeyinstances
  defaultCompositionRef:
    name: valkeyinstance
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              size:
                type: string
                description: Instance size (maps to CPU/memory)
                enum: ["small", "medium", "large"]
                default: small
              persistence:
                type: object
                description: Optional persistent storage (survives pod restarts)
                properties:
                  enabled:
                    type: boolean
                    default: false
                  size:
                    type: string
                    description: PVC size (e.g. 1Gi)
                    default: "1Gi"
                default: {}
            required: []
          status:
            type: object
            properties:
              ready:
                type: boolean
                description: Whether the Valkey instance is ready
              endpoint:
                type: string
                description: In-cluster connection endpoint (host:6379)
```

- [ ] **Step 2: Validate the XRD applies (CRD gets established)**

Run: `kubectl apply -f configuration/xrd.yaml && kubectl get xrd valkeyinstances.openportal.dev`
Expected: `valkeyinstances.openportal.dev` shown with `ESTABLISHED=True OFFERED` within a few seconds (`kubectl get crd valkeyinstances.openportal.dev` also exists).

- [ ] **Step 3: Commit**

```bash
git add configuration/xrd.yaml
git commit -m "feat: add ValkeyInstance XRD"
```

---

### Task 4: Write the Composition (renders a ValkeyCluster)

**Files:**
- Create: `template-valkey/main/configuration/composition.yaml`

**Interfaces:**
- Consumes: XRD `ValkeyInstance` (Task 3). Produces composition `valkeyinstance` that emits one provider-kubernetes `Object` wrapping a `ValkeyCluster`, and sets `status.ready`/`status.endpoint`.

- [ ] **Step 1: Write `configuration/composition.yaml`**

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: valkeyinstance
spec:
  compositeTypeRef:
    apiVersion: openportal.dev/v1alpha1
    kind: ValkeyInstance
  mode: Pipeline
  pipeline:
  - step: render-valkeycluster
    functionRef:
      name: function-go-templating
    input:
      apiVersion: gotemplating.crossplane.io/v1beta1
      kind: GoTemplate
      source: Inline
      inline:
        template: |
          {{- $xr := .observed.composite.resource }}
          {{- $name := $xr.metadata.name }}
          {{- $ns := $xr.metadata.namespace }}
          {{- $size := $xr.spec.size | default "small" }}
          {{- $persist := $xr.spec.persistence | default dict }}
          {{- $sizes := dict
                "small"  (dict "cpu" "50m"  "memReq" "128Mi" "memLim" "256Mi")
                "medium" (dict "cpu" "250m" "memReq" "512Mi" "memLim" "1Gi")
                "large"  (dict "cpu" "500m" "memReq" "1Gi"   "memLim" "2Gi") }}
          {{- $res := index $sizes $size }}
          ---
          apiVersion: kubernetes.m.crossplane.io/v1alpha1
          kind: Object
          metadata:
            name: {{ $name }}-valkey
            namespace: {{ $ns }}
            annotations:
              gotemplating.fn.crossplane.io/composition-resource-name: valkeycluster
          spec:
            readiness:
              policy: DeriveFromObject
            forProvider:
              manifest:
                apiVersion: valkey.io/v1alpha1
                kind: ValkeyCluster
                metadata:
                  name: {{ $name }}
                  namespace: {{ $ns }}
                spec:
                  shards: 1
                  replicas: 0
                  resources:
                    requests:
                      cpu: {{ $res.cpu }}
                      memory: {{ $res.memReq }}
                    limits:
                      memory: {{ $res.memLim }}
                  {{- if $persist.enabled }}
                  persistence:
                    size: {{ $persist.size | default "1Gi" }}
                  {{- end }}
            providerConfigRef:
              kind: ClusterProviderConfig
              name: kubernetes-provider
          ---
          apiVersion: openportal.dev/v1alpha1
          kind: ValkeyInstance
          status:
            endpoint: "valkey-{{ $name }}.{{ $ns }}.svc.cluster.local:6379"
  - step: ready
    functionRef:
      name: function-auto-ready
```

- [ ] **Step 2: Render locally with the crossplane CLI (fast unit test)**

Create a throwaway `/tmp/valkey-render/` with `xr.yaml`, `composition.yaml` (copy), and `functions.yaml`:

`functions.yaml`:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-go-templating
  annotations: {render.crossplane.io/runtime: Development}
---
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-auto-ready
  annotations: {render.crossplane.io/runtime: Development}
```

Run: `crossplane render /tmp/valkey-render/xr.yaml configuration/composition.yaml /tmp/valkey-render/functions.yaml`
Expected: output contains a `kind: Object` whose `forProvider.manifest` is a `ValkeyCluster` with `shards: 1`, `replicas: 0`, and `resources.requests.cpu: 50m` for a `small` XR.

> Note: `crossplane render` needs the functions running in Development mode or pulls them; if offline, skip to Step 3 (cluster apply) which exercises the real functions already installed.

- [ ] **Step 3: Commit**

```bash
git add configuration/composition.yaml
git commit -m "feat: add ValkeyInstance composition"
```

---

### Task 5: RBAC for provider-kubernetes to manage ValkeyCluster

**Files:**
- Create: `template-valkey/main/configuration/rbac.yaml`

**Interfaces:**
- Consumes: nothing. Produces: a ClusterRole (aggregated to provider-kubernetes) granting CRUD on `valkey.io` resources so the Composition's `Object` can create `ValkeyCluster`.

- [ ] **Step 1: Write `configuration/rbac.yaml`**

```yaml
# Allow provider-kubernetes to manage ValkeyCluster resources
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: provider-kubernetes-valkey
  labels:
    rbac.crossplane.io/aggregate-to-provider-kubernetes: "true"
rules:
- apiGroups: ["valkey.io"]
  resources: ["valkeyclusters", "valkeyclusters/status"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

- [ ] **Step 2: Apply and verify aggregation**

Run: `kubectl apply -f configuration/rbac.yaml && kubectl get clusterrole provider-kubernetes-valkey`
Expected: ClusterRole created. (Aggregation merges it into the provider-kubernetes ClusterRole automatically.)

- [ ] **Step 3: Commit**

```bash
git add configuration/rbac.yaml
git commit -m "feat: add RBAC for provider-kubernetes to manage ValkeyCluster"
```

---

### Task 6: End-to-end example + cluster integration test

**Files:**
- Create: `template-valkey/main/examples/xr.yaml`

**Interfaces:**
- Consumes: XRD (Task 3), Composition (Task 4), RBAC (Task 5), operator (Task 1). Produces: a working ordered instance, proving the whole path.

- [ ] **Step 1: Write `examples/xr.yaml`**

```yaml
apiVersion: openportal.dev/v1alpha1
kind: ValkeyInstance
metadata:
  name: demo
  namespace: valkey-demo
spec:
  size: small
  persistence:
    enabled: true
    size: 1Gi
```

- [ ] **Step 2: Apply XRD + Composition + RBAC + example to the cluster**

Run:
```bash
kubectl create namespace valkey-demo --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f configuration/xrd.yaml -f configuration/composition.yaml -f configuration/rbac.yaml
kubectl apply -f examples/xr.yaml
```
Expected: `valkeyinstance.openportal.dev/demo created` and no errors.

- [ ] **Step 3: Verify the XR becomes Ready and the underlying ValkeyCluster runs**

Run:
```bash
kubectl -n valkey-demo wait --for=condition=Ready valkeyinstance/demo --timeout=180s
kubectl -n valkey-demo get valkeycluster demo -o jsonpath='{.status.state}{"\n"}'
kubectl -n valkey-demo get valkeyinstance demo -o jsonpath='{.status.endpoint}{"\n"}'
```
Expected: XR condition `Ready=True`; ValkeyCluster state `Ready`; endpoint `valkey-demo.valkey-demo.svc.cluster.local:6379`.

- [ ] **Step 4: Functional check (PING + persistence)**

Run:
```bash
kubectl -n valkey-demo exec valkey-demo-0-0-0 -c server -- valkey-cli ping
kubectl -n valkey-demo get pvc
```
Expected: `PONG`; a bound PVC `valkey-demo-0-0-data`.

- [ ] **Step 5: Tear down the test instance**

Run: `kubectl delete -f examples/xr.yaml && kubectl delete namespace valkey-demo`
Expected: clean deletion (finalizers resolve).

- [ ] **Step 6: Commit + push + PR**

```bash
git add examples/xr.yaml
git commit -m "feat: add ValkeyInstance example and verify end-to-end"
git push -u origin main
```

---

### Task 7: Register the template in the catalog (GitOps)

**Files:**
- Create in the `catalog` repo: `catalog/main/templates/valkey/xrd.yaml`, `catalog/main/templates/valkey/composition.yaml` (or a reference per the catalog's existing pattern)

**Interfaces:**
- Consumes: the released XRD + Composition from template-valkey. Produces: Flux-synced template availability in clusters.

- [ ] **Step 1: Inspect the catalog's existing template registration pattern**

Run: `ls catalog/main/templates/ && sed -n '1,40p' catalog/main/templates/*/xrd.yaml | head -40`
Expected: see how an existing template (e.g. dns-record) is referenced; mirror that exact structure for `valkey/`.

- [ ] **Step 2: Add the valkey template entry mirroring that pattern**

Copy `configuration/xrd.yaml` and `configuration/composition.yaml` into `catalog/main/templates/valkey/` (or add the reference file the catalog uses), matching the existing convention discovered in Step 1.

- [ ] **Step 3: Commit + PR in the catalog repo**

```bash
cd catalog/main
git checkout -b feat/add-valkey-template
git add templates/valkey
git commit -m "feat: register valkey template"
git push -u origin feat/add-valkey-template
gh pr create --repo open-service-portal/catalog --title "feat: register valkey template" --body-file <body>
```

---

## Self-Review

- **Spec coverage:** §2 operator → Task 1. §3 XRD/params/status → Task 3. §4 Composition/provider-kubernetes/readiness → Task 4. §5 operator install → Task 1. §7 validation → Task 6. §9 repo structure → Tasks 2–5. Catalog/GitOps (workflow doc) → Task 7. Deferred items (§6) intentionally absent. No gaps.
- **Type consistency:** XRD `compositeTypeRef` `openportal.dev/v1alpha1`/`ValkeyInstance` matches across Tasks 3, 4, 6. Composition name `valkeyinstance` matches `defaultCompositionRef` in Task 3. ValkeyCluster fields (`shards`, `replicas`, `resources`, `persistence.size`) match the operator CRD verified in `_work/reference/valkey-io/valkey-operator`.
- **Open risk to verify during execution:** the go-templating `status.endpoint` patch on the composite and the `function-auto-ready` readiness propagation are exercised by Task 6 Step 3; if `status.ready` does not flip, adjust the readiness step (set `status.ready` explicitly from the observed ValkeyCluster `status.state` in the go-template).
