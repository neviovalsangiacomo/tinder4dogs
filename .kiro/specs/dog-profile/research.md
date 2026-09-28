# Research & Design Decisions

## Summary

- **Feature**: `dog-profile`
- **Discovery Scope**: Extension (light discovery)
- **Key Findings**:
  - The `dog` package has no service and no domain rules today; the controller
    talks to the repository directly. This feature introduces the first real
    domain rules in that package, so a service is now required by `AGENTS.md`.
  - There is no `@ControllerAdvice` anywhere in the project, and Spring Boot's
    default `ProblemDetail` body carries no field-level detail. Requirements
    1.4 and 4.2 demand that the offending attribute be identified, so error
    handling is new infrastructure, not a detail.
  - Typing the new vocabulary fields as enums in the request DTO would defeat
    those same requirements: Jackson fails before Bean Validation runs, and the
    resulting response cannot name the field.
  - No new third-party dependency is needed. Everything required is already on
    the classpath.

## Research Log

### Existing HTTP surface and its error conventions

- **Context**: The feature adds update and removal to a package that has only
  ever supported create and read, and it must not disturb matching.
- **Sources Consulted**: `dog/DogController.kt`, `match/MatchController.kt`,
  `match/MatchScoreService.kt`, `.kiro/steering/structure.md`.
- **Findings**:
  - `POST /api/dogs` returns 201 with no `Location` header; `GET /api/dogs`
    returns the whole table unpaged; `GET /api/dogs/{id}` returns 404 with an
    empty body. There is no `PUT`, `PATCH` or `DELETE`.
  - `match` translates absence to 404 and unusable stored data to 422, always
    with an empty body. `GET /api/matches/{id}` drops unscorable *candidates*
    silently but fails on an unscorable *subject*.
  - `structure.md` fixes the layering: controllers map HTTP to calls, services
    hold domain rules, repositories are Spring Data interfaces.
- **Implications**: The 404/422 split is an established convention in this
  codebase and the new endpoints should follow it rather than invent a third
  style. Empty error bodies, however, cannot satisfy requirements 1.4 and 4.2.

### How validation failures are actually reported today

- **Context**: Requirements 1.4 and 4.2 require the response to identify which
  attribute was rejected.
- **Sources Consulted**: `pom.xml`, `dog/DogController.kt`,
  `src/main/resources/application.yaml`, Spring Boot 4 error-handling defaults.
- **Findings**:
  - `spring-boot-starter-validation` is present and `@Valid` is applied to the
    create request, but the only constraints are `@NotBlank` on name and breed
    and `@Min(0)` on age.
  - There is no `@ControllerAdvice`, no `@ExceptionHandler`, and no
    `spring.mvc.problemdetails` configuration. A constraint violation therefore
    produces a bare `ProblemDetail`: `type`, `title`, `status`, `instance`, and
    no `errors`.
  - A malformed enum value fails earlier still, as
    `HttpMessageNotReadableException`, which is also reported as a bare 400.
- **Implications**: A `@RestControllerAdvice` that publishes a field-keyed
  `errors` map is required. It must be scoped to the `dog` package so that the
  existing empty-body conventions in `match` remain untouched.

### Vocabulary validation: where the failure must occur

- **Context**: Requirement 1.4 requires the offending attribute to be named
  when a size, energy or temperament value is outside its fixed set.
- **Sources Consulted**: Jackson enum deserialization behaviour, Jakarta Bean
  Validation `ConstraintValidator` contract.
- **Findings**:
  - If a request DTO field is typed as an enum, an unknown value aborts
    deserialization *before* Bean Validation executes. The framework never sees
    a constraint violation, so no field-level error can be produced.
  - If the field is typed as `String` and carries a constraint that checks
    membership of the enum, the failure arrives as a
    `MethodArgumentNotValidException` with the field path intact.
- **Implications**: Request DTOs accept `String` for the three vocabulary
  attributes and validate membership with a custom constraint. Responses and
  the entity remain strongly typed as enums, so the weak typing is confined to
  the request boundary.

### Enforcing one active profile per owner without a race

- **Context**: Requirement 5.3 rejects a create when the owner already has an
  active profile. Requirement 8.4 frees the slot when a profile is removed.
- **Sources Consulted**: PostgreSQL partial unique index behaviour,
  `.kiro/steering/tech.md`.
- **Findings**:
  - A service-level "check then insert" is not atomic. Two concurrent creates
    for the same owner can both pass the check.
  - PostgreSQL supports `CREATE UNIQUE INDEX ... WHERE removed_at IS NULL`,
    which enforces uniqueness over active rows only and permits any number of
    removed rows for the same owner.
- **Implications**: Both mechanisms are used. The service check produces the
  user-facing response; the index closes the race. The integrity violation the
  index raises must be mapped to the same response as the service check, or the
  race would surface as a 500.

### Disclosure in the occupied-owner rejection

- **Context**: Requirement 5.3 requires rejecting without disclosing that the
  identifier is in use. The owner identifier is an unverified claim (5.5), so
  any deterministic signal is an enumeration oracle.
- **Findings**:
  - 409 Conflict conventionally means the request conflicts with existing
    state. The status alone is the disclosure.
  - A 400 carrying no `errors` map is distinguishable from genuine field
    validation, which always carries one. The absence is itself a signal.
  - 422 is already this codebase's "well-formed request that cannot be
    processed" status, and this feature reuses it for other cases, so an
    occupied owner is not uniquely identifiable from it.
- **Implications**: 422 with a generic detail message and no `errors` map.
  The residual weakness — success versus rejection still distinguishes a free
  identifier from a taken one — is inherent while identity is unverified, and
  is recorded as a limitation that F-01 resolves.

### Tolerating the deliberately corrupt legacy row

- **Context**: Requirement 5.7 requires ownership backfill to preserve the
  stored row with a negative age; `tech.md` forbids "fixing" it.
- **Sources Consulted**: `changes/004-legacy-dog-row.sql`, `application.yaml`.
- **Findings**:
  - The row is `Nonna`, age `-3`. The `age` column deliberately has no `CHECK`
    constraint, and the comment in the changeset explicitly forbids adding one.
  - `ddl-auto: validate` means entity and schema must agree exactly at startup.
- **Implications**: Every new column is nullable or backfilled; no constraint
  is added to `age`; the backfill assigns `Nonna` an owner like every other
  row. Read paths must keep returning it (9.4).

## Architecture Pattern Evaluation

| Option | Description | Strengths | Risks / Limitations | Notes |
|--------|-------------|-----------|---------------------|-------|
| Service in `dog` package | Introduce `DogService` holding the new domain rules, controller delegates | Matches `structure.md` layering and `AGENTS.md`; unit-testable without Spring | One more file in the slice | **Selected.** `AGENTS.md` explicitly anticipates this: add the service the moment domain rules appear |
| Rules in controller | Keep the current controller-to-repository shape | No new file | Violates `AGENTS.md` ("domain rules never in controllers"); rules become untestable without MVC | Rejected |
| Separate `owner` package | Model ownership as its own concept with its own persistence | Cleanest path to v2 multi-dog | Requirements fixed ownership as a bare unverified identifier with no owner record; a package for one column is speculative | Rejected as premature |
| Removal via `canScore` | Express removal as unscorability so matching drops removed dogs through the existing filter | No repository change | Conflates corrupt data with lifecycle state; `MatchScoreService` is a scoring rule, not a lifecycle filter | Rejected |

## Design Decisions

### Decision: Soft delete represented by `removed_at`

- **Context**: Requirements 8.1–8.6 require removal to retain data, hide the
  profile from lists and matching, keep it readable by id, free the owner slot,
  and be idempotent.
- **Alternatives Considered**:
  1. `status` enum column (`ACTIVE`/`REMOVED`)
  2. Nullable `removed_at` timestamp
- **Selected Approach**: `removed_at TIMESTAMP NULL`; removed is defined as
  `removed_at IS NOT NULL`.
- **Rationale**: It answers "is it removed" and "when was it removed" with one
  column, and it is exactly what a partial unique index needs as its predicate.
- **Trade-offs**: A timestamp admits no future third state without migration;
  none is in scope.
- **Follow-up**: Idempotent removal must not overwrite the original timestamp
  (8.6).

### Decision: Partial unique index as the ownership backstop

- **Context**: Requirement 5.3 under concurrency.
- **Alternatives Considered**:
  1. Service check only
  2. Plain unique constraint on `owner_id`
  3. Service check plus partial unique index
- **Selected Approach**: Option 3, with the integrity violation mapped to the
  same 422 as the service check.
- **Rationale**: A plain unique constraint would forbid a removed profile and a
  new one sharing an owner, contradicting 8.4.
- **Trade-offs**: The rule is stated in two places and both must move together.
- **Follow-up**: A test must pin that the mapped violation is a 422, not a 500.

### Decision: String-typed vocabulary fields on requests only

- **Context**: Requirement 1.4, against Jackson's deserialization order.
- **Selected Approach**: Requests carry `String`; a custom constraint validates
  membership; entity and responses use enums.
- **Rationale**: The only placement that yields a field-named error.
- **Trade-offs**: One conversion step from validated string to enum in the
  service.
- **Follow-up**: The constraint message must list the permitted values.

### Decision: PATCH semantics for completing a profile

- **Context**: Requirement 6.5 requires setting one optional attribute without
  supplying the others; 3.5 requires photo lists to be replaced wholesale.
- **Alternatives Considered**:
  1. `PUT` with full representation
  2. `PATCH` with absent-means-unchanged fields
- **Selected Approach**: `PATCH`, with an explicitly present `photos` list
  replacing the stored list in full.
- **Rationale**: `PUT` would force clients to resend every attribute, which is
  precisely what 6.5 rules out.
- **Trade-offs**: Absent and explicit-null are distinct, which the DTO must
  model deliberately.
- **Follow-up**: Clearing an optional attribute is out of scope; no requirement
  asks for it, and conflating it with "unchanged" would be a silent guess.

### Decision: Removed profiles are absent from matching

- **Context**: Requirement 8.2 excludes removed profiles from match candidacy;
  8.3 keeps them readable by id on the profile endpoint.
- **Selected Approach**: Match endpoints resolve dogs through active-only
  repository lookups, so a removed dog yields 404 there, while
  `GET /api/dogs/{id}` still returns it with its removal indicated.
- **Rationale**: Removal withdraws a dog from the matching domain; it does not
  corrupt it, so 422 would misdescribe it.
- **Trade-offs**: `match` changes in a spec named for `dog`. The dependency
  direction is unchanged (`match` → `dog`), and `AGENTS.md` requires the commit
  message to say so.
- **Follow-up**: `MatchScoreService` is not touched (9.3).

### Decision: Deterministic synthetic owners for existing rows

- **Context**: Requirements 5.6 and 5.7.
- **Alternatives Considered**:
  1. `gen_random_uuid()` per row
  2. Deterministic `legacy-<id>`
- **Selected Approach**: Deterministic, derived from the primary key.
- **Rationale**: Reproducible across environments, trivially distinct, and
  legible when reading the table. A random backfill makes two databases
  disagree for no benefit.
- **Trade-offs**: The synthetic identifiers are recognizable as synthetic,
  which is desirable here.
- **Follow-up**: The generated values must satisfy the same owner-identifier
  format the API enforces.

## Synthesis Outcomes

- **Generalization**: Photos, temperaments and preferences are all ordered or
  unordered child collections of one dog. Only photos need order, so only
  photos carry a position column; no shared abstraction is introduced for three
  collections with different semantics.
- **Build vs. adopt**: Nothing here warrants a new dependency. Bean Validation
  supplies the constraint contract, PostgreSQL supplies partial unique indexes,
  Spring supplies `ProblemDetail`, and `spring-boot-starter-test` already
  bundles Mockito for the repository-dependent service tests. No library was
  rejected because none was needed.
- **Simplification**: No owner entity, table or endpoint, because requirements
  fixed ownership as a bare identifier. No temperament count cap, because no
  requirement asks for one. No completeness column, because 7.4 requires it to
  follow the data, which a derived value does by construction.

## Risks & Mitigations

- Race on concurrent creates for one owner — partial unique index, with the
  violation mapped to the same 422 as the service check.
- `ddl-auto: validate` fails startup on any entity/schema drift — every new
  column nullable or backfilled within the same changeset file, and the entity
  mapping reviewed against the SQL before the changeset is considered done.
- Lazy loading of new collections with `open-in-view: false` — new collections
  are fetched eagerly, matching the existing `preferences` mapping.
- The entire `dog` package is currently untested, so continuity guarantees
  (9.1–9.5) rest on nothing — new unit tests for the service and for match
  filtering are part of this feature, not a follow-up.
- Scope creep into F-03 and F-04 — mating attributes stay optional and
  unscored; `MatchScoreService` is explicitly out of boundary.

## References

- `docs/PRD.md` — F-02 Dog profile, and F-03/F-04 for what is deliberately deferred
- `.kiro/steering/tech.md` — Liquibase ownership, `ddl-auto: validate`, `open-in-view: false`, testing conventions
- `.kiro/steering/structure.md` — package-per-concept, layer responsibilities, naming
- `AGENTS.md` — service placement, `require` at function top, changeset immutability
