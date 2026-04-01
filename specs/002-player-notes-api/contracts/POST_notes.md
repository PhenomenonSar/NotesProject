# POST /notes — Create Note

## Description

Creates a new note for the specified player. The pair `(userId, uniqCode)` must be unique
among active notes. At most 100 active notes are allowed per session.

## Authentication

`Authorization: Bearer <JWT>` — required. `userId` is derived from the JWT.

## Request

**Content-Type**: `application/json`

```json
{
  "uniqCode": "player-abc-123",
  "data": {
    "rating": 8,
    "comment": "Strong left foot"
  }
}
```

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `uniqCode` | string | yes | Non-blank; immutable after creation |
| `data` | object | yes | Valid JSON object; max 10 KB (UTF-8) |

## Response

### 201 Created

```json
{
  "id": 1,
  "userId": 42,
  "uniqCode": "player-abc-123",
  "noteTimestamp": 1743580800,
  "noteEOLTimestamp": 1746172800,
  "data": {
    "rating": 8,
    "comment": "Strong left foot"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | int | Auto-generated note identifier |
| `userId` | int | Caller identity from JWT |
| `uniqCode` | string | Player identifier (immutable) |
| `noteTimestamp` | int (Unix) | Creation time (seconds since epoch) |
| `noteEOLTimestamp` | int (Unix) | Expiry time = `noteTimestamp + 30 days` |
| `data` | object | Stored JSON content |

### Error Responses

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid JWT |
| 409 | Active note for this `uniqCode` already exists in the session |
| 422 | `uniqCode` is blank, `data` is not a valid JSON object, `data` exceeds 10 KB, or session has 100 active notes |
