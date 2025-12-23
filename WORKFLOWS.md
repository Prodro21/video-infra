# Component Interactions & Workflows

## Core Workflows

### 1. Live Game Recording

This is the primary workflow during a live football game.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         LIVE GAME WORKFLOW                                   │
└─────────────────────────────────────────────────────────────────────────────┘

  BEFORE GAME
  ───────────

  Dashboard                    video-platform                  Database
     │                              │                              │
     │ POST /sessions              │                              │
     │ {name, type: "game",        │                              │
     │  opponent, season, week}    │                              │
     │────────────────────────────▶│ INSERT session               │
     │                              │ status: "scheduled"          │
     │                              │─────────────────────────────▶│
     │◀────────────────────────────│                              │
     │ session_id                  │                              │


  GAME START
  ──────────

  Operator                     video-platform                  Capture/MediaMTX
     │                              │                              │
     │ POST /sessions/:id/start    │                              │
     │────────────────────────────▶│ UPDATE status: "active"      │
     │                              │ SET actual_start = NOW()     │
     │                              │                              │
     │                              │ WebSocket broadcast          │
     │                              │ {type: "session_start"}      │
     │                              │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ▶│
     │                              │                              │
                                    │                              │
  go-video-capture                 │                              │
     │                              │                              │
     │ POST /channels/:id/activate │                              │
     │────────────────────────────▶│ UPDATE channel status        │
     │                              │                              │
     │ POST /channels/:id/heartbeat│ (every 5s)                   │
     │────────────────────────────▶│ UPDATE last_seen_at          │
     │                              │                              │
     │ SRT stream ─────────────────────────────────────────────── ▶│ MediaMTX
     │                              │                              │ records fMP4


  DURING GAME - CLIP CREATION
  ───────────────────────────

  iPad (Coach)                 video-platform                  Clip Generator
     │                              │                              │
     │ [Coach sees good play]      │                              │
     │ [Marks IN at 14:23:15]      │                              │
     │                              │                              │
     │ POST /clips                 │                              │
     │ {session_id, channel_id,    │                              │
     │  start_time: "14:23:15",    │                              │
     │  duration_seconds: 0}       │                              │
     │────────────────────────────▶│ INSERT clip                  │
     │                              │ status: "pending"            │
     │◀────────────────────────────│                              │
     │ clip_id (ghost clip)        │                              │
     │                              │                              │
     │ [Coach sees play end]       │                              │
     │ [Marks OUT at 14:23:27]     │                              │
     │                              │                              │
     │ PATCH /clips/:id            │                              │
     │ {duration_seconds: 12}      │                              │
     │────────────────────────────▶│ UPDATE clip                  │
     │                              │ Queue for generation         │
     │                              │─────────────────────────────▶│
     │                              │                              │
     │                              │         [Clip Generator]     │
     │                              │         Find recording file  │
     │                              │         FFmpeg extract        │
     │                              │         Generate thumbnail   │
     │                              │                              │
     │                              │◀─────────────────────────────│
     │                              │ Clip ready                   │
     │                              │ UPDATE status: "ready"       │
     │                              │                              │
     │ WebSocket                   │                              │
     │ {type: "clip_ready",        │                              │
     │  clip_id, session_id}       │                              │
     │◀─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│                              │
     │                              │                              │
     │ GET /clips/:id/stream       │                              │
     │────────────────────────────▶│ Stream video                 │
     │◀────────────────────────────│                              │
     │ [Play video]                │                              │


  HALFTIME/TIMEOUT - REVIEW
  ─────────────────────────

  iPad (Coach)                 video-platform                  Dashboard
     │                              │                              │
     │ GET /clips?session_id=X     │                              │
     │    &is_favorite=true        │                              │
     │────────────────────────────▶│                              │
     │◀────────────────────────────│                              │
     │ [View favorite clips]       │                              │
     │                              │                              │
     │ POST /tags                  │                              │
     │ {clip_id, quarter: 2,       │                              │
     │  down: 3, play_type: "Pass",│                              │
     │  result: "Complete",        │                              │
     │  notes: "Great route"}      │                              │
     │────────────────────────────▶│ INSERT tag                   │
     │                              │                              │


  GAME END
  ────────

  Operator                     video-platform                  All Clients
     │                              │                              │
     │ POST /sessions/:id/complete │                              │
     │────────────────────────────▶│ UPDATE status: "completed"   │
     │                              │ SET actual_end = NOW()       │
     │                              │                              │
     │                              │ WebSocket broadcast          │
     │                              │ {type: "session_end"}        │
     │                              │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ▶│
```

---

### 2. Post-Game Analysis

After the game, analysts review footage and create highlight clips.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       POST-GAME ANALYSIS WORKFLOW                            │
└─────────────────────────────────────────────────────────────────────────────┘

  Analyst                      Dashboard                       video-platform
     │                              │                              │
     │ Open session detail         │                              │
     │─────────────────────────────▶│                              │
     │                              │ GET /sessions/:id            │
     │                              │────────────────────────────▶│
     │                              │◀────────────────────────────│
     │                              │                              │
     │                              │ GET /sessions/:id/clips      │
     │                              │────────────────────────────▶│
     │                              │◀────────────────────────────│
     │◀─────────────────────────────│ Show session + clips        │
     │                              │                              │
     │ Click on clip               │                              │
     │─────────────────────────────▶│                              │
     │                              │ GET /clips/:id/stream        │
     │                              │────────────────────────────▶│
     │                              │◀────────────────────────────│
     │◀─────────────────────────────│ Video player                │
     │                              │                              │
     │ Browse full recording       │                              │
     │─────────────────────────────▶│                              │
     │                              │ WebRTC from MediaMTX         │
     │                              │─────────────────────────────▶│ MediaMTX
     │                              │◀────────────────────────────│
     │◀─────────────────────────────│ Full recording playback     │
     │                              │                              │
     │ Found good play, create clip│                              │
     │─────────────────────────────▶│                              │
     │                              │ POST /clips                  │
     │                              │ {start_time, duration}       │
     │                              │────────────────────────────▶│
     │                              │◀────────────────────────────│
     │                              │                              │
     │                              │ [Clip generation in bg]      │
     │                              │                              │
     │                              │ WebSocket: clip_ready        │
     │◀─────────────────────────────│◀─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│
     │                              │                              │
     │ Add detailed tags           │                              │
     │─────────────────────────────▶│                              │
     │                              │ POST /tags                   │
     │                              │ {quarter, down, formation,   │
     │                              │  play_type, result, players} │
     │                              │────────────────────────────▶│
```

---

### 3. AI-Assisted Review (MCP)

Using Claude Desktop with the video-mcp server.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AI-ASSISTED REVIEW WORKFLOW                           │
└─────────────────────────────────────────────────────────────────────────────┘

  User                         Claude Desktop                  video-mcp
     │                              │                              │
     │ "Analyze yesterday's game"  │                              │
     │────────────────────────────▶│                              │
     │                              │                              │
     │                              │ [Claude interprets request]  │
     │                              │                              │
     │                              │ Call tool: list_sessions     │
     │                              │ {type: "game", limit: 5}     │
     │                              │─────────────────────────────▶│
     │                              │                              │──┐
     │                              │                              │  │ GET /sessions
     │                              │                              │  │ to video-platform
     │                              │                              │◀─┘
     │                              │◀─────────────────────────────│
     │                              │ [Sessions list]              │
     │                              │                              │
     │                              │ Read resource:               │
     │                              │ video://sessions             │
     │                              │─────────────────────────────▶│
     │                              │                              │──┐
     │                              │                              │  │ Fetch sessions
     │                              │                              │◀─┘
     │                              │◀─────────────────────────────│
     │                              │ [Session data]               │
     │                              │                              │
     │                              │ Call tool: list_clips        │
     │                              │ {session_id: "..."}          │
     │                              │─────────────────────────────▶│
     │                              │◀─────────────────────────────│
     │                              │ [Clips with tags]            │
     │                              │                              │
     │                              │ [Claude analyzes data]       │
     │                              │                              │
     │◀────────────────────────────│                              │
     │ "Based on the session data, │                              │
     │  here's my analysis:        │                              │
     │  - 45 total plays           │                              │
     │  - Run/Pass split: 60/40    │                              │
     │  - 3rd down conversion: 42% │                              │
     │  - Key plays to review..."  │                              │
     │                              │                              │
     │ "Mark the touchdown clips   │                              │
     │  as favorites"              │                              │
     │────────────────────────────▶│                              │
     │                              │                              │
     │                              │ Call tool: favorite_clip     │
     │                              │ (for each TD clip)           │
     │                              │─────────────────────────────▶│
     │                              │                              │──┐
     │                              │                              │  │ POST /clips/:id/favorite
     │                              │                              │◀─┘
     │                              │◀─────────────────────────────│
     │◀────────────────────────────│ "Done! Marked 4 clips"       │
```

---

### 4. iPad Real-Time Updates

How iPads stay synchronized during a game.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        iPAD REAL-TIME SYNC                                   │
└─────────────────────────────────────────────────────────────────────────────┘

  iPad App                     video-platform                  WebSocket Hub
     │                              │                              │
     │ Connect WebSocket           │                              │
     │ ws://server/ws              │                              │
     │────────────────────────────▶│ Register client              │
     │                              │─────────────────────────────▶│
     │◀────────────────────────────│ Connected                    │
     │                              │                              │
     │                              │                              │
  ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─
     │                              │                              │
     │   [Another coach creates    │                              │
     │    a clip on their iPad]    │                              │
     │                              │◀─────────────────────────────│
     │                              │ Clip ready                   │
     │                              │                              │
     │                              │ Broadcast to all clients     │
     │                              │─────────────────────────────▶│
     │ {type: "clip_ready",        │                              │
     │  clip_id, session_id,       │                              │
     │  thumbnail_url}             │                              │
     │◀─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│                              │
     │                              │                              │
     │ [App shows notification]    │                              │
     │ [Auto-downloads clip]       │                              │
     │                              │                              │
     │ GET /clips/:id              │                              │
     │────────────────────────────▶│                              │
     │◀────────────────────────────│                              │
     │                              │                              │
     │ GET /clips/:id/stream       │                              │
     │────────────────────────────▶│                              │
     │◀────────────────────────────│ Video data                   │
     │                              │                              │
     │ [Clip ready for playback]   │                              │
```

---

## Data Models & Relationships

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DATA MODEL                                         │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                              SESSION                                     │
  │  id, name, type, status, opponent, location, season, week               │
  │  scheduled_start, actual_start, actual_end                              │
  │  clip_count, tag_count, total_duration_seconds                          │
  └─────────────────────────────────────────────────────────────────────────┘
                │                                    │
                │ 1:N                                │ 1:N
                ▼                                    ▼
  ┌─────────────────────────────┐    ┌─────────────────────────────────────┐
  │          CHANNEL            │    │              CLIP                    │
  │  id, name, input_url        │    │  id, session_id, channel_id         │
  │  status, last_seen_at       │    │  title, start_time, duration        │
  │  resolution, framerate      │    │  file_path, thumbnail_path          │
  └─────────────────────────────┘    │  status, is_favorite, view_count    │
                │                    └─────────────────────────────────────┘
                │ 1:N                                │
                │                                    │ 1:N
                ▼                                    ▼
  ┌─────────────────────────────┐    ┌─────────────────────────────────────┐
  │     Channel recordings      │    │              TAG                     │
  │  (MediaMTX fMP4 files)      │    │  id, clip_id, session_id            │
  │  /recordings/{channel}/     │    │  quarter, down, distance, yard_line │
  │     2024-09-01_13-00-00.mp4 │    │  play_type, formation, result       │
  └─────────────────────────────┘    │  yards_gained, players[], labels[]  │
                                     │  notes, coach_notes                  │
                                     │  is_important, is_reviewed           │
                                     └─────────────────────────────────────┘
```

---

## Component Communication Matrix

| From → To | Protocol | Endpoint | Purpose |
|-----------|----------|----------|---------|
| Dashboard → Platform | HTTP | REST API | CRUD operations |
| Dashboard → Platform | WebSocket | /ws | Real-time events |
| iPad → Platform | HTTP | REST API | CRUD operations |
| iPad → Platform | WebSocket | /ws | Real-time events |
| Capture → Platform | HTTP | REST API | Register clips, heartbeat |
| Capture → MediaMTX | SRT | :8890 | Video stream |
| Platform → MediaMTX | HTTP | :9997 | Recording metadata |
| MCP → Platform | HTTP | REST API | All operations |
| Dashboard → MediaMTX | WebRTC | :8889 | Live preview |

---

## Error Handling

### Clip Generation Failures

```
  Clip Generator                video-platform                  Clients
     │                              │                              │
     │ FFmpeg fails                │                              │
     │ (file not found, codec err) │                              │
     │─────────────────────────────▶│                              │
     │                              │ UPDATE clip                  │
     │                              │ status: "failed"             │
     │                              │ error_message: "..."         │
     │                              │                              │
     │                              │ WebSocket broadcast          │
     │                              │ {type: "clip_failed",        │
     │                              │  clip_id, error}             │
     │                              │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ▶│
     │                              │                              │
     │                              │                              │ Show error to user
     │                              │                              │ Option to retry
```

### Channel Disconnection

```
  go-video-capture              video-platform                  Dashboard
     │                              │                              │
     │ [Heartbeat stops]           │                              │
     │                              │                              │
     │                              │ [After 30s timeout]          │
     │                              │ UPDATE channel               │
     │                              │ status: "error"              │
     │                              │                              │
     │                              │ WebSocket broadcast          │
     │                              │ {type: "channel_status",     │
     │                              │  channel_id, status: "error"}│
     │                              │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ▶│
     │                              │                              │ Show warning
     │                              │                              │
     │ [Reconnect]                 │                              │
     │ POST /channels/:id/heartbeat│                              │
     │────────────────────────────▶│ UPDATE status: "active"      │
     │                              │                              │
     │                              │ WebSocket broadcast          │
     │                              │ {type: "channel_status",     │
     │                              │  channel_id, status: "active"}│
     │                              │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ▶│ Clear warning
```

---

## State Machines

### Session States

```
                    ┌──────────┐
                    │ scheduled│
                    └────┬─────┘
                         │ start()
                         ▼
              ┌─────────────────────┐
         ┌───▶│       active        │◀──┐
         │    └─────────┬───────────┘   │
         │              │               │
         │    pause()   │               │ resume()
         │              ▼               │
         │    ┌─────────────────────┐   │
         │    │       paused        │───┘
         │    └─────────────────────┘
         │              │
         │              │ complete()
         │              ▼
         │    ┌─────────────────────┐
         └────│      completed      │
              └─────────────────────┘
```

### Clip States

```
     ┌─────────┐
     │ pending │
     └────┬────┘
          │ Generator picks up
          ▼
    ┌────────────┐
    │ processing │
    └──────┬─────┘
           │
     ┌─────┴─────┐
     │           │
     ▼           ▼
┌─────────┐ ┌────────┐
│  ready  │ │ failed │
└─────────┘ └────────┘
                │
                │ retry
                ▼
          ┌─────────┐
          │ pending │
          └─────────┘
```

### Channel States

```
                  ┌──────────┐
            ┌────▶│ inactive │◀────┐
            │     └────┬─────┘     │
            │          │           │
   deactivate()   activate()   error
            │          │           │
            │          ▼           │
            │     ┌────────┐       │
            └─────│ active │───────┘
                  └────┬───┘
                       │ timeout / error
                       ▼
                  ┌────────┐
                  │ error  │
                  └────────┘
```
