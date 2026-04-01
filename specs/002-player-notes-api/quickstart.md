# Quickstart: Player Notes API

**Branch**: `002-player-notes-api` | **Date**: 2026-04-01

---

## Prerequisites

- Java 21+
- Maven 3.9+
- Docker (optional, for PostgreSQL in production profile)

---

## Run Locally (H2 dev profile)

```bash
# From project root
./mvnw spring-boot:run
```

The service starts on `http://localhost:8080` with an H2 in-memory database.
Schema is created automatically on startup and dropped on shutdown.

---

## Configuration

All configuration lives in `src/main/resources/application.yml`.

### Required environment variable

| Variable | Description | Example |
|---|---|---|
| `APP_SECURITY_JWT_SECRET` | Base64-encoded HS256 secret (min 32 bytes / 256 bits) | `dGVzdHNlY3JldGtleXRoYXRpczMyYnl0ZXNsb25n` |

Generate a secret (bash):
```bash
openssl rand -base64 32
```

### Dev profile (`application.yml` defaults)

```yaml
app:
  security:
    jwt-secret: ${APP_SECURITY_JWT_SECRET}
  cleanup:
    cron: "0 0 2 * * *"   # daily at 02:00

spring:
  datasource:
    url: jdbc:h2:mem:notes
  jpa:
    hibernate:
      ddl-auto: create-drop
```

### Production profile (`application-prod.yml`)

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/notes
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true
```

---

## Key API Calls

Obtain a JWT from your auth service, then:

```bash
TOKEN="<your.jwt.here>"

# Create a note
curl -X POST http://localhost:8080/notes \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"uniqCode":"player-123","content":{"rating":8,"comment":"Strong left foot"}}'

# List notes
curl http://localhost:8080/notes \
  -H "Authorization: Bearer $TOKEN"

# Update note content
curl -X PUT http://localhost:8080/notes/{id} \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content":{"rating":9,"comment":"Improved finishing"}}'

# Extend note lifetime (+30 days from now)
curl -X POST http://localhost:8080/notes/{id}/extend \
  -H "Authorization: Bearer $TOKEN"

# Delete a single note
curl -X DELETE http://localhost:8080/notes/{id} \
  -H "Authorization: Bearer $TOKEN"

# Delete all notes
curl -X DELETE http://localhost:8080/notes \
  -H "Authorization: Bearer $TOKEN"

# Health check
curl http://localhost:8080/actuator/health
```

---

## Project Structure

```
src/
└── main/
    ├── java/com/example/playernotes/
    │   ├── PlayerNotesApplication.java       # Entry point + @EnableScheduling
    │   ├── config/
    │   │   └── SecurityConfig.java           # JWT decoder + filter chain
    │   ├── controller/
    │   │   └── NoteController.java           # REST endpoints
    │   ├── service/
    │   │   └── NoteService.java              # Business logic + validation
    │   ├── repository/
    │   │   └── NoteRepository.java           # Spring Data JPA repository
    │   ├── entity/
    │   │   └── Note.java                     # JPA entity
    │   ├── dto/
    │   │   ├── CreateNoteRequest.java
    │   │   ├── UpdateNoteRequest.java
    │   │   └── NoteResponse.java
    │   ├── scheduler/
    │   │   └── ExpiredNoteCleanupJob.java    # @Scheduled purge job
    │   └── exception/
    │       ├── GlobalExceptionHandler.java   # @RestControllerAdvice
    │       ├── NoteNotFoundException.java
    │       ├── DuplicateNoteException.java
    │       └── NoteLimitExceededException.java
    └── resources/
        ├── application.yml
        ├── application-prod.yml
        └── db/migration/
            └── V1__create_notes_table.sql    # Flyway migration (prod)
pom.xml
```

---

## Validation Checklist (manual smoke test)

- [ ] `POST /notes` with no token → 401
- [ ] `POST /notes` with valid token and `{"uniqCode":"p1","content":{}}` → 201
- [ ] `POST /notes` with same `uniqCode` again → 409
- [ ] `POST /notes` with content that is a JSON array → 422
- [ ] `GET /notes` → 200 with the created note, newest first
- [ ] `PUT /notes/{id}` with new content → 200, `updatedAt` changed, `expiresAt` unchanged
- [ ] `POST /notes/{id}/extend` → 200, `expiresAt` ≈ now + 30 days
- [ ] `DELETE /notes/{id}` → 204, note gone from list
- [ ] `DELETE /notes` → 204, list empty
- [ ] `GET /actuator/health` → 200 `{"status":"UP"}`
