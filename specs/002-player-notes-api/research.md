# Research: Player Notes API

**Branch**: `002-player-notes-api` | **Date**: 2026-04-01
**Input**: Feature spec + clarifications (JWT local validation, hybrid expiry, Java 21 + Spring Boot 3.x)

---

## Decision 1 — JWT Validation Strategy

**Decision**: HS256 (HMAC-SHA256) symmetric key, validated locally via `NimbusJwtDecoder.withSecretKey()`.

**Rationale**: Single microservice with a shared secret — no JWK endpoint or key-pair management
needed. Simpler config, no outbound network dependency per request.

**Alternatives considered**:
- RSA asymmetric: suitable for multi-service architectures where auth server keeps private key.
  Overkill for a single service with a shared secret.
- Custom JWT filter with `jjwt`: more code, no additional benefit over Spring Security 6 native support.

**Key implementation note**:
No YAML property path exists for symmetric keys — a `JwtDecoder` bean must be declared explicitly:
```java
NimbusJwtDecoder.withSecretKey(key).macAlgorithm(MacAlgorithm.HS256).build()
```
Secret must be ≥ 32 bytes (256 bits). Inject via environment variable / `application.yml` property,
never hardcode.

---

## Decision 2 — User Identity Extraction

**Decision**: Use `jwt.getSubject()` — Spring Security 6 automatically maps the JWT `sub` claim
to the principal. Accessible via `@AuthenticationPrincipal Jwt jwt` in controllers.

**Rationale**: Standard Spring Security 6 convention. No custom converter required.

---

## Decision 3 — JSON Object Validation

**Decision**: `objectMapper.readTree(content)` + `node.isObject()` using Spring Boot's
auto-configured `ObjectMapper` bean.

**Rationale**: Handles all rejection cases in one step — malformed JSON throws
`JsonProcessingException`, and `isObject()` returns `false` for arrays, primitives, and JSON null.

**Implementation pattern**:
```java
JsonNode node = objectMapper.readTree(content);   // throws on invalid JSON
if (node == null || !node.isObject()) { /* reject */ }
```

---

## Decision 4 — Content Size Check (10 KB)

**Decision**: `content.getBytes(StandardCharsets.UTF_8).length` — byte-accurate UTF-8 measurement.

**Rationale**: `String.length()` counts UTF-16 code units and undercounts multi-byte characters
(emoji, CJK). UTF-8 byte count is what the spec requires. Allocation overhead is negligible at ≤ 10 KB.

---

## Decision 5 — Uniqueness Enforcement (author + uniqCode, active notes only)

**Decision**: Application-level check + DB unique constraint as fallback.

**Primary guard** — before create, query:
```
existsByAuthorIdAndUniqCodeAndExpiresAtAfter(authorId, uniqCode, Instant.now())
```
Returns 409 if an active note already exists for the pair.

**DB safety net** — a unique index on `(author_id, uniq_code)` cannot use `expires_at > NOW()`
as a partial-index predicate (PostgreSQL requires immutable expressions). Therefore the DB
constraint is not applicable for time-based uniqueness. The app-level check is the primary guard.
Concurrent duplicate creates are handled by catching any DB integrity violation and mapping it
to HTTP 409 as a final fallback.

**Alternatives considered**:
- Static `is_active` boolean column with a DB partial unique index `WHERE is_active = TRUE`:
  more complex schema, requires the cleanup job to flip the flag. Rejected for v1 (YAGNI).

---

## Decision 6 — Expired Note Cleanup (Background Job)

**Decision**: Spring `@Scheduled(cron = ...)` on a `@Component` class; `@EnableScheduling`
on the main application class.

**Schedule**: Daily at 02:00 (`0 0 2 * * *`), configurable via `app.cleanup.cron` property.

**Implementation**:
```java
@Scheduled(cron = "${app.cleanup.cron:0 0 2 * * *}")
public void purgeExpired() {
    noteRepository.deleteAllByExpiresAtBefore(Instant.now());
}
```

**Rationale**: Low-frequency cleanup is sufficient given lazy read-time filtering. Daily off-peak
run avoids any write contention during business hours.

---

## Decision 7 — List Ordering

**Decision**: `ORDER BY created_at DESC` (newest first). Enforced at the repository query level,
not post-fetch in Java.

---

## Decision 8 — Storage Profile

**Decision**: H2 in-memory for local dev (`spring.datasource.url=jdbc:h2:mem:notes`),
PostgreSQL for production. Spring Data JPA handles both via the same entity/repository code.
Schema managed via `spring.jpa.hibernate.ddl-auto=create-drop` (dev) / `validate` (prod)
with Flyway migrations for production schema management.

---

## Summary Table

| # | Topic | Decision |
|---|---|---|
| 1 | JWT algorithm | HS256, `NimbusJwtDecoder.withSecretKey()` |
| 2 | User identity | `jwt.getSubject()` via `@AuthenticationPrincipal Jwt` |
| 3 | JSON validation | `readTree()` + `isObject()` |
| 4 | Size check | `getBytes(UTF_8).length` |
| 5 | Uniqueness | App-level `existsBy...ExpiresAtAfter(now)` + 409 on DB violation |
| 6 | Cleanup job | `@Scheduled(cron)` + `@EnableScheduling`, daily at 02:00 |
| 7 | List order | `ORDER BY created_at DESC` at query level |
| 8 | Storage | H2 (dev) / PostgreSQL (prod), Flyway for prod migrations |
