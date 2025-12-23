# Kubernetes Deployment

Kubernetes manifests for deploying the video platform to a cluster.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         KUBERNETES CLUSTER                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         Ingress (nginx)                              │    │
│  │  video.example.com → video-dashboard                                │    │
│  │  api.video.example.com → video-platform                             │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                               │                                              │
│         ┌─────────────────────┴─────────────────────┐                       │
│         │                                           │                       │
│         ▼                                           ▼                       │
│  ┌─────────────────────┐                 ┌─────────────────────┐           │
│  │  video-dashboard    │                 │   video-platform    │           │
│  │  (Deployment)       │                 │   (Deployment)      │           │
│  │  replicas: 2        │                 │   replicas: 2-3     │           │
│  │  nginx + React SPA  │────────────────▶│   Go API server     │           │
│  └─────────────────────┘   /api proxy    └──────────┬──────────┘           │
│                                                      │                       │
│                                                      ▼                       │
│                                           ┌─────────────────────┐           │
│                                           │     PostgreSQL      │           │
│                                           │   (StatefulSet)     │           │
│                                           │   replicas: 1       │           │
│                                           └─────────────────────┘           │
│                                                      │                       │
│  ┌───────────────────────────────────────────────────┴───────────────────┐  │
│  │                    Persistent Volumes                                  │  │
│  │  postgres-data (10-50Gi)    clips-pvc (50-200Gi)                      │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Structure

```
k8s/
├── base/                      # Base manifests
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── secrets.yaml
│   ├── postgres.yaml
│   ├── video-platform.yaml
│   ├── video-dashboard.yaml
│   └── ingress.yaml
│
└── overlays/
    ├── staging/               # Staging environment
    │   ├── kustomization.yaml
    │   └── ingress-patch.yaml
    │
    └── production/            # Production environment
        ├── kustomization.yaml
        └── ingress-patch.yaml
```

## Prerequisites

1. **Kubernetes cluster** (1.25+)
2. **kubectl** configured
3. **Ingress controller** (nginx-ingress recommended)
4. **cert-manager** (for TLS certificates)
5. **Container images** pushed to registry

## Quick Start

### Deploy to Staging

```bash
# Preview what will be deployed
kubectl kustomize k8s/overlays/staging

# Apply to cluster
kubectl apply -k k8s/overlays/staging

# Check status
kubectl -n video-platform-staging get pods
```

### Deploy to Production

```bash
# Update secrets first!
kubectl -n video-platform create secret generic video-platform-secrets \
  --from-literal=POSTGRES_PASSWORD='your-secure-password' \
  --from-literal=DATABASE_URL='postgres://video:your-secure-password@postgres:5432/video_platform?sslmode=disable' \
  --dry-run=client -o yaml | kubectl apply -f -

# Deploy
kubectl apply -k k8s/overlays/production

# Check status
kubectl -n video-platform get pods
```

## Configuration

### Update Domain Names

Edit `k8s/overlays/{staging,production}/ingress-patch.yaml`:

```yaml
spec:
  tls:
    - hosts:
        - your-domain.com
        - api.your-domain.com
      secretName: video-platform-tls
  rules:
    - host: your-domain.com
```

### Update Secrets

**Never commit real secrets!** Update in cluster:

```bash
kubectl -n video-platform create secret generic video-platform-secrets \
  --from-literal=POSTGRES_PASSWORD='secure-password-here' \
  --from-literal=DATABASE_URL='postgres://video:secure-password-here@postgres:5432/video_platform?sslmode=disable' \
  --dry-run=client -o yaml | kubectl apply -f -
```

Or use external secrets manager (Vault, AWS Secrets Manager, etc.)

### Update Image Tags

Edit `k8s/overlays/{staging,production}/kustomization.yaml`:

```yaml
images:
  - name: ghcr.io/prodro21/video-platform
    newTag: v1.2.0
  - name: ghcr.io/prodro21/video-dashboard
    newTag: v1.2.0
```

## Components

### PostgreSQL

- **Type:** StatefulSet (single replica)
- **Storage:** 10Gi (staging) / 50Gi (production)
- **Credentials:** From secrets

### video-platform

- **Type:** Deployment
- **Replicas:** 1 (staging) / 3 (production)
- **Port:** 8080
- **Health:** `/health` endpoint
- **Storage:** clips-pvc for video files

### video-dashboard

- **Type:** Deployment
- **Replicas:** 1 (staging) / 2 (production)
- **Port:** 80
- **Proxy:** Routes `/api/*` to video-platform

### Ingress

- **Controller:** nginx
- **TLS:** cert-manager with Let's Encrypt
- **Hosts:** Configurable per environment

## Operations

### View Logs

```bash
# All platform logs
kubectl -n video-platform logs -l app=video-platform -f

# All dashboard logs
kubectl -n video-platform logs -l app=video-dashboard -f

# PostgreSQL logs
kubectl -n video-platform logs -l app=postgres -f
```

### Scale

```bash
# Scale platform
kubectl -n video-platform scale deployment video-platform --replicas=5

# Scale dashboard
kubectl -n video-platform scale deployment video-dashboard --replicas=3
```

### Database Access

```bash
# Port forward
kubectl -n video-platform port-forward svc/postgres 5432:5432

# Connect
psql -h localhost -U video -d video_platform
```

### Backup Database

```bash
# Create backup
kubectl -n video-platform exec -it postgres-0 -- \
  pg_dump -U video video_platform > backup.sql

# Restore
kubectl -n video-platform exec -i postgres-0 -- \
  psql -U video video_platform < backup.sql
```

## Building Images

### video-platform

```bash
cd video-platform
docker build -t ghcr.io/prodro21/video-platform:v1.0.0 .
docker push ghcr.io/prodro21/video-platform:v1.0.0
```

### video-dashboard

```bash
cd video-dashboard
docker build -t ghcr.io/prodro21/video-dashboard:v1.0.0 .
docker push ghcr.io/prodro21/video-dashboard:v1.0.0
```

## Monitoring

### Check Health

```bash
# Platform health
kubectl -n video-platform exec -it deploy/video-platform -- wget -qO- http://localhost:8080/health

# Dashboard health
kubectl -n video-platform exec -it deploy/video-dashboard -- wget -qO- http://localhost/nginx-health
```

### Resource Usage

```bash
kubectl -n video-platform top pods
```

## Troubleshooting

### Pods Not Starting

```bash
# Check events
kubectl -n video-platform get events --sort-by='.lastTimestamp'

# Describe pod
kubectl -n video-platform describe pod <pod-name>
```

### Database Connection Issues

```bash
# Check PostgreSQL is ready
kubectl -n video-platform exec -it postgres-0 -- pg_isready -U video

# Check connection from platform
kubectl -n video-platform exec -it deploy/video-platform -- \
  sh -c 'echo "SELECT 1" | psql $DATABASE_URL'
```

### Ingress Not Working

```bash
# Check ingress status
kubectl -n video-platform describe ingress video-platform-ingress

# Check cert-manager certificates
kubectl -n video-platform get certificates
```
