# DevOps Practical Assignment — Harbor + k3d + Argo CD

A complete GitOps deployment pipeline using Harbor OCI registry, k3d Kubernetes cluster, and Argo CD — running entirely on a local Windows 10 machine via Docker Desktop + WSL2.

## Stack

| Component | Version | Role |
|-----------|---------|------|
| Harbor | v2.14.4 | OCI registry for Docker images and Helm charts |
| k3d | latest | k3s Kubernetes cluster inside Docker |
| Argo CD | stable | GitOps continuous delivery |
| Helm | v3 | Application packaging and deployment |
| Traefik | built-in | Ingress controller (k3d default) |
| cert-manager | latest | Automatic TLS certificate management |

## Architecture

```
Windows 10 + Docker Desktop (WSL2)
│
├── Docker Engine
│   ├── Harbor (docker compose)
│   │   ├── harbor.local:8443  (HTTPS)
│   │   ├── project: library
│   │   │   ├── solar-system:v9        (Docker image)
│   │   │   └── solar-system:0.1.2     (Helm chart OCI)
│   │
│   └── k3d cluster (k3s in Docker)
│       ├── loadbalancer  → ports 80/443
│       ├── server-0 node
│       └── agent-0 node
│           ├── namespace: cert-manager
│           │   └── cert-manager + ClusterIssuer (selfsigned)
│           ├── namespace: argocd
│           │   └── Argo CD
│           ├── namespace: dev
│           │   └── solar-system (1 replica, HTTPS)
│           └── namespace: prod
│               └── solar-system (2 replicas, HTTPS)
```

## Repository Structure

```
.
├── harbor/
│   └── harbor.yml                  # Harbor configuration
├── solar-system-chart/
│   └── solar-system/               # Helm chart
│       ├── Chart.yaml
│       ├── values.yaml             # default values
│       ├── values-dev.yaml         # dev environment overrides
│       ├── values-prod.yaml        # prod environment overrides
│       └── templates/
│           ├── _helpers.tpl
│           ├── deployment.yaml
│           ├── service.yaml
│           └── ingress.yaml
├── argocd/
│   ├── app-dev.yaml                # Argo CD Application for dev
│   └── app-prod.yaml               # Argo CD Application for prod
├── k3d/
│   └── registries.yaml             # k3d registry config
│   └── cluster-issuer.yaml         # cert-manager ClusterIssuer
├── README.md
└── screenshots/                    # Screenshots of harbor, argocd, web app ui with valid certs
```

## Prerequisites

- Windows 10 with Docker Desktop (WSL2 backend)
- WSL2 with Ubuntu 24
- Tools installed in WSL2: `kubectl`, `helm`, `k3d`, `argocd` CLI, `openssl`

## Step-by-Step Setup

### 1. Generate SSL Certificate

```bash
mkdir -p harbor/certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout harbor/certs/key.pem \
  -out harbor/certs/cert.pem \
  -subj "/CN=harbor.local" \
  -addext "subjectAltName=DNS:harbor.local,IP:127.0.0.1"
```

### 2. Deploy Harbor

Configure `harbor/harbor.yml`:

```yaml
hostname: harbor.local
http:
  port: 8080
https:
  port: 8443
  certificate: /hostfs/path/to/harbor/certs/cert.pem
  private_key: /hostfs/path/to/harbor/certs/key.pem
```

```bash
cd harbor
./prepare
./install.sh
docker compose up -d
```

Add to Windows hosts (`C:\Windows\System32\drivers\etc\hosts`):
```
127.0.0.1 harbor.local
```

Harbor UI available at: `https://harbor.local:8443`

### 3. Push Docker Image to Harbor

```bash
# Import cert into Windows Certificate Store
Import-Certificate -FilePath "harbor\certs\cert.pem" `
  -CertStoreLocation Cert:\LocalMachine\Root

# Pull, retag, push
docker pull siddharth67/solar-system:v9
docker tag siddharth67/solar-system:v9 harbor.local:8443/library/solar-system:v9
docker login harbor.local:8443
docker push harbor.local:8443/library/solar-system:v9
```

### 4. Deploy k3d Cluster

Create k3d cluster:

```bash
k3d cluster create k3d-main \
  --port "80:80@loadbalancer" \
  --port "443:443@loadbalancer" \
  --agents 1
```

Connect networks of k3d and harbor:

```bash
docker network connect k3d-k3d-main nginx
docker network connect k3d-k3d-main harbor-core
docker network connect k3d-k3d-main registry
```

Get IP of harbor nginx container in `k3d-k3d-main` network:

```
docker inspect nginx
```

Add `harbor.local` to CoreDNS with the IP that was found before:

```bash
KUBE_EDITOR="nano" kubectl edit configmap coredns -n kube-system
# Add to NodeHosts section:
# 172.18.0.6 harbor.local       <-- use your actual IP
```

Copy Harbor certificate to k3d nodes so containerd can pull images:

```bash
docker exec k3d-k3d-main-server-0 mkdir -p /etc/ssl/certs/harbor
docker exec k3d-k3d-main-agent-0 mkdir -p /etc/ssl/certs/harbor

docker cp harbor/certs/cert.pem k3d-k3d-main-server-0:/etc/ssl/certs/harbor/ca.crt
docker cp harbor/certs/cert.pem k3d-k3d-main-agent-0:/etc/ssl/certs/harbor/ca.crt
```

Create `/etc/rancher/k3s/registries.yaml` on k3d-k3d-main-server-0 and k3d-k3d-main-agent-0:

```yaml
mirrors:
  "harbor.local:8443":
    endpoint:
      - "https://harbor.local:8443"
configs:
  "harbor.local:8443":
    tls:
      ca_file: /etc/ssl/certs/harbor/ca.crt
```

### 5. Deploy Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Access UI
kubectl port-forward svc/argocd-server -n argocd 9090:443
# Open: https://localhost:9090
```

### 6. Connect Harbor to Argo CD

```bash
argocd login localhost:9090 --username admin --insecure

argocd repo add harbor.local:8443/library \
  --type helm \
  --name harbor-library \
  --enable-oci \
  --username admin \
  --password <password> \
  --insecure-skip-server-verification

argocd repo list   # STATUS should be Successful
```

### 7. Package and Push Helm Chart

```bash
cd solar-system-chart

helm lint solar-system/

helm package solar-system/

helm registry login harbor.local:8443 \
  --username admin \
  --password <password> \
  --insecure

helm push solar-system-0.1.1.tgz oci://harbor.local:8443/library \
  --ca-file ../harbor/certs/cert.pem
```

### 8. Deploy to dev and prod via Argo CD

```bash
# Create namespaces
kubectl create namespace dev
kubectl create namespace prod

# Deploy applications
kubectl apply -f argocd/app-dev.yaml
kubectl apply -f argocd/app-prod.yaml

# Verify
argocd app list
kubectl get pods -n dev
kubectl get pods -n prod
```

### 9. Setup TLS for Ingress with cert-manager

Install cert-manager:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

Create cluster-issuer with file `k3d/cluster-issuer.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

Verify it (READY should be True):

```bash
kubectl get clusterissuer
```

Verify certificates were issued:

```bash
kubectl get certificate -n dev
kubectl get certificate -n prod
# READY should be True
```

Add to Windows hosts (`C:\Windows\System32\drivers\etc\hosts`):
```
127.0.0.1 solar-system-dev.local
127.0.0.1 solar-system-prod.local
```

Applications available at:
- `https://solar-system-dev.local` — dev environment (1 replica)
- `https://solar-system-prod.local` — prod environment (2 replicas)

Import the CA cert into Windows Certificate Store to suppress a warning in a browser:

```powershell
# PowerShell as Admin
kubectl get secret -n dev solar-system-dev-solar-system-tls \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > selfsigned-ca-dev.crt

kubectl get secret -n dev solar-system-prod-solar-system-tls \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > selfsigned-ca-dev.crt

Import-Certificate -FilePath "selfsigned-ca-dev.crt" `
  -CertStoreLocation Cert:\LocalMachine\Root

Import-Certificate -FilePath "selfsigned-ca-prod.crt" `
  -CertStoreLocation Cert:\LocalMachine\Root
```

After that, you can see a message stating that the certificates are valid in a browser.

## Helm Chart Overview

The chart deploys the following Kubernetes resources:

**Deployment** — with resource requests/limits and health probes:
```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "100m"
  limits:
    memory: "128Mi"
    cpu: "200m"
livenessProbe:
  httpGet:
    path: /
    port: 80
readinessProbe:
  httpGet:
    path: /
    port: 80
```

**Service** — ClusterIP on port 80

**Ingress** — Traefik ingress with TLS termination via cert-manager

## Environment Differences

| Parameter | dev | prod |
|-----------|-----|------|
| Replicas | 1 | 2 |
| CPU request | 50m | 100m |
| Memory request | 32Mi | 64Mi |
| CPU limit | 100m | 200m |
| Memory limit | 64Mi | 128Mi |
| Ingress host | solar-system-dev.local | solar-system-prod.local |
| TLS | enabled (self-signed) | enabled (self-signed) |

## Argo CD Applications

Both applications use automated sync with self-heal and pruning enabled:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

## Notes

- Harbor runs outside the k3d cluster via docker compose
- k3d nodes access Harbor through the Docker bridge network (`172.18.0.x`)
- CoreDNS NodeHosts entry ensures Harbor DNS resolution inside the cluster
- SSL certificate uses SAN (Subject Alternative Name) as required by modern TLS clients
- Helm charts are stored as OCI artifacts in Harbor
- cert-manager automates the full TLS certificate lifecycle: issuance, storage as Secret, and renewal