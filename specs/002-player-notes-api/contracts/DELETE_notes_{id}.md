# DELETE /notes/{id} — Delete a Single Note

## Description

Permanently removes the note identified by `{id}` if it belongs to the current session.
The operation is idempotent-friendly: a second delete on the same ID returns 404.

## Authentication

`Authorization: Bearer <JWT>` — required.

## Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | int | Note identifier |

## Request

No request body.

## Response

### 204 No Content

Note permanently deleted. No response body.

### Error Responses

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid JWT |
| 404 | Note not found, already deleted, or belongs to a different session |
