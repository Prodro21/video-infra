# Video Platform Infrastructure

Infrastructure, deployment configurations, and documentation for the video streaming platform.

## Contents

```
video-infra/
├── k8s/                    # Kubernetes manifests
│   ├── base/               # Base configurations
│   └── overlays/           # Environment-specific (staging, production)
├── ARCHITECTURE.md         # System architecture overview
├── API.md                  # REST API reference
├── DEPLOYMENT.md           # Deployment guide
└── WORKFLOWS.md            # Component interaction flows
```

## Quick Start

### Deploy to Kubernetes

```bash
# Staging
kubectl apply -k k8s/overlays/staging

# Production
kubectl apply -k k8s/overlays/production
```

See [k8s/README.md](k8s/README.md) for detailed Kubernetes instructions.

## Documentation

| Document | Description |
|----------|-------------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | System components and structure |
| [API.md](API.md) | Complete REST API reference |
| [DEPLOYMENT.md](DEPLOYMENT.md) | Local, Docker, and K8s deployment |
| [WORKFLOWS.md](WORKFLOWS.md) | Data flows and component interactions |

## Related Repositories

| Repository | Description |
|------------|-------------|
| [video-platform](https://github.com/Prodro21/video-platform) | Core API server |
| [video-dashboard](https://github.com/Prodro21/video-dashboard) | Admin web UI |
| [video-ipad](https://github.com/Prodro21/video-ipad) | iPad app for coaches |
| [video-mcp](https://github.com/Prodro21/video-mcp) | MCP server for Claude |
| [video-edge](https://github.com/Prodro21/video-edge) | Edge server for stadiums |
| [go-video-capture](https://github.com/Prodro21/go-video-capture) | Video capture application |
