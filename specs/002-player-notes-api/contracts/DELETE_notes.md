# DELETE /notes — Delete All Notes

## Description

Permanently removes all active notes belonging to the authenticated session.
Idempotent: returns 204 even if no notes exist.

## Authentication

`Authorization: Bearer <JWT>` — required.

## Request

No request body.

## Response

### 204 No Content

All notes permanently deleted. No response body.

### Error Responses

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid JWT |
