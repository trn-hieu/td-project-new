---
paths: ["backend/src/**/*.java"]
---

# JPA / Spring Boot Rules

Applies to all entities, repositories, services, controllers and tests created or modified in `backend/`
(Spring Boot, MySQL, Flyway, Maven, base package `com.thanhdo.tdprojectbe`).

The DB is shared with the legacy system, so existing code follows the legacy schema. Follow the conventions
below for new code and for code you touch; do not mass-rewrite untouched files.

## JPA first (highest priority)

Existing code often writes queries as native SQL strings. **New data access must use JPA**, and native SQL is the
last resort. Pick the first option that works:

1. **Derived query** (`findByEmailAndDeleteFlag`, `existsBy…`, `countBy…`).
2. **JPQL `@Query`** against entities/fields (`SELECT e FROM Employee e WHERE e.deleteFlag = :deleteFlag`), never
   table or column names.
3. **`Specification` + `JpaSpecificationExecutor`** (or Criteria API) for dynamic filters. Never build `WHERE` by
   string concatenation.
4. **Projection** (interface projection, or a JPQL constructor expression into a `record`) when only some columns
   are needed. Do not fetch whole entities just to map a few fields.
5. **Native SQL**: only when JPA cannot express it, and the reason is stated in a comment directly above the query.
   Valid reasons: DB-specific function or stored function (`working_hours_accurate_8h`, `FIND_IN_SET`,
   `CONVERT_TZ`), CTE / window function, or a measured performance problem on a heavy report query.
   "Easier to write" or "ported from the legacy system" is not a reason.

Rules for the cases that remain native:
- Always named parameters (`:name`); never concatenate user input into SQL. Only constant fragments (as
  `EmployeeRepository.SEARCH_CONDITION`) may be concatenated.
- Prefer `@Query(nativeQuery = true)` on a Spring Data repository over `EntityManager.createNativeQuery`. Use
  `EntityManager` only when the SQL must be assembled dynamically and a `Specification` cannot do it.
- Paged native queries need an explicit `countQuery`.
- Keep native SQL inside the repository, never in a service or controller.
- Map results into a `record` / projection, not `Object[]`.
- New code must not use `JdbcTemplate` or raw JDBC.
- **Touching an existing native query**: if the change is more than a one-line fix and the query can be written
  in JPQL / `Specification`, convert it as part of the change. If you leave it native, do not extend it further
  (no new conditions or joins); say why in the summary.
- Soft-deleted rows must still be excluded in native queries (`delete_flag = '0'`).

## Package layout

Package by feature, not by layer: `<feature>/` holds the controller and service, with sub-packages
`entity/`, `repository/` and `dto/` where they exist (`employee/`, `branch/`, `auth/`…). Shared code is in
`common/` (`exception`, `response`, `time`), configuration in `config/`.
- Services are concrete classes annotated `@Service`; **no `XxxService` interface + `impl/` pair** unless there are
  at least two implementations.
- Follow the neighbouring feature before inventing a new structure.

## Entity

- Declare entities with `@Entity`, `@Table(name = "...")` (the real legacy table name), `@Getter`.
  `@Setter` is accepted where existing entities already use it (e.g. `Employee`), but new fields that must stay
  immutable or need invariants use an explicit domain method instead of a public setter.
- Forbid `@Data`, `@EqualsAndHashCode` and `@ToString` on entities: they trigger lazy loading and recursion.
- JPA needs a no-arg constructor: add `@NoArgsConstructor(access = AccessLevel.PROTECTED)` to new entities.
- Id: `@Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;` with `@Column(name = "...")` when
  the legacy column is not `id`.
- Composite keys: `@EmbeddedId` + `@MapsId`; the key class is `@Embeddable`, `Serializable`, with
  `@EqualsAndHashCode`, `@Getter` and `@NoArgsConstructor`.
- Always map every column explicitly with `@Column(name = "...")` (the DB is not generated from entities), with
  `nullable` and `length` matching the real schema.
- **Soft delete follows the legacy column**: `delete_flag` (`String`, `"0"` active, `"1"` deleted), exposed as
  constants on the entity (`Employee.DELETE_FLAG_ACTIVE` / `DELETE_FLAG_DELETED`). Deleting sets the flag, it
  never calls `delete()`. For a new table, ask before choosing a different soft-delete column.
- Every read query (derived, JPQL, Specification, native; list, count and exists) must filter on the flag.
- Dates: use the type the entity's neighbours use (`LocalDateTime`, columns are server-local time). Do not
  introduce `Instant` into a legacy table. Use `@CreationTimestamp` / `@UpdateTimestamp` or `@PrePersist` /
  `@PreUpdate` for audit columns.
- Relationships: `@ManyToOne(fetch = FetchType.LAZY)` is mandatory; forbid `FetchType.EAGER`. Many legacy tables
  reference each other by plain id columns (`employeeId`, `branch` code); prefer mapping the id column instead of
  a relationship unless the relationship is actually navigated.
- `@OneToMany` only when necessary, always with `mappedBy`.
- Never serialize an entity in an API response; use a DTO record.
- Do not access lazy relationships outside a transaction. Prevent N+1 with `@EntityGraph`, `JOIN FETCH` or
  projections.

## Schema and migration

- `spring.jpa.hibernate.ddl-auto=none`: Hibernate never changes the schema, and does not validate it either, so a
  wrong column name only fails at runtime. After mapping a new entity or column, run a repository test that
  reads it (see Tests).
- Every schema change is a new Flyway file in `src/main/resources/db/migration`, named
  `V<yyyyMMddHHmmss>__<snake_case_description>.sql` (e.g. `V20261012104500__create_refresh_tokens.sql`), written
  in MySQL syntax.
- Never edit a migration that has been applied to a shared environment. The history table is shared with the legacy
  system (`out-of-order`, `ignore-missing` are on), so do not renumber, delete or re-baseline.

## Repository

- Interface `XxxRepository extends JpaRepository<Xxx, Long>` in `<feature>/repository/` (or the feature root if
  the feature has no sub-packages). Composite keys use `JpaRepository<Xxx, XxxId>`.
- Dynamic search screens: `JpaSpecificationExecutor<Xxx>` and `Specification` helpers instead of a string-built
  query.
- List endpoints take `Pageable` and return `Page<>`. No unbounded `findAll()` on a table that can grow. The
  exception is a small reference table already read in full (e.g. branches, roles), and an export that
  intentionally reads everything; the latter passes `Pageable.unpaged()` explicitly.
- Bulk updates: `@Modifying(clearAutomatically = true)` + JPQL `@Query`.
- Never put `@Transactional` on a repository.

## Service

- Concrete `@Service` class with `@RequiredArgsConstructor`; no interface/impl pair.
- `@Transactional(readOnly = true)` on read methods, `@Transactional` on write methods (as in `EmployeeService`).
  Do not leave a method that writes without a transaction.
- Services take and return DTO records; entities do not leave the service layer. Mapping entity → DTO happens
  inside the transaction so lazy data is available.
- Not-found and business-rule violations throw `BusinessException(ErrorCode.X, "thông báo tiếng Việt")`
  (`ErrorCode.NOT_FOUND` for missing data). There is no `ResourceNotFoundException`. Never return `null` for
  a missing entity: use `orElseThrow`.
- Field-level errors (duplicate email, etc.) use the `BusinessException(code, message, fieldErrors)` overload so
  the UI can mark the right input.
- Reuse existing services (e.g. `CurrentEmployeeService`) before writing new data access for the same data.

## Controller

- `@RestController`, `@RequestMapping("/api/<feature>")`; call services only, never inject repositories.
- Return `ApiResponse<T>` built with `ApiResponse.ok(data, message)` (`common.response`). Binary downloads (Excel,
  files) return `ResponseEntity<byte[]>`. Errors are thrown, never returned: `GlobalExceptionHandler` renders them.
- Lists return `ApiResponse<PageResponse<T>>` via `PageResponse.from(page)`; `page` is 0-based.
- Request bodies and path/query constraints use `@Valid` / `@Validated`. DTOs are records with `jakarta.validation`
  constraints and Vietnamese messages.
- No business logic and no `try/catch` in controllers. Access rules live in `SecurityConfig`
  (authority-based), not scattered in controllers.
- Do not change an existing API contract (path, fields, message text the frontend relies on) unless asked.

## DTO / Mapper

- DTOs are `record`s in `<feature>/dto/`. Map by hand with a static factory (`XxxResponse.from(entity)`); no
  MapStruct.
- Everything a `from(entity)` touches must already be loaded.

## General

- Constructor injection via `@RequiredArgsConstructor` + `private final`. No `@Autowired`
  (`MsbProfileService` still has one; do not copy it).
- Configuration through `@ConfigurationProperties` / `@Value`; no hard-coded secrets, keys or URLs.
- Comments and user-facing messages are in Vietnamese, matching the codebase.
- Do not add dependencies without clear justification, and do not touch unrelated files.
- Follow `rules/design-principles.md` (YAGNI → KISS → DRY → SOLID).

## Tests

- A repository change (new query, new entity mapping) needs a repository test with `@DataJpaTest` (see
  `MsbProfileRepositoryTest`) that executes the query, since `ddl-auto=none` means mapping mistakes do not fail
  at startup.
- Service logic is tested with plain unit tests (see `AuthServiceTest`).
- Run `./mvnw test` (on Windows `mvnw.cmd test`) from `backend/`.

## Checklist before submitting

Review the code just written or modified and fix violations before declaring complete:

- [ ] Data access uses derived query / JPQL / `Specification` / projection; native SQL only where JPA cannot do it,
      with a justification comment, named parameters, and an explicit `countQuery` for paging.
- [ ] No string-concatenated user input in any query; no `JdbcTemplate`.
- [ ] A touched native query was converted to JPA where feasible, or left unchanged and not extended.
- [ ] Every read/count/exists query, native ones included, excludes soft-deleted rows (`delete_flag`).
- [ ] Entity: no `@Data` / `@EqualsAndHashCode` / `@ToString`; explicit `@Column` mapping; `@ManyToOne` is `LAZY`;
      protected no-arg constructor on new entities.
- [ ] Schema change has a new `V<timestamp>__<name>.sql`; no applied migration was edited.
- [ ] A repository test executes any new or changed query.
- [ ] List endpoints are paged (`Pageable` → `Page<>` → `PageResponse`), or the full read is deliberate.
- [ ] Service: `@Transactional(readOnly = true)` on reads, `@Transactional` on writes; no entity leaves the service.
- [ ] Not-found and rule violations throw `BusinessException` with an `ErrorCode`; nothing returns `null` for missing
      data.
- [ ] Controller injects no repository, has no business logic and no `try/catch`; returns `ApiResponse` via
      `ApiResponse.ok(...)`.
- [ ] No field `@Autowired`; no hard-coded configuration.
- [ ] `./mvnw test` passes; the final diff has no unrelated changes.
