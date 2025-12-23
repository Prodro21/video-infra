# Video Platform API Reference

Base URL: `http://localhost:8080/api/v1`

## Authentication

Currently no authentication required (local development). Production will use JWT tokens.

## Common Response Format

### Success
```json
{
  "id": "uuid",
  "field": "value",
  ...
}
```

### Error
```json
{
  "error": "error message",
  "details": "optional details"
}
```

### Paginated List
```json
{
  "items": [...],
  "total": 100,
  "limit": 20,
  "offset": 0
}
```

---

## Sessions

Sessions represent recording events (games, practices, scrimmages).

### Session Object

| Field | Type | Description |
|-------|------|-------------|
| id | string (uuid) | Unique identifier |
| name | string | Session name |
| session_type | string | `game`, `practice`, `scrimmage` |
| status | string | `scheduled`, `active`, `paused`, `completed` |
| scheduled_start | datetime | Planned start time (ISO 8601) |
| actual_start | datetime | When recording started |
| actual_end | datetime | When recording ended |
| opponent | string | Opponent team name (games) |
| location | string | Venue name |
| season | integer | Season year |
| week | integer | Week number |
| notes | string | Additional notes |
| clip_count | integer | Number of clips |
| tag_count | integer | Number of tags |
| total_duration_seconds | integer | Total video duration |
| created_at | datetime | Creation timestamp |
| updated_at | datetime | Last update timestamp |

### List Sessions

```
GET /sessions
```

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| limit | integer | 20 | Max results |
| offset | integer | 0 | Skip results |
| status | string | - | Filter by status |
| type | string | - | Filter by session_type |
| season | integer | - | Filter by season |

**Response:** `200 OK`
```json
{
  "items": [Session],
  "total": 50,
  "limit": 20,
  "offset": 0
}
```

### Create Session

```
POST /sessions
```

**Request Body:**
```json
{
  "name": "Week 1 vs Eagles",
  "session_type": "game",
  "scheduled_start": "2024-09-01T13:00:00Z",
  "opponent": "Eagles",
  "location": "Home Stadium",
  "season": 2024,
  "week": 1,
  "notes": "Season opener"
}
```

**Response:** `201 Created`
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Week 1 vs Eagles",
  "session_type": "game",
  "status": "scheduled",
  ...
}
```

### Get Session

```
GET /sessions/:id
```

**Response:** `200 OK` - Session object

### Update Session

```
PATCH /sessions/:id
```

**Request Body:** (all fields optional)
```json
{
  "name": "Updated Name",
  "opponent": "New Opponent"
}
```

**Response:** `200 OK` - Updated session

### Delete Session

```
DELETE /sessions/:id
```

**Response:** `204 No Content`

### Start Session

```
POST /sessions/:id/start
```

Sets status to `active` and records `actual_start`.

**Response:** `200 OK` - Updated session

### Pause Session

```
POST /sessions/:id/pause
```

Sets status to `paused`.

**Response:** `200 OK` - Updated session

### Complete Session

```
POST /sessions/:id/complete
```

Sets status to `completed` and records `actual_end`.

**Response:** `200 OK` - Updated session

### Get Session Clips

```
GET /sessions/:id/clips
```

Returns all clips for a session.

**Response:** `200 OK` - Paginated clip list

### Get Session Tags

```
GET /sessions/:id/tags
```

Returns all tags for a session.

**Response:** `200 OK` - Paginated tag list

---

## Clips

Clips are video segments extracted from session recordings.

### Clip Object

| Field | Type | Description |
|-------|------|-------------|
| id | string (uuid) | Unique identifier |
| session_id | string (uuid) | Parent session |
| channel_id | string | Source channel |
| title | string | Clip title |
| start_time | datetime | Clip start (ISO 8601) |
| end_time | datetime | Clip end |
| duration_seconds | float | Duration in seconds |
| file_path | string | Storage path |
| file_size_bytes | integer | File size |
| thumbnail_path | string | Thumbnail image path |
| format | string | Video format (mp4) |
| codec | string | Video codec (h264) |
| resolution | string | e.g., "1920x1080" |
| status | string | `pending`, `processing`, `ready`, `failed` |
| error_message | string | Error details if failed |
| view_count | integer | Number of views |
| is_favorite | boolean | Marked as favorite |
| created_at | datetime | Creation timestamp |
| updated_at | datetime | Last update timestamp |

### List Clips

```
GET /clips
```

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| limit | integer | 20 | Max results |
| offset | integer | 0 | Skip results |
| session_id | uuid | - | Filter by session |
| channel_id | string | - | Filter by channel |
| status | string | - | Filter by status |
| is_favorite | boolean | - | Filter favorites |

**Response:** `200 OK` - Paginated clip list

### Create Clip

```
POST /clips
```

Creates a clip and queues it for generation.

**Request Body:**
```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "channel_id": "endzone",
  "title": "Touchdown Run Q2",
  "start_time": "2024-09-01T14:23:15Z",
  "duration_seconds": 12.5
}
```

**Response:** `201 Created` - Clip object (status: pending)

### Get Clip

```
GET /clips/:id
```

**Response:** `200 OK` - Clip object

### Update Clip

```
PATCH /clips/:id
```

**Request Body:**
```json
{
  "title": "Updated Title",
  "is_favorite": true
}
```

**Response:** `200 OK` - Updated clip

### Delete Clip

```
DELETE /clips/:id
```

**Response:** `204 No Content`

### Toggle Favorite

```
POST /clips/:id/favorite
```

Toggles the `is_favorite` flag.

**Response:** `200 OK`
```json
{
  "is_favorite": true
}
```

### Record View

```
POST /clips/:id/watch
```

Increments the view count.

**Response:** `200 OK`

### Download Clip

```
GET /clips/:id/download
```

**Response:** `200 OK` - Binary video file
- Content-Type: `video/mp4`
- Content-Disposition: `attachment`

### Get Thumbnail

```
GET /clips/:id/thumbnail
```

**Response:** `200 OK` - JPEG image
- Content-Type: `image/jpeg`

### Stream Clip

```
GET /clips/:id/stream
```

Supports range requests for seeking.

**Response:** `200 OK` or `206 Partial Content`
- Content-Type: `video/mp4`
- Accept-Ranges: `bytes`

### Upload Clip

```
POST /clips/upload
```

Used by go-video-capture to upload clips directly.

**Content-Type:** `multipart/form-data`

**Form Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| file | file | yes | Video file |
| session_id | uuid | yes | Target session |
| channel_id | string | yes | Source channel |
| play_id | string | no | Play identifier |
| title | string | no | Clip title |
| start_time | integer | yes | Unix timestamp (ms) |
| end_time | integer | yes | Unix timestamp (ms) |
| duration_seconds | float | yes | Duration |
| tags | json | no | Initial tags |

**Response:** `201 Created` - Clip object

### Get Clip Tags

```
GET /clips/:id/tags
```

**Response:** `200 OK` - Paginated tag list

---

## Channels

Channels represent video input sources (cameras).

### Channel Object

| Field | Type | Description |
|-------|------|-------------|
| id | string | Channel identifier (e.g., "endzone") |
| name | string | Display name |
| description | string | Description |
| input_type | string | e.g., "srt", "rtsp" |
| input_url | string | Stream URL |
| resolution | string | e.g., "1920x1080" |
| framerate | integer | FPS |
| bitrate | integer | Bitrate (kbps) |
| status | string | `inactive`, `active`, `error` |
| last_seen_at | datetime | Last heartbeat |
| error_message | string | Error details |
| created_at | datetime | Creation timestamp |
| updated_at | datetime | Last update timestamp |

### List Channels

```
GET /channels
```

**Response:** `200 OK`
```json
{
  "items": [Channel],
  "total": 4
}
```

### Create Channel

```
POST /channels
```

**Request Body:**
```json
{
  "id": "endzone",
  "name": "End Zone Camera",
  "description": "North end zone wide angle",
  "input_type": "srt",
  "input_url": "srt://mediamtx:8890?streamid=endzone",
  "resolution": "1920x1080",
  "framerate": 60,
  "bitrate": 5500
}
```

**Response:** `201 Created` - Channel object

### Get Channel

```
GET /channels/:id
```

**Response:** `200 OK` - Channel object

### Update Channel

```
PATCH /channels/:id
```

**Request Body:** (all fields optional)
```json
{
  "name": "Updated Name",
  "resolution": "1280x720"
}
```

**Response:** `200 OK` - Updated channel

### Delete Channel

```
DELETE /channels/:id
```

**Response:** `204 No Content`

### Activate Channel

```
POST /channels/:id/activate
```

Sets status to `active`.

**Response:** `200 OK` - Updated channel

### Deactivate Channel

```
POST /channels/:id/deactivate
```

Sets status to `inactive`.

**Response:** `200 OK` - Updated channel

### Heartbeat

```
POST /channels/:id/heartbeat
```

Updates `last_seen_at` timestamp. Used by capture processes to signal they're alive.

**Response:** `200 OK`
```json
{
  "acknowledged": true
}
```

---

## Tags

Tags are annotations attached to clips with football-specific metadata.

### Tag Object

| Field | Type | Description |
|-------|------|-------------|
| id | string (uuid) | Unique identifier |
| clip_id | string (uuid) | Parent clip |
| session_id | string (uuid) | Parent session |
| quarter | integer | Game quarter (1-4, OT) |
| down | integer | Down (1-4) |
| distance | integer | Yards to go |
| yard_line | integer | Field position |
| play_type | string | "Run", "Pass", "Kick", etc. |
| formation | string | Offensive formation |
| result | string | "Complete", "Incomplete", "Touchdown", etc. |
| yards_gained | integer | Yards on play |
| category | string | Custom category |
| labels | string[] | Custom labels |
| players | string[] | Player names/numbers |
| notes | string | General notes |
| coach_notes | string | Coach-specific notes |
| is_important | boolean | Flagged as important |
| is_reviewed | boolean | Marked as reviewed |
| tagged_by | string | Who created the tag |
| tagged_at | datetime | When tagged |
| created_at | datetime | Creation timestamp |
| updated_at | datetime | Last update timestamp |

### List Tags

```
GET /tags
```

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| limit | integer | Max results |
| offset | integer | Skip results |
| clip_id | uuid | Filter by clip |
| session_id | uuid | Filter by session |
| play_type | string | Filter by play type |
| category | string | Filter by category |
| is_important | boolean | Filter important |
| result | string | Filter by result |

**Response:** `200 OK` - Paginated tag list

### Create Tag

```
POST /tags
```

**Request Body:**
```json
{
  "clip_id": "550e8400-e29b-41d4-a716-446655440000",
  "session_id": "550e8400-e29b-41d4-a716-446655440001",
  "quarter": 2,
  "down": 3,
  "distance": 7,
  "yard_line": 35,
  "play_type": "Pass",
  "formation": "Shotgun",
  "result": "Complete",
  "yards_gained": 12,
  "players": ["12", "88"],
  "notes": "Great route by WR",
  "is_important": true,
  "tagged_by": "coach_smith"
}
```

**Response:** `201 Created` - Tag object

### Get Tag

```
GET /tags/:id
```

**Response:** `200 OK` - Tag object

### Update Tag

```
PATCH /tags/:id
```

**Request Body:** (all fields optional)
```json
{
  "notes": "Updated notes",
  "is_reviewed": true
}
```

**Response:** `200 OK` - Updated tag

### Delete Tag

```
DELETE /tags/:id
```

**Response:** `204 No Content`

### Search Tags

```
POST /tags/search
```

Full-text search across tags.

**Request Body:**
```json
{
  "query": "touchdown",
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "play_types": ["Pass", "Run"],
  "is_important": true,
  "limit": 20
}
```

**Response:** `200 OK` - Paginated tag list

### Mark Reviewed

```
POST /tags/:id/reviewed
```

Sets `is_reviewed` to true.

**Response:** `200 OK` - Updated tag

### Toggle Important

```
POST /tags/:id/important
```

Toggles the `is_important` flag.

**Response:** `200 OK`
```json
{
  "is_important": true
}
```

---

## WebSocket Events

Connect to `/ws` for real-time updates.

### Connection

```javascript
const ws = new WebSocket('ws://localhost:8080/ws');
```

### Event Format

```json
{
  "type": "event_type",
  "data": { ... },
  "timestamp": "2024-09-01T14:23:15Z"
}
```

### Event Types

| Event | Description | Data |
|-------|-------------|------|
| `clip_ready` | Clip finished processing | `{ clip_id, session_id }` |
| `clip_failed` | Clip processing failed | `{ clip_id, error }` |
| `session_start` | Session recording started | `{ session_id }` |
| `session_end` | Session recording ended | `{ session_id }` |
| `clip_segment_ready` | Ghost clip segment available | `{ clip_id, segment_index }` |
| `channel_status` | Channel status changed | `{ channel_id, status }` |

---

## System Endpoints

### Health Check

```
GET /health
```

**Response:** `200 OK`
```json
{
  "status": "healthy",
  "timestamp": "2024-09-01T14:23:15Z",
  "service": "video-system"
}
```

### API Status

```
GET /api/v1/status
```

**Response:** `200 OK`
```json
{
  "status": "ok",
  "version": "0.1.0"
}
```

---

## Error Codes

| HTTP Code | Description |
|-----------|-------------|
| 200 | Success |
| 201 | Created |
| 204 | No Content (deleted) |
| 400 | Bad Request (validation error) |
| 404 | Not Found |
| 500 | Internal Server Error |

---

## Rate Limits

No rate limits in development. Production limits TBD.

---

## Examples

### Create a Game Session and Start Recording

```bash
# Create session
curl -X POST http://localhost:8080/api/v1/sessions \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Week 5 vs Bears",
    "session_type": "game",
    "opponent": "Bears",
    "season": 2024,
    "week": 5
  }'

# Start recording
curl -X POST http://localhost:8080/api/v1/sessions/{session_id}/start
```

### Create a Clip from Recording

```bash
curl -X POST http://localhost:8080/api/v1/clips \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "550e8400-e29b-41d4-a716-446655440000",
    "channel_id": "endzone",
    "title": "Great TD Run",
    "start_time": "2024-09-01T14:23:15Z",
    "duration_seconds": 15
  }'
```

### Tag a Clip with Play Details

```bash
curl -X POST http://localhost:8080/api/v1/tags \
  -H "Content-Type: application/json" \
  -d '{
    "clip_id": "550e8400-e29b-41d4-a716-446655440001",
    "session_id": "550e8400-e29b-41d4-a716-446655440000",
    "quarter": 3,
    "down": 1,
    "distance": 10,
    "yard_line": 25,
    "play_type": "Run",
    "formation": "I-Form",
    "result": "Touchdown",
    "yards_gained": 25,
    "players": ["22"],
    "is_important": true
  }'
```
