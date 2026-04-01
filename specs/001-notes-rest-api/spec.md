# Feature Specification: Notes REST API

**Feature Branch**: `001-notes-rest-api`
**Created**: 2026-03-31
**Status**: Draft
**Input**: User description: "Для стороннего клиента (отображение и работа с заметками) необходимо сделать REST сервис который умеет: сохранять заметку (главная сущность), обновлять заметку, удалять заметку, получать полный список заметок, который фильтруется по авторизованной сессии."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Create a Note (Priority: P1)

A third-party client sends a request to save a new note. The service accepts the note
content, associates it with the caller's session identity, persists it, and returns the
created note with its assigned identifier and timestamps.

**Why this priority**: Creating notes is the foundational operation — without it, no
other operation has anything to act upon. This is the entry point for all data in the system.

**Independent Test**: Can be tested by sending a create request with valid session credentials
and verifying the response contains an ID and that the note appears in subsequent list queries
for the same session.

**Acceptance Scenarios**:

1. **Given** a client with a valid session token, **When** they submit a note with title and
   content, **Then** the service returns HTTP 201 with the created note including its unique
   ID and creation timestamp.
2. **Given** a client with a valid session token, **When** they submit a note with only a title
   (no content), **Then** the service accepts it and returns HTTP 201.
3. **Given** a client with no session token or an invalid one, **When** they attempt to create
   a note, **Then** the service returns HTTP 401 and does not persist any data.
4. **Given** a client with a valid session token, **When** they submit a note with an empty
   title, **Then** the service returns HTTP 422 with a validation error message.

---

### User Story 2 - List Notes Filtered by Session (Priority: P1)

A third-party client requests the full list of notes belonging to their session. The service
returns only notes associated with the caller's authenticated identity — notes from other
sessions are never returned.

**Why this priority**: Retrieving notes is the primary display action for any client. Session
isolation is the core data-privacy guarantee of the service.

**Independent Test**: Can be tested by creating notes under two different sessions and verifying
that each session's list call returns only its own notes.

**Acceptance Scenarios**:

1. **Given** a client with a valid session token that has created notes, **When** they request
   the list of notes, **Then** the service returns HTTP 200 with all notes belonging to that
   session only.
2. **Given** two different sessions each with their own notes, **When** session A requests its
   list, **Then** no notes from session B appear in the response.
3. **Given** a client with a valid session token but no notes yet created, **When** they
   request the list, **Then** the service returns HTTP 200 with an empty list.
4. **Given** a client with no session token or an invalid one, **When** they request the list,
   **Then** the service returns HTTP 401.

---

### User Story 3 - Update a Note (Priority: P2)

A third-party client sends an update request for an existing note, providing the note ID
and the new content. The service verifies ownership (the note belongs to the caller's session),
applies the changes, and returns the updated note.

**Why this priority**: Editing is secondary to creation and listing. Clients must first be
able to create and see notes before editing becomes meaningful.

**Independent Test**: Can be tested by creating a note, updating it, and verifying the changed
content is returned; also verify a different session cannot update the note.

**Acceptance Scenarios**:

1. **Given** a valid session with an existing note, **When** the client updates title or
   content, **Then** the service returns HTTP 200 with the updated note and a new
   `updated_at` timestamp.
2. **Given** a valid session, **When** the client attempts to update a note that belongs to a
   different session, **Then** the service returns HTTP 404 (note not found for this session).
3. **Given** a valid session, **When** the client attempts to update a note that does not
   exist, **Then** the service returns HTTP 404.
4. **Given** a valid session, **When** the client sends an update with an empty title,
   **Then** the service returns HTTP 422 with a validation error.

---

### User Story 4 - Delete a Note (Priority: P2)

A third-party client sends a delete request for a note by ID. The service confirms the note
belongs to the caller's session and permanently removes it.

**Why this priority**: Deletion completes the full lifecycle. It is secondary to creation,
listing, and editing.

**Independent Test**: Can be tested by creating a note, deleting it, and verifying it no
longer appears in the list; also verify a different session cannot delete it.

**Acceptance Scenarios**:

1. **Given** a valid session with an existing note, **When** the client deletes it by ID,
   **Then** the service returns HTTP 204 and the note no longer appears in the session's list.
2. **Given** a valid session, **When** the client attempts to delete a note belonging to a
   different session, **Then** the service returns HTTP 404.
3. **Given** a valid session, **When** the client attempts to delete a note that does not
   exist, **Then** the service returns HTTP 404.
4. **Given** a client with no session token, **When** they attempt to delete a note,
   **Then** the service returns HTTP 401.

---

### Edge Cases

- What happens when a session token has expired mid-request?
  → The service returns HTTP 401; no partial writes are committed.
- What happens when a note title exceeds a reasonable length limit?
  → The service returns HTTP 422 with a descriptive validation error.
- What happens when the persistence layer is unavailable during a write?
  → The service returns HTTP 503; the client can safely retry.
- What happens if the same note is deleted twice?
  → The second delete returns HTTP 404 (idempotent-friendly behaviour).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The service MUST allow an authenticated client to create a note with at minimum
  a title; content is optional.
- **FR-002**: The service MUST persist each note and associate it permanently with the
  session identity that created it.
- **FR-003**: The service MUST return the complete list of notes for the requesting session
  and MUST NOT include notes from other sessions.
- **FR-004**: The service MUST allow an authenticated client to update the title and/or
  content of a note they own.
- **FR-005**: The service MUST allow an authenticated client to permanently delete a note
  they own.
- **FR-006**: The service MUST reject all requests that do not carry a valid session token
  with HTTP 401.
- **FR-007**: The service MUST return HTTP 404 when a client references a note that does not
  belong to their session, regardless of whether the note exists for another session.
- **FR-008**: The service MUST validate input data and return HTTP 422 for invalid payloads
  (e.g., missing required fields, values exceeding length limits).
- **FR-009**: The service MUST return structured error responses so clients can display
  meaningful error messages.

### Key Entities

- **Note**: The primary entity. Attributes: unique identifier, title (required), content
  (optional), session owner identifier, creation timestamp, last-updated timestamp.
- **Session**: Represents the authenticated caller identity. Derived from the incoming
  authorization token on each request; not persisted by this service.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A client can successfully create, retrieve, update, and delete a note within a
  single interaction without receiving an unexpected error.
- **SC-002**: Notes from one session are never visible to another session — zero cross-session
  data leakage across all tested scenarios.
- **SC-003**: All four CRUD operations complete and return a response within 1 second under
  normal load conditions.
- **SC-004**: Invalid or missing authorization is rejected 100% of the time across all
  endpoints — no unauthorized data access in any tested scenario.
- **SC-005**: The service returns actionable, human-readable error messages for at least the
  following failure categories: unauthorized access, resource not found, and invalid input.

## Assumptions

- A session token is supplied by the client in the `Authorization` HTTP header using the
  Bearer scheme (`Authorization: Bearer <token>`). The service validates this token but does
  not issue tokens — token issuance is handled by an external identity provider or auth service.
- A "session" maps to a stable, unique identity derived from the validated token (e.g., a
  user ID or client ID embedded in the token payload).
- Note content is plain text; rich text or markdown rendering is out of scope for this service.
- Pagination of the notes list is out of scope for v1 — the full list is always returned.
- Soft-delete (archive) is out of scope — deletion is permanent.
- The service operates per-session (no shared notes, no collaboration features) in this version.
- File or image attachments are out of scope for this version.
