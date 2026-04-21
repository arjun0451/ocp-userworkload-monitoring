# OpenShift Workload Monitoring Dashboard

A production-grade, Red Hat-themed namespace and workload visibility dashboard for OpenShift clusters. Built with Node.js, native Kubernetes HTTPS API (no `oc` binary), and a single-file SPA frontend.

---

## Table of Contents

1. [Architecture](#architecture)
2. [Features](#features)
3. [Prerequisites](#prerequisites)
4. [Installation — Form 1: Direct Manifest](#installation--form-1-direct-manifest)
5. [Installation — Form 2: Helm Chart](#installation--form-2-helm-chart)
6. [Build the Container Image](#build-the-container-image)
7. [Configuration](#configuration)
8. [AppGroup Label Filter](#appgroup-label-filter)
9. [RBAC — What the Dashboard Can Access](#rbac--what-the-dashboard-can-access)
10. [Rollback](#rollback)
11. [Cleanup](#cleanup)
12. [Local Development](#local-development)
13. [Troubleshooting](#troubleshooting)

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  Browser (SPA — index.html)                         │
│  Red Hat themed · Dark/Light · Auto-refresh         │
└───────────────────┬─────────────────────────────────┘
                    │ GET /api/dashboard
┌───────────────────▼─────────────────────────────────┐
│  Node.js Express  (src/server.js)                   │
│  Port 3000 · UBI9 minimal · UID auto (restricted-v2)│
└───────────────────┬─────────────────────────────────┘
                    │ HTTPS (in-cluster SA token)
┌───────────────────▼─────────────────────────────────┐
│  Kubernetes API Server                              │
│  /api/v1/namespaces                                 │
│  /api/v1/pods (all namespaces)                      │
│  /api/v1/nodes                                      │
│  /apis/apps/v1/deployments (all namespaces)         │
│  /apis/apps/v1/statefulsets (all namespaces)        │
└─────────────────────────────────────────────────────┘
```

**No `oc` binary required.** Data is fetched using native HTTPS requests to the K8s API using the pod's mounted ServiceAccount token.

**SCC:** `restricted-v2` (OCP default). No `anyuid`, no hardcoded `runAsUser`. OCP injects the namespace UID automatically — fully portable across all clusters.

---

## Features

| Feature | Detail |
|---|---|
| Namespace tiles | One card per user namespace (all `openshift-*`, `kube-*`, `default` excluded) |
| Workload counts | Deployments, StatefulSets, Pods per namespace with expandable detail |
| Pod health bar | Running / Pending / NotReady / Error / Succeeded per namespace |
| Pod status | 11 classified states: Running, Pending, NotReady, CrashLoopBackOff, OOMKilled, ImagePull, ConfigError, Unschedulable, Failed, Terminating, Unknown |
| Node view | Collapsible node strip with role (CP/Worker), zone, readiness |
| AppGroup filter | Collapsible section — filter namespaces by `appgroup` label, executive summary per group |
| Filter bar | Namespace search (comma-separated), health chips, pod status, pod name, node |
| Sort | Critical first / A→Z / Z→A / Most pods / Fewest pods / Most errors |
| Clickable summary tiles | Tiles drive pod status filters; Nodes tile expands node strip |
| Auto-refresh | On/Off toggle with configurable interval (1–60 min, default 10) |
| Dark / Light mode | Toggle with localStorage persistence |
| Executive summary | Per-appgroup: namespace count, deployments, statefulsets, pod status breakdown |

---

## Prerequisites

| Requirement | Version |
|---|---|
| OpenShift | 4.10+ |
| `oc` CLI (client only) | Any recent |
| Podman or Docker | For image builds |
| Helm (optional) | 3.x |
| Quay.io account | For image hosting |

---

## Installation — Form 1: Direct Manifest

**Fastest path — no Helm required.**

```bash
# 1. Edit image reference
vi k8s/all-in-one.yaml
# Find:  image: quay.io/arjun0451/backup/ocp-health-dashboard:v5
# Replace with your built image tag

# 2. Apply everything
oc apply -f k8s/all-in-one.yaml

# 3. Watch rollout
oc rollout status deploy/ocp-ns-dashboard -n ocp-ns-dashboard

# 4. Get the Route URL
oc get route ocp-ns-dashboard -n ocp-ns-dashboard -o jsonpath='{.spec.host}'
```

**Verify SCC (should be `restricted-v2`):**
```bash
oc get pod -n ocp-ns-dashboard \
  -o jsonpath='{.items[0].metadata.annotations.openshift\.io/scc}'
```

---

## Installation — Form 2: Helm Chart

```bash
# Default install (namespace auto-created)
helm install ocp-ns-dashboard ./helm/ocp-ns-dashboard \
  -n ocp-ns-dashboard --create-namespace

# With production overrides
helm install ocp-ns-dashboard ./helm/ocp-ns-dashboard \
  -f helm/values-production.yaml \
  -n ocp-ns-dashboard --create-namespace

# Check status
helm status ocp-ns-dashboard -n ocp-ns-dashboard

# Upgrade (e.g. new image tag)
helm upgrade ocp-ns-dashboard ./helm/ocp-ns-dashboard \
  --set image.tag=v6 -n ocp-ns-dashboard

# Uninstall
helm uninstall ocp-ns-dashboard -n ocp-ns-dashboard
oc delete namespace ocp-ns-dashboard
```

### Key Helm values

| Value | Default | Description |
|---|---|---|
| `image.repository` | `quay.io/arjun0451/backup/ocp-health-dashboard` | Image registry path |
| `image.tag` | `v5` | Image tag |
| `namespace` | `ocp-ns-dashboard` | Target namespace |
| `route.enabled` | `true` | Create OpenShift Route |
| `route.host` | `""` | Custom hostname (empty = auto) |
| `resources.requests.cpu` | `50m` | CPU request |
| `resources.requests.memory` | `64Mi` | Memory request |
| `extraEnv` | `[]` | Additional env vars (e.g. `APP_GROUP_LABEL`) |

---

## Build the Container Image

The image uses a **two-stage UBI9 build** — builder stage installs Node.js dependencies as root, runtime stage drops to non-root for OCP compatibility.

### Mac M3 → linux/amd64 (OCP x86 nodes)

```bash
podman build \
  --platform linux/amd64 \
  -t quay.io/<your-org>/ocp-ns-dashboard:v1 \
  -f Containerfile \
  .

podman push quay.io/<your-org>/ocp-ns-dashboard:v1
```

### Docker buildx

```bash
docker buildx build \
  --platform linux/amd64 \
  -t quay.io/<your-org>/ocp-ns-dashboard:v1 \
  --push \
  -f Containerfile \
  .
```

### Private Quay registry — create pull secret

```bash
oc create secret docker-registry quay-pull-secret \
  --docker-server=quay.io \
  --docker-username=<username> \
  --docker-password=<token> \
  -n ocp-ns-dashboard

oc secrets link ocp-ns-dashboard-sa quay-pull-secret \
  --for=pull -n ocp-ns-dashboard
```

### Disconnected / air-gapped clusters

Mirror base images before building:

```bash
skopeo copy \
  docker://registry.access.redhat.com/ubi9/nodejs-20:latest \
  docker://registry.internal.example.com/ubi9/nodejs-20:latest

skopeo copy \
  docker://registry.access.redhat.com/ubi9/nodejs-20-minimal:latest \
  docker://registry.internal.example.com/ubi9/nodejs-20-minimal:latest
```

Update `FROM` lines in `Containerfile` to point to your internal registry.

---

## Configuration

### Environment Variables

| Variable | Default | Description |
|---|---|---|
| `PORT` | `3000` | HTTP listen port |
| `NODE_ENV` | `production` | Node environment |
| `APP_GROUP_LABEL` | `appgroup` | Namespace label key for group filter |
| `KUBERNETES_SERVICE_HOST` | auto (in-cluster) | K8s API server hostname |
| `KUBERNETES_SERVICE_PORT` | auto (in-cluster) | K8s API server port |

### Customising the label key

If your team uses a different label key (e.g. `environment`, `team`, `squad`):

```yaml
# In all-in-one.yaml Deployment env section
- name: APP_GROUP_LABEL
  value: environment   # namespaces labelled environment=prod / environment=dev
```

```yaml
# In Helm values-production.yaml
extraEnv:
  - name: APP_GROUP_LABEL
    value: environment
```

---

## AppGroup Label Filter

The dashboard reads the `appgroup` label from each namespace to enable group-based filtering.

### Label namespaces during onboarding

```bash
# Label existing namespace
oc label namespace payments appgroup=apple
oc label namespace inventory appgroup=apple
oc label namespace billing appgroup=mango

# Label at creation
oc create namespace analytics
oc label namespace analytics appgroup=mango

# Verify
oc get namespaces -l appgroup=apple
oc get namespaces --show-labels | grep appgroup
```

### How it works in the dashboard

1. **App Groups section** (collapsible, minimised by default) appears in the main area between summary tiles and the node strip.
2. Click to expand — shows one chip per distinct `appgroup` value found in the cluster.
3. Namespaces with **no `appgroup` label** appear under the **No Label** chip.
4. Clicking a chip filters namespace cards to show only that group.
5. **Executive summary** appears immediately — shows namespace count, deployment count, StatefulSet count, and full pod status breakdown for the selected group.
6. Pod status filters (Running / Pending / etc.) still apply **within** the selected group — they do not collapse the namespace list.
7. Click **✕ Clear selection** to reset the appgroup filter.

### Filter logic separation

```
Namespace-level filters (independent):
  ├── Namespace name search (comma-separated)
  ├── Health chip (Healthy / Warning / Critical)
  └── AppGroup chip — NEVER depends on pod status

Pod-level filters (applied within visible namespaces):
  ├── Pod name search
  ├── Pod status chip (Running / CrashLoop / ImagePull / etc.)
  └── Node filter
```

Selecting an appgroup shows **all namespaces** in that group regardless of pod health. Pod status filters then apply as a second layer within those namespaces.

---

## RBAC — What the Dashboard Can Access

The `ocp-ns-dashboard-viewer` ClusterRole grants exclusively:

| API Group | Resource | Verbs |
|---|---|---|
| `""` (core) | `namespaces` | get, list, watch |
| `""` (core) | `pods` | get, list, watch |
| `""` (core) | `nodes` | get, list, watch |
| `apps` | `deployments` | get, list, watch |
| `apps` | `statefulsets` | get, list, watch |

**No write, delete, exec, escalate, or secret access.**

Verify permissions:

```bash
# Should return "yes"
oc auth can-i list pods \
  --as=system:serviceaccount:ocp-ns-dashboard:ocp-ns-dashboard-sa \
  --all-namespaces

# Should return "no"
oc auth can-i delete pods \
  --as=system:serviceaccount:ocp-ns-dashboard:ocp-ns-dashboard-sa \
  --all-namespaces

oc auth can-i get secrets \
  --as=system:serviceaccount:ocp-ns-dashboard:ocp-ns-dashboard-sa \
  --all-namespaces
```

---

## Rollback

### Option A — Imperative (uses K8s revision history)

```bash
# View available revisions
oc rollout history deployment/ocp-ns-dashboard -n ocp-ns-dashboard

# Roll back one revision
oc rollout undo deployment/ocp-ns-dashboard -n ocp-ns-dashboard

# Roll back to specific revision
oc rollout undo deployment/ocp-ns-dashboard \
  --to-revision=2 -n ocp-ns-dashboard
```

### Option B — Declarative (pin to previous image tag)

```bash
# Edit rollback.yaml — set image tag to previous known-good version
vi k8s/rollback.yaml

# Apply
oc apply -f k8s/rollback.yaml
oc rollout status deploy/ocp-ns-dashboard -n ocp-ns-dashboard
```

---

## Cleanup

```bash
# Make executable (first time only)
chmod +x k8s/cleanup.sh

# Dry-run — see what would be deleted
./k8s/cleanup.sh --dry-run

# Interactive — prompts for confirmation
./k8s/cleanup.sh

# Force — no prompt (CI/automation)
./k8s/cleanup.sh --force
```

The script handles deletion in dependency order: Route → Service → Deployment → RBAC → ServiceAccount → Namespace. Automatically force-removes finalizers if namespace gets stuck in `Terminating`.

---

## Local Development

```bash
# Install dependencies
npm install

# Set credentials (reads from current oc login)
export K8S_TOKEN=$(oc whoami -t)
export KUBERNETES_SERVICE_HOST=$(oc config view --minify \
  -o jsonpath='{.clusters[0].cluster.server}' | sed 's|https://||' | cut -d: -f1)
export KUBERNETES_SERVICE_PORT=6443
export APP_GROUP_LABEL=appgroup   # optional override

# Start server
npm start

# Open browser
open http://localhost:3000
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Pod stuck in `Forbidden` / SCC error | Hardcoded `runAsUser` | Remove `runAsUser` and `fsGroup` from the Deployment spec — OCP `restricted-v2` assigns namespace UID automatically |
| `npm error EACCES: permission denied, mkdir '/app/node_modules'` | UBI9 nodejs image non-root user | Use `USER root` in builder stage before `npm install`, then `chown` in runtime stage after all copies |
| `npm ci` fails with "package-lock.json not found" | No lockfile in repo | Use `npm install` instead of `npm ci` in the Containerfile |
| Dashboard shows empty grid | SA token can't reach API | Check `oc logs -l app=ocp-ns-dashboard -n ocp-ns-dashboard` and verify RBAC with `oc auth can-i` |
| AppGroup filter not visible | No `appgroup` labels on any namespace | Label at least one namespace: `oc label namespace <ns> appgroup=<value>` |
| Namespace card missing | Namespace excluded by system filter | Ensure namespace name doesn't start with `openshift-` or `kube-`, and isn't `default` |
| Route returns 503 | Pod not ready | `oc rollout status deploy/ocp-ns-dashboard -n ocp-ns-dashboard` |

---

## File Structure

```
ocp-ns-dashboard/
├── Containerfile                  # 2-stage UBI9 build
├── package.json
├── src/
│   └── server.js                  # Express + K8s HTTPS API client
├── public/
│   └── index.html                 # Red Hat SPA dashboard
├── k8s/
│   ├── all-in-one.yaml            # Single-file direct install
│   ├── rollback.yaml              # Declarative rollback manifest
│   └── cleanup.sh                 # Full teardown script
└── helm/
    ├── ocp-ns-dashboard/
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   └── templates/
    │       ├── _helpers.tpl
    │       ├── NOTES.txt
    │       ├── namespace.yaml
    │       ├── serviceaccount.yaml
    │       ├── rbac.yaml
    │       ├── deployment.yaml
    │       ├── service.yaml
    │       └── route.yaml
    └── values-production.yaml     # Production override example
```

---

*OpenShift Workload Monitoring Dashboard — Platform SRE Tooling*
*Native K8s API · restricted-v2 SCC · Red Hat UBI9 · No oc binary*
