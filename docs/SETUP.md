# apitrace-k8s — Complete Setup & Architecture Guide

> Live Kubernetes API communication dashboard — Ubuntu ARM64 · Podman · Kind · React · Node.js

---

## Table of Contents

- [What it does](#what-it-does)
- [Architecture](#architecture)
- [API Communication Chain](#api-communication-chain)
- [Tech Stack](#tech-stack)
- [Part 1 — Mac host setup](#part-1--mac-host-setup-multipass-vm)
- [Part 2 — Ubuntu VM tools](#part-2--install-tools-inside-ubuntu-vm)
- [Part 3 — Kubernetes cluster](#part-3--create-kubernetes-cluster)
- [Part 4 — Project structure](#part-4--project-structure)
- [Part 5 — Build & deploy](#part-5--build--deploy)
- [Part 6 — Observability](#part-6--observability-endpoints)
- [Part 7 — Security](#part-7--security)
- [Part 8 — Useful commands](#part-8--useful-commands)
- [Part 9 — Teardown](#part-9--teardown)

---

## What it does

apitrace-k8s is an open-source live dashboard that traces every API call across your entire Kubernetes infrastructure stack — from the moment you run `kubectl apply` to the pod reaching `Running` status. It also integrates Prometheus, Grafana, Jaeger, Loki, and HAProxy into a single real-time view.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        INTERNET / CLIENT                            │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 1 — EDGE                                                     │
│                                                                     │
│   ┌─────────────┐    ┌──────────────┐    ┌────────────────────┐   │
│   │   HAProxy   │───▶│ TLS/cert-mgr │───▶│   Rate Limiting    │   │
│   │  TCP/HTTP   │    │  Let's Encrypt│   │   100 req/min      │   │
│   └─────────────┘    └──────────────┘    └────────────────────┘   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 2 — ROUTING                                                  │
│                                                                     │
│   ┌─────────────────┐    ┌──────────────┐    ┌─────────────────┐  │
│   │ Nginx Ingress   │───▶│  Gateway API │───▶│   API Gateway   │  │
│   │ Controller      │    │  HTTPRoute   │    │  Kong / APISIX  │  │
│   └─────────────────┘    └──────────────┘    └─────────────────┘  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 3 — KUBERNETES CONTROL PLANE                                 │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                    API SERVER (:6443)                       │  │
│   │          Central hub — all communication flows here         │  │
│   └──────┬──────────────┬──────────────┬───────────────┬───────┘  │
│          │              │              │               │            │
│          ▼              ▼              ▼               ▼            │
│   ┌──────────┐  ┌──────────────┐  ┌──────────┐  ┌──────────┐    │
│   │   etcd   │  │  Controller  │  │ Scheduler│  │  kubelet  │   │
│   │  (state) │  │   Manager    │  │ (binding)│  │  (nodes)  │   │
│   └──────────┘  └──────────────┘  └──────────┘  └─────┬────┘    │
│                                                         │          │
│                                                         ▼          │
│                                                  ┌──────────────┐  │
│                                                  │CRI/containerd│  │
│                                                  │   (runc)     │  │
│                                                  └──────┬───────┘  │
│                                                         │           │
│                                                         ▼           │
│                                                  ┌──────────────┐  │
│                                                  │     PODS     │  │
│                                                  │  apik8s-trace│  │
│                                                  └──────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 4 — OBSERVABILITY                                            │
│                                                                     │
│  ┌────────────┐  ┌─────────┐  ┌──────────────┐  ┌─────────────┐  │
│  │ Prometheus │  │  Loki   │  │    Jaeger    │  │   Grafana   │  │
│  │  PromQL    │  │  LogQL  │  │   TraceQL    │  │  Dashboards │  │
│  │  metrics   │  │   logs  │  │   traces     │  │   panels    │  │
│  └─────┬──────┘  └────┬────┘  └──────┬───────┘  └──────┬──────┘  │
│        └──────────────┴───────────────┴─────────────────┘          │
│                                       │                             │
│                                       ▼                             │
│                        ┌─────────────────────────┐                 │
│                        │   apitrace-k8s backend  │                 │
│                        │   Node.js + Fastify      │                 │
│                        │   WebSocket /ws/events   │                 │
│                        └──────────────┬──────────┘                 │
│                                       │                             │
│                                       ▼                             │
│                        ┌─────────────────────────┐                 │
│                        │  apitrace-k8s frontend  │                 │
│                        │  React + React Flow      │                 │
│                        │  Recharts + TypeScript   │                 │
│                        └─────────────────────────┘                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## API Communication Chain

Every time a pod is created, **12 API calls** fire in sequence. apitrace-k8s captures and visualizes all of them live:

```
┌────┬──────────────────────────────┬─────────────────────────────────────────┬──────────┐
│ #  │ API Call                     │ What happens                            │ Status   │
├────┼──────────────────────────────┼─────────────────────────────────────────┼──────────┤
│ 01 │ kubectl → API server         │ POST /api/v1/namespaces/default/pods    │ pending  │
│ 02 │ API server → etcd            │ Persist pod spec (phase: Pending)       │ pending  │
│ 03 │ API server → controller-mgr  │ Watch event: ADDED pod, reconcile RS    │ watch    │
│ 04 │ API server → scheduler       │ Watch event: pod.spec.nodeName == ""    │ watch    │
│ 05 │ Scheduler → API server       │ POST /binding — assign node: node-01   │ bound    │
│ 06 │ API server → etcd            │ Persist binding, nodeName=node-01      │ pending  │
│ 07 │ kubelet ← API server         │ Watch: pod bound to this node           │ watch    │
│ 08 │ kubelet → CRI/containerd     │ RunPodSandbox · CreateContainer (runc)  │ pending  │
│ 09 │ kubelet → registry           │ Image pull: docker.io/nginx:latest      │ pulling  │
│ 10 │ kubelet → API server         │ PATCH /pods/status → phase: Running     │ running  │
│ 11 │ API server → etcd            │ Persist pod status (Running, PodIP)     │ running  │
│ 12 │ API server → all watchers    │ Emit MODIFIED event to watch streams    │ emitted  │
└────┴──────────────────────────────┴─────────────────────────────────────────┴──────────┘
```

---

## Tech Stack

```
┌─────────────────────────────────────────────────────────────────────┐
│  FRONTEND                                                           │
│  TypeScript · React 18 · React Flow · Recharts · Vite · Inter font │
├─────────────────────────────────────────────────────────────────────┤
│  BACKEND                                                            │
│  TypeScript · Node.js 20 · Fastify · WebSocket · @k8s/client-node  │
├─────────────────────────────────────────────────────────────────────┤
│  OBSERVABILITY QUERY LANGUAGES                                      │
│  PromQL (Prometheus) · LogQL (Loki) · TraceQL (Jaeger/Tempo)       │
├─────────────────────────────────────────────────────────────────────┤
│  INFRASTRUCTURE                                                     │
│  HAProxy · Nginx Ingress · Gateway API · cert-manager · Helm        │
├─────────────────────────────────────────────────────────────────────┤
│  CONTAINER & ORCHESTRATION                                          │
│  Podman (rootless, daemonless) · Kind · Kubernetes 1.30            │
├─────────────────────────────────────────────────────────────────────┤
│  CI/CD                                                              │
│  GitHub Actions · GHCR (GitHub Container Registry)                 │
├─────────────────────────────────────────────────────────────────────┤
│  OS / VM                                                            │
│  Ubuntu 24.04 ARM64 · Multipass (on Apple Silicon Mac)             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 1 — Mac host setup (Multipass VM)

### 1.1 Install Multipass

```bash
brew install --cask multipass
```

### 1.2 Launch Ubuntu 24.04 VM (ARM64)

```bash
multipass launch --name apitrace --cpus 2 --memory 4G --disk 20G
```

### 1.3 Shell into the VM

```bash
multipass shell apitrace
```

> All commands from Part 2 onwards run **inside** the VM.

---

## Part 2 — Install tools inside Ubuntu VM

### 2.1 Update and install base packages

```bash
sudo apt-get update -y
sudo apt-get install -y curl git
```

### 2.2 Install Node.js 20

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
node --version
npm --version
```

### 2.3 Install kubectl — ARM64

> ⚠️ Always use `arm64` on Apple Silicon — `amd64` will give "Exec format error"

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

### 2.4 Install kind — ARM64

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

### 2.5 Install Podman (rootless, no daemon)

```bash
sudo apt-get install -y podman podman-docker
podman --version
```

> **Why Podman over Docker?**
> - No background root daemon — more secure
> - Rootless by default — no sudo needed
> - Drop-in Docker replacement — same CLI
> - OCI compliant — same image format

### 2.6 Configure kind to use Podman

```bash
export KIND_EXPERIMENTAL_PROVIDER=podman
echo 'export KIND_EXPERIMENTAL_PROVIDER=podman' >> ~/.bashrc
source ~/.bashrc
```

---

## Part 3 — Create Kubernetes cluster

### 3.1 Create Kind cluster

```bash
kind create cluster --name apitrace-k8s
```

Expected output:
```
✓ Ensuring node image (kindest/node:v1.30.0)
✓ Preparing nodes
✓ Writing configuration
✓ Starting control-plane
✓ Installing CNI
✓ Installing StorageClass
Set kubectl context to "kind-apitrace-k8s"
```

### 3.2 Verify cluster is healthy

```bash
kubectl cluster-info --context kind-apitrace-k8s
kubectl get nodes
kubectl get namespaces
```

Expected nodes output:
```
NAME                         STATUS   ROLES           AGE
apitrace-k8s-control-plane   Ready    control-plane   60s
```

---

## Part 4 — Project structure

```
apitrace-k8s/
│
├── frontend/                        # React dashboard (TypeScript)
│   ├── src/
│   │   ├── components/
│   │   │   ├── flow/                # React Flow — animated API graph
│   │   │   │   └── ApiFlowGraph.tsx
│   │   │   ├── charts/              # Recharts — activity sparklines
│   │   │   │   └── ActivityChart.tsx
│   │   │   ├── stream/              # Live event stream table
│   │   │   │   └── EventStream.tsx
│   │   │   ├── stats/               # Metric stat cards
│   │   │   │   └── StatCards.tsx
│   │   │   └── topology/            # Observability panel
│   │   │       └── ObservabilityPanel.tsx
│   │   ├── hooks/
│   │   │   └── usePodEvents.ts      # WebSocket live events
│   │   ├── pages/
│   │   │   └── LandingPage.tsx      # Main dashboard page
│   │   └── types/
│   │       └── index.ts             # Shared TypeScript types
│   ├── nginx.conf                   # Nginx config for prod image
│   ├── package.json
│   ├── vite.config.ts
│   └── tsconfig.json
│
├── backend/                         # Node.js API aggregator
│   ├── src/
│   │   └── index.ts                 # Fastify + K8s watch stream
│   ├── package.json
│   └── tsconfig.json
│
├── k8s-deploy/                      # All Kubernetes manifests
│   ├── namespace/
│   │   └── namespace.yaml           # apik8s-trace namespace
│   ├── rbac/
│   │   └── serviceaccount.yaml      # Read-only ClusterRole + binding
│   ├── configmaps/
│   │   └── backend-config.yaml      # Env vars for backend
│   ├── secrets/
│   │   └── secrets.yaml             # Grafana API key (base64)
│   ├── backend/
│   │   ├── deployment.yaml          # imagePullPolicy: IfNotPresent
│   │   └── service.yaml             # ClusterIP :3001
│   ├── frontend/
│   │   ├── deployment.yaml          # imagePullPolicy: IfNotPresent
│   │   └── service.yaml             # ClusterIP :80
│   ├── observability/
│   │   ├── prometheus/
│   │   │   ├── configmap.yaml       # PromQL scrape config
│   │   │   └── deployment.yaml      # prom/prometheus:v2.51.0
│   │   ├── grafana/
│   │   │   └── deployment.yaml      # grafana/grafana:10.4.0
│   │   ├── jaeger/
│   │   │   └── deployment.yaml      # jaegertracing/all-in-one:1.56
│   │   └── loki/
│   │       └── deployment.yaml      # grafana/loki:2.9.6
│   ├── ingress/
│   │   └── ingress.yaml             # Nginx ingress + security headers
│   ├── Dockerfile.backend           # Multi-stage node:20-alpine build
│   ├── Dockerfile.frontend          # Multi-stage nginx:1.25-alpine
│   └── kustomization.yaml           # Apply all at once
│
└── docs/
    └── SETUP.md                     # This file
```

---

## Part 5 — Build & deploy

### 5.1 Clone the repo

```bash
git clone https://github.com/prrabbhanjon/apitrace-k8s ~/apitrace-k8s
cd ~/apitrace-k8s
```

### 5.2 Build frontend npm dependencies

```bash
cd ~/apitrace-k8s/frontend
npm install
cd ~/apitrace-k8s
```

### 5.3 Build Podman images

```bash
# Backend — context is ./backend folder
podman build \
  -f k8s-deploy/Dockerfile.backend \
  -t apitrace-backend:local \
  ./backend

# Frontend — context is ./frontend folder (nginx.conf must be here)
podman build \
  -f k8s-deploy/Dockerfile.frontend \
  -t apitrace-frontend:local \
  ./frontend

# Confirm images built
podman images | grep apitrace
```

### 5.4 Load images into Kind cluster

```bash
# Save to tar archives
podman save -o /tmp/apitrace-backend.tar  apitrace-backend:local
podman save -o /tmp/apitrace-frontend.tar apitrace-frontend:local

# Load into kind-apitrace-k8s cluster
kind load image-archive --name apitrace-k8s /tmp/apitrace-backend.tar
kind load image-archive --name apitrace-k8s /tmp/apitrace-frontend.tar

# Verify images are inside the cluster
podman exec -it apitrace-k8s-control-plane crictl images | grep apitrace
```

### 5.5 Create namespace

```bash
kubectl apply -f k8s-deploy/namespace/namespace.yaml
kubectl get namespace apik8s-trace
```

### 5.6 Apply RBAC

```bash
kubectl apply -f k8s-deploy/rbac/serviceaccount.yaml
```

### 5.7 Apply ConfigMaps and Secrets

```bash
kubectl apply -f k8s-deploy/configmaps/
kubectl apply -f k8s-deploy/secrets/
```

### 5.8 Deploy all workloads

```bash
kubectl apply -k k8s-deploy/
```

### 5.9 Watch pods come up

```bash
kubectl get pods -n apik8s-trace -w
```

Expected final state:
```
NAME                                 READY   STATUS    RESTARTS   AGE
apitrace-backend-xxx                 1/1     Running   0          2m
apitrace-frontend-xxx                1/1     Running   0          2m
grafana-xxx                          1/1     Running   0          2m
jaeger-xxx                           1/1     Running   0          2m
loki-xxx                             1/1     Running   0          2m
prometheus-xxx                       1/1     Running   0          2m
```

### 5.10 Access the dashboard

```bash
kubectl port-forward svc/apitrace-frontend 8080:80 -n apik8s-trace
```

**From Mac**, get the VM IP:
```bash
multipass info apitrace | grep IPv4
```

Then open: **http://VM_IP:8080**

---

## Part 6 — Observability endpoints

Port-forward each service to access from your browser:

```bash
# Prometheus UI + PromQL
kubectl port-forward svc/prometheus 9090:9090 -n apik8s-trace
# → http://localhost:9090

# Grafana dashboards  (login: admin / apitrace-admin)
kubectl port-forward svc/grafana 3000:3000 -n apik8s-trace
# → http://localhost:3000

# Jaeger distributed traces
kubectl port-forward svc/jaeger 16686:16686 -n apik8s-trace
# → http://localhost:16686

# Loki log queries
kubectl port-forward svc/loki 3100:3100 -n apik8s-trace
# → http://localhost:3100
```

### Key PromQL queries

```promql
# Scheduler bind latency P99
histogram_quantile(0.99, sum(rate(scheduler_binding_duration_seconds_bucket[5m])) by (le))

# API server request rate by verb
sum(rate(apiserver_request_total[1m])) by (verb)

# etcd request latency P99
histogram_quantile(0.99, sum(rate(etcd_request_duration_seconds_bucket[5m])) by (le, operation))

# Kubelet pod start time P99
histogram_quantile(0.99, sum(rate(kubelet_pod_start_duration_seconds_bucket[5m])) by (le))

# Pod phase counts
count by (phase) (kube_pod_status_phase{phase=~"Running|Pending|Failed"})
```

---

## Part 7 — Security

```
┌─────────────────────────────────────────────────────────────────────┐
│  SECURITY CONTROLS                                                  │
├──────────────────────┬──────────────────────────────────────────────┤
│ HTTP Headers         │ Content-Security-Policy                      │
│                      │ Strict-Transport-Security (HSTS 2yr)         │
│                      │ X-Frame-Options: DENY                        │
│                      │ X-Content-Type-Options: nosniff              │
│                      │ Referrer-Policy: strict-origin               │
│                      │ Permissions-Policy: camera=(), mic=()        │
├──────────────────────┼──────────────────────────────────────────────┤
│ Kubernetes RBAC      │ Read-only ClusterRole                        │
│                      │ verbs: get, list, watch ONLY                 │
│                      │ No create / update / delete / patch          │
├──────────────────────┼──────────────────────────────────────────────┤
│ Container security   │ Non-root user (UID 1000)                     │
│                      │ readOnlyRootFilesystem: true                 │
│                      │ allowPrivilegeEscalation: false              │
│                      │ capabilities: drop ALL                       │
├──────────────────────┼──────────────────────────────────────────────┤
│ Secrets              │ Environment variables only                   │
│                      │ Never in source code or image layers         │
│                      │ Kubernetes Secrets (base64 encoded)          │
├──────────────────────┼──────────────────────────────────────────────┤
│ Network              │ CORS allow-list — no wildcard *              │
│                      │ WebSocket origin check                       │
│                      │ Rate limit: 100 req/min per IP               │
│                      │ Rate limit: 10 req/min on feedback endpoint  │
├──────────────────────┼──────────────────────────────────────────────┤
│ Images               │ Multi-stage build — minimal Alpine base      │
│                      │ No dev dependencies in production image      │
│                      │ Final image < 50MB                           │
└──────────────────────┴──────────────────────────────────────────────┘
```

### RBAC manifest

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: apitrace-k8s-reader
rules:
  - apiGroups: [""]
    resources: ["pods","nodes","events","services"]
    verbs: ["get","list","watch"]   # read-only
  - apiGroups: ["gateway.networking.k8s.io"]
    resources: ["gateways","httproutes"]
    verbs: ["get","list","watch"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get","list","watch"]
```

---

## Part 8 — Useful commands

```bash
# All resources in namespace
kubectl get all -n apik8s-trace

# Watch pod status live
kubectl get pods -n apik8s-trace -w

# Stream backend logs
kubectl logs -f -l app=apitrace-backend -n apik8s-trace

# Stream frontend logs
kubectl logs -f -l app=apitrace-frontend -n apik8s-trace

# Describe a failing pod
kubectl describe pod -l app=apitrace-backend -n apik8s-trace

# Check image pull events
kubectl get events -n apik8s-trace --sort-by='.lastTimestamp'

# Exec into backend pod
kubectl exec -it -n apik8s-trace \
  $(kubectl get pod -l app=apitrace-backend -n apik8s-trace -o name | head -1) \
  -- sh

# Restart a deployment
kubectl rollout restart deployment/apitrace-backend -n apik8s-trace

# Check cluster images (inside kind)
podman exec -it apitrace-k8s-control-plane crictl images

# Check current K8s context
kubectl config current-context

# List all contexts
kubectl config get-contexts
```

---

## Part 9 — Teardown

```bash
# Delete just the namespace + all its resources
kubectl delete namespace apik8s-trace

# Delete entire Kind cluster
kind delete cluster --name apitrace-k8s

# Remove Podman images
podman rmi apitrace-backend:local apitrace-frontend:local

# Stop Multipass VM (from Mac terminal)
multipass stop apitrace

# Delete Multipass VM entirely (from Mac terminal)
multipass delete apitrace && multipass purge
```

---

## Common errors and fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Exec format error` | Wrong arch binary | Use `arm64` on Apple Silicon |
| `ImagePullBackOff` | Image not in cluster | Run `kind load image-archive` |
| `spec.selector immutable` | Old deployment exists | Delete deployment first, re-apply |
| `npm ci` fails | No package-lock.json | Use `npm install` in Dockerfile |
| `nginx.conf not found` | Wrong build context | nginx.conf must be in `./frontend/` |
| `context must be a directory` | Folder doesn't exist | Run `mkdir -p backend/src frontend/src` |
| Pod stuck `Pending` | No resources | Check `kubectl describe pod` |

---

## GitHub

- **Repo:** https://github.com/prrabbhanjon/apitrace-k8s
- **Issues:** https://github.com/prrabbhanjon/apitrace-k8s/issues
- **License:** MIT

---

*Built with ❤️ — open source, free to use and modify*
