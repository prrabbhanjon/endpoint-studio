# apitrace-k8s

> Live Kubernetes API communication dashboard

See the full setup guide in [docs/SETUP.md](docs/SETUP.md)

## Quick start

```bash
# 1. Clone
git clone https://github.com/prrabbhanjon/apitrace-k8s

# 2. Launch Ubuntu VM (Mac)
multipass launch --name apitrace --cpus 2 --memory 4G --disk 20G
multipass shell apitrace

# 3. Install tools (ARM64)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs podman podman-docker curl git

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-arm64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

export KIND_EXPERIMENTAL_PROVIDER=podman
kind create cluster --name apitrace-k8s

# 4. Build and deploy
podman build -f k8s-deploy/Dockerfile.backend -t apitrace-backend:local ./backend
podman build -f k8s-deploy/Dockerfile.frontend -t apitrace-frontend:local ./frontend
podman save -o /tmp/apitrace-backend.tar apitrace-backend:local
podman save -o /tmp/apitrace-frontend.tar apitrace-frontend:local
kind load image-archive --name apitrace-k8s /tmp/apitrace-backend.tar
kind load image-archive --name apitrace-k8s /tmp/apitrace-frontend.tar
kubectl apply -k k8s-deploy/
kubectl port-forward svc/apitrace-frontend 8080:80 -n apik8s-trace
```

## What it traces

All 12 API calls when a pod is created:
kubectl → API server → etcd → controller-mgr → scheduler → etcd → kubelet → CRI → registry → API server → etcd → all watchers

## Stack

TypeScript · React 18 · React Flow · Recharts · Node.js · Fastify · Podman · Kind · Prometheus · Grafana · Jaeger · Loki · HAProxy

## License

MIT
