# Deployment Guide

## Quick Start (Development)

### Prerequisites

- Go 1.23+
- Node.js 18+
- PostgreSQL 15+
- FFmpeg 6+
- Docker (optional)

### 1. Start PostgreSQL

```bash
# Using Docker
docker run -d --name postgres \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=video_platform \
  postgres:15

# Or use existing PostgreSQL
createdb video_platform
```

### 2. Start video-platform

```bash
cd video-platform

# Set environment
export DATABASE_URL="postgres://postgres:postgres@localhost:5432/video_platform?sslmode=disable"
export CLIPS_PATH="./clips"
export RECORDINGS_PATH="./recordings"

# Run migrations
go run ./cmd/migrate up

# Start server
go run ./cmd/server
# Server running at http://localhost:8080
```

### 3. Start video-dashboard

```bash
cd video-dashboard

# Install dependencies
npm install

# Start dev server
npm run dev
# Dashboard at http://localhost:5173
```

### 4. Start video-mcp (Optional, for Claude Desktop)

```bash
cd video-mcp
go build -o video-mcp ./cmd/server

# Add to Claude Desktop config
```

---

## Production Deployment

### Docker Compose

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: video_platform
      POSTGRES_USER: video
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U video -d video_platform"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  mediamtx:
    image: bluenviron/mediamtx:latest
    ports:
      - "8889:8889"   # WebRTC/HLS
      - "8890:8890"   # SRT input
      - "9997:9997"   # API
    volumes:
      - ./mediamtx.yml:/mediamtx.yml
      - recordings:/recordings
    command: /mediamtx /mediamtx.yml

  video-platform:
    build:
      context: ./video-platform
      dockerfile: Dockerfile
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://video:${DB_PASSWORD}@postgres:5432/video_platform?sslmode=disable
      REDIS_URL: redis://redis:6379
      CLIPS_PATH: /data/clips
      RECORDINGS_PATH: /recordings
      MEDIAMTX_API: http://mediamtx:9997
    ports:
      - "8080:8080"
    volumes:
      - clips:/data/clips
      - recordings:/recordings

  video-dashboard:
    build:
      context: ./video-dashboard
      dockerfile: Dockerfile
    ports:
      - "80:80"
    depends_on:
      - video-platform

volumes:
  postgres_data:
  redis_data:
  clips:
  recordings:
```

### Dockerfiles

**video-platform/Dockerfile:**

```dockerfile
# Build stage
FROM golang:1.23-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /video-platform ./cmd/server

# Runtime stage
FROM alpine:3.19

RUN apk add --no-cache ffmpeg

COPY --from=builder /video-platform /usr/local/bin/
COPY migrations /app/migrations

WORKDIR /app
EXPOSE 8080

CMD ["video-platform"]
```

**video-dashboard/Dockerfile:**

```dockerfile
# Build stage
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Runtime stage
FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**video-dashboard/nginx.conf:**

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # SPA routing
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API proxy
    location /api/ {
        proxy_pass http://video-platform:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # WebSocket proxy
    location /ws {
        proxy_pass http://video-platform:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

### MediaMTX Configuration

**mediamtx.yml:**

```yaml
logLevel: info
logDestinations: [stdout]

api: yes
apiAddress: :9997

rtsp: no
rtmp: no
hls: yes
webrtc: yes

hlsAddress: :8889
webrtcAddress: :8889

srt: yes
srtAddress: :8890

# Path configuration
paths:
  all:
    # Enable recording
    record: yes
    recordPath: /recordings/%path/%Y-%m-%d_%H-%M-%S
    recordFormat: fmp4
    recordSegmentDuration: 2s
    recordDeleteAfter: 24h

  endzone:
    source: publisher

  sideline:
    source: publisher

  booth:
    source: publisher
```

### Deploy Commands

```bash
# Create .env file
cat > .env << EOF
DB_PASSWORD=your-secure-password
EOF

# Start all services
docker-compose up -d

# Check logs
docker-compose logs -f

# Run migrations
docker-compose exec video-platform video-platform migrate up

# Stop all services
docker-compose down
```

---

## Stadium Deployment

For on-site stadium deployments without reliable internet.

### Hardware Requirements

**Edge Server:**
- CPU: 8+ cores (Intel i7/Xeon or AMD Ryzen 7)
- RAM: 32GB+
- Storage: 2TB+ NVMe SSD
- Network: Gigabit Ethernet

**Capture Stations (per camera):**
- CPU: 6+ cores
- RAM: 16GB+
- GPU: NVIDIA GTX 1650+ (optional, for NVENC)
- Capture Card: Blackmagic DeckLink Mini Recorder 4K

**Network:**
- Managed Gigabit Switch
- 5GHz WiFi Access Points (for iPads)
- Wired connections for all capture stations

### Network Topology

```
┌─────────────────────────────────────────────────────────────┐
│                     Stadium Network                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │            Managed Switch (10.0.0.0/24)             │    │
│  │                                                     │    │
│  │  Port 1:  Edge Server (10.0.0.1)                   │    │
│  │  Port 2:  Capture PC 1 (10.0.0.10)                 │    │
│  │  Port 3:  Capture PC 2 (10.0.0.11)                 │    │
│  │  Port 4:  Capture PC 3 (10.0.0.12)                 │    │
│  │  Port 5:  WiFi AP 1 (10.0.0.20)                    │    │
│  │  Port 6:  WiFi AP 2 (10.0.0.21)                    │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  WiFi Network: VideoCoach-5G (WPA3)                         │
│  iPad DHCP Range: 10.0.0.100-199                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Edge Server Setup

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh

# Clone repositories
git clone https://github.com/Prodro21/video-platform.git
git clone https://github.com/Prodro21/video-dashboard.git

# Create docker-compose.yml (see above)

# Configure static IP
sudo nmcli con mod "Wired connection 1" \
  ipv4.addresses "10.0.0.1/24" \
  ipv4.method manual

# Start services
docker-compose up -d
```

### Capture Station Setup

```bash
# Install dependencies
sudo apt install ffmpeg

# Clone capture software
git clone https://github.com/Prodro21/go-video-capture.git
cd go-video-capture

# Build
go build -o capture ./cmd/capture

# Configure
cat > config.yaml << EOF
channels:
  - name: endzone
    input_device: /dev/video0
    resolution: 1920x1080
    framerate: 60
    srt_target: srt://10.0.0.1:8890?streamid=publish:endzone
    platform_api: http://10.0.0.1:8080
EOF

# Run
./capture --config config.yaml
```

### iPad Configuration

1. Connect to VideoCoach-5G WiFi
2. Open Safari, go to http://10.0.0.1
3. Add to Home Screen for app-like experience

Or install native VideoCoach app:
1. Install via TestFlight or enterprise distribution
2. Configure server URL: http://10.0.0.1:8080

---

## Cloud Backup (Optional)

### S3 Sync

Add to docker-compose.yml:

```yaml
  s3-sync:
    image: amazon/aws-cli
    environment:
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_KEY}
    volumes:
      - clips:/data/clips:ro
    entrypoint: /bin/sh -c
    command: |
      while true; do
        aws s3 sync /data/clips s3://your-bucket/clips --storage-class STANDARD_IA
        sleep 3600
      done
```

### Database Backup

```bash
# Backup
docker-compose exec postgres pg_dump -U video video_platform > backup.sql

# Restore
docker-compose exec -T postgres psql -U video video_platform < backup.sql
```

---

## Monitoring

### Health Checks

```bash
# API health
curl http://localhost:8080/health

# PostgreSQL
docker-compose exec postgres pg_isready

# MediaMTX
curl http://localhost:9997/v3/paths/list
```

### Logging

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f video-platform

# With timestamps
docker-compose logs -f --timestamps
```

### Metrics (Future)

Prometheus metrics endpoint planned at `/metrics`.

---

## Troubleshooting

### Common Issues

**Database connection failed:**
```bash
# Check PostgreSQL is running
docker-compose ps postgres

# Check connection
docker-compose exec postgres psql -U video -d video_platform -c "SELECT 1"
```

**Clips not generating:**
```bash
# Check FFmpeg
docker-compose exec video-platform ffmpeg -version

# Check recordings directory
docker-compose exec video-platform ls -la /recordings

# Check clip generator logs
docker-compose logs video-platform | grep -i clip
```

**WebSocket not connecting:**
```bash
# Test WebSocket
wscat -c ws://localhost:8080/ws

# Check nginx proxy config
docker-compose exec video-dashboard cat /etc/nginx/conf.d/default.conf
```

**SRT stream not received:**
```bash
# Check MediaMTX logs
docker-compose logs mediamtx

# List active paths
curl http://localhost:9997/v3/paths/list

# Test SRT connection
ffplay "srt://localhost:8890?mode=caller&streamid=endzone"
```

### Reset Everything

```bash
# Stop and remove all containers, volumes
docker-compose down -v

# Remove all data
rm -rf ./data

# Fresh start
docker-compose up -d
```

---

## Security Considerations

### Production Checklist

- [ ] Change default database password
- [ ] Enable HTTPS with valid certificate
- [ ] Configure firewall (only expose 80/443)
- [ ] Set up authentication (JWT)
- [ ] Enable CORS restrictions
- [ ] Regular security updates
- [ ] Backup strategy in place
- [ ] Monitor logs for anomalies

### HTTPS Setup

```bash
# Using certbot with nginx
certbot --nginx -d video.yourdomain.com
```

Or use Traefik as reverse proxy with automatic Let's Encrypt.
