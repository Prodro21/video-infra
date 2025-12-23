# Video Streaming Platform - Architecture

## System Overview

A distributed video streaming platform for sports coaching, enabling real-time clip capture, annotation, and review across multiple devices.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FIELD/STADIUM                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐               │
│   │   Camera 1   │     │   Camera 2   │     │   Camera N   │               │
│   │  (End Zone)  │     │  (Sideline)  │     │   (Booth)    │               │
│   └──────┬───────┘     └──────┬───────┘     └──────┬───────┘               │
│          │                    │                    │                        │
│          │ SDI/HDMI           │                    │                        │
│          ▼                    ▼                    ▼                        │
│   ┌────────────────────────────────────────────────────────────────┐       │
│   │                    CAPTURE STATIONS                             │       │
│   │  ┌────────────┐  ┌────────────┐  ┌────────────┐                │       │
│   │  │ BlackMagic │  │ BlackMagic │  │ BlackMagic │                │       │
│   │  │  DeckLink  │  │  DeckLink  │  │  DeckLink  │                │       │
│   │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘                │       │
│   │        │               │               │                        │       │
│   │        ▼               ▼               ▼                        │       │
│   │  ┌─────────────────────────────────────────────────────┐       │       │
│   │  │     go-video-capture (or python-capture)            │       │       │
│   │  │     • H.264 encoding                                │       │       │
│   │  │     • SRT stream output                             │       │       │
│   │  │     • Local recording                               │       │       │
│   │  └─────────────────────┬───────────────────────────────┘       │       │
│   └────────────────────────┼────────────────────────────────────────┘       │
│                            │ SRT (port 8890)                                │
│                            ▼                                                │
│   ┌────────────────────────────────────────────────────────────────┐       │
│   │                      MediaMTX                                   │       │
│   │  • SRT receive (8890)                                          │       │
│   │  • fMP4 recording                                              │       │
│   │  • WebRTC/HLS output (8889)                                    │       │
│   │  • API (9997)                                                  │       │
│   └────────────────────────┬───────────────────────────────────────┘       │
│                            │                                                │
│                            ▼                                                │
│   ┌────────────────────────────────────────────────────────────────┐       │
│   │                  video-platform (Go)                            │       │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │       │
│   │  │ Sessions │ │  Clips   │ │ Channels │ │   Tags   │           │       │
│   │  └──────────┘ └──────────┘ └──────────┘ └──────────┘           │       │
│   │  • REST API (:8080)                                            │       │
│   │  • WebSocket events                                            │       │
│   │  • PostgreSQL storage                                          │       │
│   │  • Clip generation (FFmpeg)                                    │       │
│   └────────────────────────┬───────────────────────────────────────┘       │
│                            │                                                │
│         ┌──────────────────┼──────────────────┐                            │
│         │                  │                  │                            │
│         ▼                  ▼                  ▼                            │
│   ┌───────────┐     ┌───────────┐     ┌───────────┐                       │
│   │  iPad 1   │     │  iPad 2   │     │  iPad N   │                       │
│   │ (Coach 1) │     │ (Coach 2) │     │ (Analyst) │                       │
│   └───────────┘     └───────────┘     └───────────┘                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Components

### 1. go-video-capture
**Repository:** `Prodro21/go-video-capture`

Multi-channel video capture application for field deployment.

```
go-video-capture/
├── cmd/capture/         # Main entry point
├── pkg/
│   ├── capture/         # FFmpeg capture management
│   ├── encoder/         # H.264 encoding pipeline
│   └── srt/             # SRT streaming output
├── configs/             # Channel configurations
└── internal/            # Internal utilities
```

**Features:**
- BlackMagic DeckLink input via FFmpeg
- Multi-channel concurrent capture
- SRT stream output to MediaMTX
- Local TS recording backup
- GPU acceleration (NVENC when available)
- Auto-reconnect with fallback slate

**Configuration:**
```yaml
channels:
  - name: endzone
    input: /dev/video0
    resolution: 1920x1080
    framerate: 60
    srt_target: srt://mediamtx:8890?streamid=publish:endzone
```

---

### 2. video-platform
**Repository:** `Prodro21/video-platform`

Core backend API server managing all video platform operations.

```
video-platform/
├── cmd/server/          # Main entry point
├── internal/
│   ├── api/             # REST API handlers
│   │   ├── sessions.go
│   │   ├── clips.go
│   │   ├── channels.go
│   │   └── tags.go
│   ├── models/          # Database models
│   ├── services/        # Business logic
│   │   ├── session.go
│   │   ├── clip.go
│   │   └── clipgen.go   # FFmpeg clip generation
│   ├── websocket/       # Real-time events
│   └── storage/         # File storage
├── migrations/          # PostgreSQL migrations
└── docs/                # API documentation
```

**API Endpoints:**

| Resource | Endpoints |
|----------|-----------|
| Sessions | `GET/POST /api/v1/sessions`, `GET/PUT/DELETE /api/v1/sessions/:id`, `POST /sessions/:id/{start,pause,complete}` |
| Clips | `GET /api/v1/clips`, `GET/DELETE /api/v1/clips/:id`, `GET /clips/:id/stream`, `GET /clips/:id/thumbnail`, `POST /clips/:id/favorite` |
| Channels | `GET/POST /api/v1/channels`, `POST /channels/:id/{activate,deactivate}`, `POST /channels/:id/heartbeat` |
| Tags | `GET/POST /api/v1/tags`, `PUT/DELETE /api/v1/tags/:id` |
| WebSocket | `WS /ws` (events: clip_ready, session_start, session_end) |

**Database Schema:**
```sql
-- Sessions
sessions (id, name, type, status, opponent, location, scheduled_at, started_at, ended_at)

-- Clips
clips (id, session_id, channel_id, start_time, duration, status, file_path, thumbnail_path, is_favorite)

-- Channels
channels (id, name, source_url, status, last_heartbeat)

-- Tags
tags (id, clip_id, type, label, timestamp, data, is_reviewed)
```

---

### 3. video-dashboard
**Repository:** `Prodro21/video-dashboard`

React-based admin dashboard for managing the video platform.

```
video-dashboard/
├── src/
│   ├── api/             # HTTP client & WebSocket
│   ├── components/
│   │   ├── layout/      # Layout, Sidebar, Header
│   │   ├── common/      # Button, Card, Modal, Table
│   │   ├── dashboard/   # Stats, ActiveSessions
│   │   ├── sessions/    # SessionList, SessionForm
│   │   ├── clips/       # ClipGrid, ClipPlayer
│   │   └── channels/    # ChannelList, HealthIndicator
│   ├── pages/           # Route pages
│   ├── stores/          # Zustand state management
│   └── types/           # TypeScript definitions
├── tailwind.config.js
└── vite.config.ts
```

**Tech Stack:**
- React 18 + TypeScript
- Vite build system
- Tailwind CSS (dark theme)
- Zustand state management
- hls.js video playback
- WebSocket real-time updates

**Routes:**
```
/                  → Dashboard (stats, active sessions)
/sessions          → Session list
/sessions/:id      → Session detail with clips
/channels          → Channel management
/clips             → Clip browser
/clips/:id         → Clip detail with player
/tags              → Tag management
```

---

### 4. video-ipad
**Repository:** `Prodro21/video-ipad`

Native iPad application for coaches to view and annotate clips.

```
video-ipad/
├── VideoCoach/
│   ├── Models/          # Session, Clip, Tag
│   ├── ViewModels/      # ObservableObject stores
│   ├── Views/
│   │   ├── Sessions/    # SessionListView, SessionDetailView
│   │   ├── Clips/       # ClipGridView, ClipPlayerView
│   │   └── Tags/        # TagOverlay, TagEditor
│   ├── Services/
│   │   ├── APIClient.swift
│   │   └── WebSocketService.swift
│   └── Utilities/
└── VideoCoach.xcodeproj
```

**Tech Stack:**
- SwiftUI for UI
- AVPlayer for video playback
- Combine for reactive updates
- Core Data for offline storage

**Features:**
- Browse sessions and clips
- Video playback with scrubbing
- Create/edit tags with timestamps
- Mark favorites
- Offline support with sync
- WebSocket real-time updates

---

### 5. video-mcp
**Repository:** `Prodro21/video-mcp`

MCP (Model Context Protocol) server for AI assistant integration.

```
video-mcp/
├── cmd/server/          # Main entry point
└── internal/
    ├── client/          # HTTP client for video-platform
    └── handlers/
        ├── tools.go     # MCP tools
        ├── resources.go # MCP resources
        └── prompts.go   # MCP prompts
```

**MCP Tools:**
- Session: list, create, start, pause, complete
- Clips: list, favorite
- Channels: list, activate, deactivate
- Tags: list, create

**MCP Resources:**
- `video://sessions` - All sessions
- `video://clips` - All clips
- `video://channels` - Channel status
- `video://tags` - All tags

**MCP Prompts:**
- `analyze_session` - Coaching analysis workflow
- `review_clips` - Clip review workflow
- `game_report` - Game report generation
- `system_status` - System health check

**Claude Desktop Config:**
```json
{
  "mcpServers": {
    "video-platform": {
      "command": "/usr/local/bin/video-mcp",
      "args": ["-api-url", "http://localhost:8080"]
    }
  }
}
```

---

### 6. operator-console (Legacy)
**Repository:** `Prodro21/operator-console`

Original React dashboard (being replaced by video-dashboard).

---

### 7. video-protocol
**Repository:** (planned)

Shared protocol definitions and types across all components.

---

## Data Flow

### Live Clip Creation Flow

```
1. Camera captures video
   │
2. go-video-capture encodes to H.264, streams via SRT
   │
3. MediaMTX receives SRT, records fMP4 segments
   │
4. Coach marks IN on iPad
   │ POST /api/v1/clips { session_id, channel_id, start_time }
   │
5. video-platform creates clip record (status: pending)
   │
6. Coach marks OUT on iPad
   │ PUT /api/v1/clips/:id { duration }
   │
7. Clip generator extracts segment from MediaMTX recording
   │ FFmpeg: copy segments, trim edges
   │
8. Clip file saved, thumbnail generated
   │ status: ready
   │
9. WebSocket broadcast: { type: "clip_ready", clip_id }
   │
10. iPads receive notification, fetch and display clip
```

### Session Lifecycle

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ scheduled│───▶│  active  │───▶│  paused  │───▶│ completed│
└──────────┘    └──────────┘    └──────────┘    └──────────┘
                     │               │
                     └───────────────┘
                        (resume)
```

---

## Network Architecture

### Stadium Deployment

```
┌─────────────────────────────────────────────────────────────┐
│                    STADIUM NETWORK                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │               Stadium WiFi (5GHz)                    │    │
│  │                      │                               │    │
│  │    ┌─────────────────┴─────────────────┐            │    │
│  │    │                                   │            │    │
│  │    ▼                                   ▼            │    │
│  │  iPads                           Capture PCs        │    │
│  │  (10.0.1.x)                      (10.0.2.x)        │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          │ LAN                               │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │               Edge Server (10.0.0.1)                 │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐             │    │
│  │  │ MediaMTX │ │ Platform │ │   Redis  │             │    │
│  │  │  :8889   │ │  :8080   │ │  :6379   │             │    │
│  │  └──────────┘ └──────────┘ └──────────┘             │    │
│  │  ┌──────────┐ ┌──────────┐                          │    │
│  │  │ Postgres │ │  Nginx   │                          │    │
│  │  │  :5432   │ │   :80    │                          │    │
│  │  └──────────┘ └──────────┘                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          │ WAN (optional)                    │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Cloud Backup                       │    │
│  │  • Video archive                                     │    │
│  │  • Clip sync                                         │    │
│  │  • Remote dashboard                                  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Ports

| Service | Port | Protocol |
|---------|------|----------|
| video-platform API | 8080 | HTTP |
| MediaMTX SRT | 8890 | SRT |
| MediaMTX WebRTC/HLS | 8889 | HTTP |
| MediaMTX API | 9997 | HTTP |
| PostgreSQL | 5432 | TCP |
| Redis | 6379 | TCP |
| Dashboard | 5173 | HTTP (dev) |

---

## Technology Stack

| Layer | Technology |
|-------|------------|
| **Capture** | Go, FFmpeg, BlackMagic SDK |
| **Streaming** | MediaMTX, SRT, fMP4, HLS |
| **Backend** | Go, Gin, PostgreSQL, Redis |
| **Web Dashboard** | React, TypeScript, Vite, Tailwind |
| **Mobile** | Swift, SwiftUI, AVPlayer |
| **AI Integration** | MCP, Claude Desktop |
| **Infrastructure** | Docker, nginx |

---

## Deployment

### Development

```bash
# Start PostgreSQL
docker run -d --name postgres -p 5432:5432 \
  -e POSTGRES_PASSWORD=postgres postgres:15

# Start video-platform
cd video-platform && go run ./cmd/server

# Start video-dashboard
cd video-dashboard && npm run dev

# Start video-mcp (for Claude Desktop)
cd video-mcp && go run ./cmd/server
```

### Production (Docker Compose)

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: video_platform
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  mediamtx:
    image: bluenviron/mediamtx:latest
    ports:
      - "8889:8889"
      - "8890:8890"
    volumes:
      - ./mediamtx.yml:/mediamtx.yml
      - ./recordings:/recordings

  video-platform:
    build: ./video-platform
    depends_on:
      - postgres
    environment:
      DATABASE_URL: postgres://postgres:${DB_PASSWORD}@postgres/video_platform
      RECORDING_PATH: /recordings
    ports:
      - "8080:8080"
    volumes:
      - ./recordings:/recordings
      - ./clips:/clips

  video-dashboard:
    build: ./video-dashboard
    ports:
      - "80:80"

volumes:
  postgres_data:
```

---

## Future Components

### Edge Server (Planned)
- Go service for stadium deployment
- Local clip caching
- Offline operation
- Cloud sync when connected

### Mobile PWA (Planned)
- Progressive Web App for quick access
- Works on any device
- Offline capable

---

## Repositories

| Component | Repository | Status |
|-----------|------------|--------|
| video-platform | github.com/Prodro21/video-platform | ✅ Active |
| video-dashboard | github.com/Prodro21/video-dashboard | ✅ Active |
| video-ipad | github.com/Prodro21/video-ipad | ✅ Active |
| video-mcp | github.com/Prodro21/video-mcp | ✅ Active |
| go-video-capture | github.com/Prodro21/go-video-capture | ✅ Active |
| operator-console | github.com/Prodro21/operator-console | 📦 Legacy |
