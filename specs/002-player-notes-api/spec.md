# Feature Specification: Player Notes API

**Feature Branch**: `002-player-notes-api`
**Created**: 2026-04-01
**Status**: Draft
**Input**: User description: "Необходимо реализовать микросервис заметок для внешнего клиента. Клиент на своей стороне должен иметь функционал добавления новой заметки, получение полного списка заметок с фильтрацией по авторизованной сессии, обновление и удаления заметки, удаление всех заметок. Каждая заметка должна быть уникальна в разрезе автор заметки и uniqCode цели заметки. Цель заметки - доп запись об игроке, то есть для одного автора может быть только одна заметка про конкретного игрока. Каждая заметка имеет срок своей жизни - 30 дней со дня создания, при этом пользователь может продлить срок жизни заметки до 30 дней неограниченное количество раз. Ограничения: пользователь может иметь максимум 100 активных заметок - то есть максимально 100 активных заметок о 100 различных игроках. Каждая запись может быть не более 10кб. Клиент (сторонняя система) может сохранять различную информацию в сущности заметки в виде json объекта, наполнение которого наш микросервис должен контролировать только с точки зрения проверки валидации json объекта."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Create a Note About a Player (Priority: P1)

An authorized client submits a new note about a specific player. The service validates the
request, checks that the author does not already have an active note for that player,
enforces the 100-note limit, and persists the note with a 30-day expiry window.

**Why this priority**: Creating notes is the foundational operation — without it no other
operation has anything to act on. It also encodes the core business rules (uniqueness,
limits, TTL) that define the product.

**Independent Test**: Send a create request with valid credentials and a new player code.
Verify the response contains an assigned ID and expiry date, and the note appears in the
subsequent list for that session.

**Acceptance Scenarios**:

1. **Given** an authorized client with fewer than 100 active notes, **When** they submit a
   note with a valid `uniqCode` and valid JSON object content, **Then** the service returns
   HTTP 201 with the created note including its ID, `expiresAt` (30 days from now), and
   timestamps.
2. **Given** an authorized client, **When** they submit a note with an empty JSON object `{}`
   as content, **Then** the service accepts it and returns HTTP 201.
3. **Given** an authorized client who already has an active note for a given player (`uniqCode`),
   **When** they attempt to create another note for the same player, **Then** the service
   returns HTTP 409 Conflict and no duplicate is persisted.
4. **Given** an authorized client who already has 100 active notes, **When** they attempt to
   create a new note, **Then** the service returns HTTP 422 with a limit-exceeded error.
5. **Given** an authorized client, **When** they submit content that is not a valid JSON
   object (invalid JSON, JSON array, JSON primitive, or null), **Then** the service returns
   HTTP 422 with a validation error.
6. **Given** an authorized client, **When** they submit content whose size exceeds 10 KB,
   **Then** the service returns HTTP 422 with a size-limit error.
7. **Given** a client with no session token or an invalid one, **When** they attempt to
   create a note, **Then** the service returns HTTP 401.

---

### User Story 2 - List Notes Filtered by Session (Priority: P1)

An authorized client requests the complete list of their active notes. The service returns
only notes belonging to the caller's session identity — expired notes and other sessions'
notes are never included.

**Why this priority**: Retrieving the note list is the primary read operation for any
client display. Session isolation is the core privacy guarantee of the service.

**Independent Test**: Create notes under two different sessions, then verify each session's
list contains only its own non-expired notes.

**Acceptance Scenarios**:

1. **Given** an authorized client with active notes, **When** they request the list, **Then**
   the service returns HTTP 200 with all active notes belonging to that session only,
   ordered by `createdAt` descending (newest first).
2. **Given** two sessions each with their own notes, **When** session A requests its list,
   **Then** no notes from session B appear.
3. **Given** an authorized client with no active notes yet, **When** they request the list,
   **Then** the service returns HTTP 200 with an empty list.
4. **Given** an authorized client with a note whose TTL has expired, **When** they request
   the list, **Then** the expired note does not appear in the response.
5. **Given** a client with no session token or an invalid one, **When** they request the
   list, **Then** the service returns HTTP 401.

---

### User Story 3 - Update a Note (Priority: P2)

An authorized client updates the content of an existing active note they own. The service
verifies ownership, replaces the content, and returns the updated note. The note's expiry
is not affected by an update.

**Why this priority**: Editing is secondary to creation and listing. Clients must first
create and view notes before editing is meaningful.

**Independent Test**: Create a note, update its content, verify the new content is returned
and the note still appears in the list with the same ID and original expiry.

**Acceptance Scenarios**:

1. **Given** an authorized client with an existing active note, **When** they submit updated
   JSON object content, **Then** the service returns HTTP 200 with the updated note and a
   refreshed `updatedAt` timestamp (expiry unchanged).
2. **Given** an authorized client, **When** they attempt to update a note belonging to a
   different session, **Then** the service returns HTTP 404.
3. **Given** an authorized client, **When** they attempt to update a note that does not
   exist, **Then** the service returns HTTP 404.
4. **Given** an authorized client, **When** they submit updated content that is not a valid
   JSON object, **Then** the service returns HTTP 422.
5. **Given** an authorized client, **When** they submit updated content exceeding 10 KB,
   **Then** the service returns HTTP 422.

---

### User Story 4 - Delete a Note (Priority: P2)

An authorized client deletes a specific note by its identifier. The service verifies
ownership and permanently removes the note, freeing the slot in the 100-note limit.

**Why this priority**: Deletion completes the note lifecycle and allows the user to free
capacity for new notes.

**Independent Test**: Create a note, delete it, verify it no longer appears in the list,
and confirm the session can now create a new note for the same player.

**Acceptance Scenarios**:

1. **Given** an authorized client with an existing active note, **When** they delete it by
   ID, **Then** the service returns HTTP 204 and the note no longer appears in the list.
2. **Given** an authorized client, **When** they attempt to delete a note belonging to a
   different session, **Then** the service returns HTTP 404.
3. **Given** an authorized client, **When** they attempt to delete a note that does not
   exist, **Then** the service returns HTTP 404.
4. **Given** an authorized client, **When** they delete the same note a second time,
   **Then** the service returns HTTP 404 (idempotent-friendly).
5. **Given** a client with no session token, **When** they attempt to delete a note,
   **Then** the service returns HTTP 401.

---

### User Story 5 - Delete All Notes (Priority: P2)

An authorized client deletes all of their active notes in a single operation. The service
permanently removes every note belonging to the caller's session.

**Why this priority**: Bulk deletion enables fast cleanup and is especially useful for
clients that need to reset their note state entirely.

**Independent Test**: Create multiple notes under a session, call delete-all, verify the
list returns an empty collection immediately after.

**Acceptance Scenarios**:

1. **Given** an authorized client with multiple active notes, **When** they request delete-all,
   **Then** the service returns HTTP 204 and the subsequent list returns empty.
2. **Given** an authorized client with no notes, **When** they request delete-all, **Then**
   the service returns HTTP 204 (idempotent — no error).
3. **Given** a client with no session token, **When** they request delete-all, **Then**
   the service returns HTTP 401.

---

### User Story 6 - Extend Note Lifetime (Priority: P3)

An authorized client extends the remaining lifetime of a specific active note by 30 days
from the current moment. This operation can be performed an unlimited number of times.

**Why this priority**: Lifetime management is a secondary lifecycle-maintenance operation.
Core CRUD must be complete and stable before expiry extension becomes useful.

**Independent Test**: Create a note, record its `expiresAt`, call extend, verify the new
`expiresAt` is 30 days from the time of the extend call (not from the original expiry).

**Acceptance Scenarios**:

1. **Given** an authorized client with an active note, **When** they call extend on it,
   **Then** the service returns HTTP 200 with `expiresAt` set to 30 days from the current
   moment.
2. **Given** an authorized client, **When** they extend the same note multiple times,
   **Then** each extension resets `expiresAt` to 30 days from the most recent call.
3. **Given** an authorized client, **When** they attempt to extend a note belonging to a
   different session, **Then** the service returns HTTP 404.
4. **Given** an authorized client, **When** they attempt to extend an already-expired note,
   **Then** the service returns HTTP 404 (expired notes are treated as non-existent).
5. **Given** a client with no session token, **When** they attempt to extend a note,
   **Then** the service returns HTTP 401.

---

### Edge Cases

- What happens when a session token expires mid-request?
  → The service returns HTTP 401; no partial writes are committed.
- What happens when note content is `null` instead of a JSON object?
  → The service returns HTTP 422 — null is not a valid JSON object.
- What happens when content is a JSON array (e.g., `[]`)?
  → The service returns HTTP 422 — only JSON objects are accepted.
- What happens when the same `uniqCode` is used after the original note for that player
  was deleted or expired?
  → A new note can be created — uniqueness constraint applies only to active notes.
- What happens if `uniqCode` is an empty string?
  → The service returns HTTP 422 — `uniqCode` is required and must be non-blank.
- What happens when the persistence layer is unavailable during a write?
  → The service returns HTTP 503; the client can safely retry.
- What happens if delete-all is called while another create is in flight for the same session?
  → The service guarantees atomicity per operation; partial states are not exposed.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The service MUST allow an authorized client to create a note identified by the
  combination of their session identity and a `uniqCode` (player identifier).
- **FR-002**: The service MUST reject creation of a note when the caller already has an active
  note with the same `uniqCode`, returning HTTP 409.
- **FR-003**: The service MUST reject creation when the caller already has 100 active notes,
  returning HTTP 422.
- **FR-004**: The service MUST persist each note with an expiry timestamp set to exactly 30
  days after creation.
- **FR-005**: The service MUST return the complete list of active (non-expired) notes for the
  requesting session, ordered by `createdAt` descending (newest first). The response MUST NOT
  include notes from other sessions or expired notes.
- **FR-006**: The service MUST allow an authorized client to replace the content of an active
  note they own without changing its expiry timestamp. `uniqCode` is immutable and MUST NOT
  be modifiable after creation.
- **FR-007**: The service MUST allow an authorized client to permanently delete a single active
  note they own.
- **FR-008**: The service MUST allow an authorized client to permanently delete all notes
  belonging to their session in a single atomic call.
- **FR-009**: The service MUST allow an authorized client to extend the lifetime of an active
  note by resetting its expiry to exactly 30 days from the moment of the call.
- **FR-010**: The service MUST validate that note content is a valid JSON object; content that
  is not valid JSON, a JSON array, a JSON primitive, or null MUST be rejected with HTTP 422.
- **FR-011**: The service MUST reject note content whose size exceeds 10 KB, returning HTTP 422.
- **FR-012**: The service MUST reject all requests that do not carry a valid JWT in the
  `Authorization: Bearer` header, returning HTTP 401. Token validation MUST be performed
  locally (no outbound auth call per request).
- **FR-013**: The service MUST return HTTP 404 when a client references a note that does not
  belong to their session, regardless of whether the note exists for another session.
- **FR-014**: The service MUST return structured error responses with actionable messages for
  all failure categories (401, 404, 409, 422, 503).
- **FR-015**: The service MUST run a background scheduled job that periodically purges expired
  notes from storage to maintain storage hygiene. The job frequency is an operational concern
  and does not affect API behaviour.

### Key Entities

- **Note**: The primary entity. Attributes: unique system identifier, session owner identity
  (derived from token), `uniqCode` (external player identifier provided by client — immutable
  after creation), content (JSON object, max 10 KB), creation timestamp, last-updated
  timestamp, expiry timestamp.
- **Session**: Represents the authenticated caller identity. Derived from the incoming
  authorization token on each request; not persisted by this service.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A client can create, retrieve, update, delete, and extend a note within a
  single interaction without receiving an unexpected error.
- **SC-002**: Notes from one session are never visible to another session — zero cross-session
  data leakage across all tested scenarios.
- **SC-003**: All CRUD operations and lifetime extension complete within 1 second under
  normal load conditions.
- **SC-004**: Invalid or missing authorization is rejected 100% of the time — no unauthorized
  access in any tested scenario.
- **SC-005**: The duplicate-note constraint (same session + `uniqCode`) is enforced 100% of
  the time — no two active notes for the same player from the same author.
- **SC-006**: The 100-note-per-session limit is enforced 100% of the time — no session
  exceeds 100 active notes.
- **SC-007**: Content exceeding 10 KB or failing JSON-object validation is rejected 100% of
  the time.
- **SC-008**: The service returns actionable, human-readable error messages for all failure
  categories: unauthorized access, resource not found, conflict, limit exceeded, and invalid
  input.

## Clarifications

### Session 2026-04-01

- Q: How does the service validate Bearer tokens? → A: JWT — service validates the token locally using a shared secret or public key (no outbound auth call per request).
- Q: How are expired notes handled in storage? → A: Hybrid — expired notes are filtered out on every read query (lazy); a background scheduled job periodically purges them for storage hygiene.
- Q: In what order should the note list be returned? → A: `createdAt` descending — newest note first.
- Q: Is `uniqCode` immutable after note creation? → A: Yes — `uniqCode` cannot be changed after creation; only note content is updatable.
- Q: Should the service apply per-session rate limiting beyond the 100-note cap? → A: No — the 100-note cap is the only throttle; no additional rate limiting in v1.

## Assumptions

- A session token is supplied in the `Authorization` HTTP header using the Bearer scheme
  (`Authorization: Bearer <token>`). The token is a JWT validated locally by the service
  using a shared secret or public key — no outbound call to an auth server is made per
  request. Token issuance is handled by an external identity or auth service.
- A "session" maps to a stable, unique identity derived from the validated token (e.g., a
  user ID or client ID embedded in the token payload).
- An "active note" is a note whose `expiresAt` is in the future. Expired notes are excluded
  from all read and mutation operations and are treated as non-existent by the API. Expired
  notes remain in storage until removed by a background scheduled cleanup job; they are
  filtered out on every read query (lazy expiry check).
- Extending a note's lifetime always resets `expiresAt` to exactly 30 days from the moment
  of the extend call — not from the current `expiresAt` value.
- The uniqueness constraint (session + `uniqCode`) applies only to active notes; after a
  note is deleted or expires, the same `uniqCode` can be used for a new note.
- `uniqCode` is a non-blank string provided by the external client. Its format and meaning
  are opaque to this service — it is treated as an opaque identifier.
- Content is validated only for structural correctness as a JSON object. The service does
  not interpret, schema-validate, or constrain the keys or values within the object.
- Content size is measured in bytes using UTF-8 encoding.
- The 100-note limit counts only active (non-expired) notes.
- Pagination of the notes list is out of scope for v1 — the full active list is always returned.
- Per-session rate limiting is out of scope for v1 — the 100-active-note cap serves as the
  only write throttle.
- Soft-delete (archive/trash) is out of scope — deletion is permanent.
- Collaboration features (shared notes between sessions) are out of scope.
- File or image attachments are out of scope.
