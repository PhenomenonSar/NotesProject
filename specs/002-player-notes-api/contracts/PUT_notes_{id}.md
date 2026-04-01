# PUT /notes/{id} — Update Note Content

## Description

Replaces the `data` field of an existing active note owned by the current session.
`uniqCode` is immutable and cannot be changed. Expiry (`noteEOLTimestamp`) is not affected.

## Authentication

`Authorization: Bearer <JWT>` — required.

## Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | int | Note identifier |

## Request

**Content-Type**: `application/json`

```json
{
  "data": {
    "rating": 9,
    "comment": "Improved finishing"
  }
}
```

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `data` | object | yes | Valid JSON object; max 10 KB (UTF-8) |

## Response

### 200 OK

```json
{
  "id": 1,
  "userId": 42,
  "uniqCode": "player-abc-123",
  "noteTimestamp": 1743580800,
  "noteEOLTimestamp": 1746172800,
  "data": {
    "rating": 9,
    "comment": "Improved finishing"
  }
}
```

`noteTimestamp` and `noteEOLTimestamp` are unchanged by an update.

### Error Responses

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid JWT |
| 404 | Note not found or does not belong to current session |
| 422 | `data` is not a valid JSON object or exceeds 10 KB |
