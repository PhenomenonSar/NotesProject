# POST /notes/{id}/extend — Extend Note Lifetime

## Description

Resets the note's `noteEOLTimestamp` to `now + 30 days` (from the moment of the call,
not from the current `noteEOLTimestamp`). Can be called an unlimited number of times.
Only active (non-expired) notes can be extended.

## Authentication

`Authorization: Bearer <JWT>` — required.

## Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | int | Note identifier |

## Request

No request body.

## Response

### 200 OK

```json
{
  "id": 1,
  "userId": 42,
  "uniqCode": "player-abc-123",
  "noteTimestamp": 1743580800,
  "noteEOLTimestamp": 1748851200,
  "data": {
    "rating": 8,
    "comment": "Strong left foot"
  }
}
```

`noteTimestamp` is unchanged. `noteEOLTimestamp` = current Unix time + 2592000 (30 days in seconds).

### Error Responses

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid JWT |
| 404 | Note not found, already expired, or belongs to a different session |
