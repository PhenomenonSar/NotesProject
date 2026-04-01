# GET /notes — List Active Notes

## Description

Returns all active (non-expired) notes belonging to the authenticated session,
ordered by `noteTimestamp` descending (newest first).

## Authentication

`Authorization: Bearer <JWT>` — required.

## Request

No request body. No query parameters.

## Response

### 200 OK

```json
[
  {
    "id": 2,
    "userId": 42,
    "uniqCode": "player-xyz-999",
    "noteTimestamp": 1743667200,
    "noteEOLTimestamp": 1746259200,
    "data": {
      "position": "striker",
      "potential": "high"
    }
  },
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
]
```

Returns an empty array `[]` if no active notes exist for the session.

Ordering: `noteTimestamp` descending — newest note first.

### Error Responses

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid JWT |
