# Requirements Document

## Project Description (Input)

**Source:** `docs/PRD.md` — F-02 Dog profile, Priority P0. Personas served: 1
(private owner looking for a playmate) and 2 (private owner looking for a
mating partner).

### Who has the problem

Private dog owners in Switzerland and Italy who want to find a compatible
playmate or mating partner for their dog. They cannot currently describe their
dog in enough detail for the app to suggest anything relevant: park encounters
are random, and social-media groups offer no way to filter by size, energy,
temperament, age, breed, pedigree, neutering or health.

### Current situation

The service stores a `Dog` with five fields only — `name`, `breed`, `gender`,
`age`, and a free-text `preferences` set (`dog/Dog.kt`). The HTTP surface is
create and read; there is no update and no delete (`dog/DogController.kt`).

Concretely, against F-02:

- **Missing attributes.** Size, energy level and character/temperament have no
  fields. Neither do the mating attributes: pedigree, neutering, health status.
- **No photos.** No field, no storage, no reference of any kind.
- **No owner.** There is no owner entity and no ownership column anywhere in
  the schema, so "one dog per owner" cannot be expressed, let alone enforced.
  F-01 (passwordless login) is not implemented.
- **Profiles cannot be completed over time.** Without update, a profile is
  whatever it was at creation, which makes the "percentage of complete
  profiles" metric unmeasurable in practice.

### What should change

An owner can create, read, update and remove the profile of exactly one dog,
carrying the full F-02 attribute set, so that F-04 scoring has real data to
work with.

### Scope decisions

These were settled during initialization and requirements clarification, and
they bind the spec:

1. **Owner identity — caller-supplied opaque identifier.** There is no owner
   record and no owner endpoint. A profile carries an owner identifier that the
   caller supplies. Real authentication remains F-01's responsibility.
2. **Photos — references only.** The profile holds an ordered list of photo
   references, at most six, the first being the primary image. Upload, binary
   storage and image processing are out of scope.
3. **`preferences` — kept, alongside the new structured attributes.** The
   boundary is documented, not enforced by code: *`preferences` describes what
   the dog likes to do* (the seed data uses `parks`, `fetch`, `naps`, `sofa`,
   `cars`, `food`); *the new attributes describe what the dog is* (size, energy
   level, temperament).

   The overlap is latent rather than actual. Nothing prevents an owner from
   entering `"energetic"` or `"small"` as a preference, and if trait-like values
   leak into that set they are silently scored at `POINTS_PER_PREFERENCE` (5,
   capped at 15) by `match/MatchScoreService.kt`, double-counting an attribute
   that F-04 will weight deliberately elsewhere. Scoring is **not** modified by
   this spec; reconciling it with the structured attributes belongs to F-04.
4. **Full lifecycle — create, read, update and soft delete.** Removal retains
   the record but withdraws the profile from matching.
5. **Mating fields are optional, with no mode gating.** Pedigree, neutering and
   health status are storable and optional here. The rule that activating
   mating mode *requires* them belongs to F-03.
6. **Structured attributes use fixed vocabularies.** Size, energy level and
   temperament accept only values from defined lists, so that F-04 can compare
   them without a later normalization pass.
7. **Existing dogs are backfilled with synthetic owners.** Every dog already in
   the database receives a distinct generated owner identifier, so ownership is
   uniform and the one-dog-per-owner rule has no grandfathered exceptions.

### Constraints

- All mating attributes are **self-declared**. No document verification in v1.
- **One dog per owner** in v1. Multiple dogs per owner is v2, and the schema
  should not make that migration gratuitously painful.
- Schema changes are Liquibase changesets in plain SQL under
  `src/main/resources/db/changelog/changes/`, added to the master index. Never
  `ddl-auto: update`, and never edit a changeset that has already run. Files
  `001`, `003` and `004` are immutable; the next file is `005`.
- Existing seed data must keep working, including the deliberately corrupt
  legacy row with a negative age (`004-legacy-dog-row.sql`). New columns must
  therefore be nullable or carry a backfill; the negative-age row must not be
  "fixed", and no `CHECK` constraint may be added to `age`.
- Domain rules live in services, not in controllers and not in entities.
  Input that can be rejected is rejected with `require` at the top of the
  function.
- Prefer named constants to inline numbers in validation logic.

### Success metrics

- Percentage of complete profiles.
- Profile creation time.

### Out of scope

Authentication (F-01), play/mating modes (F-03), changes to the compatibility
score (F-04), geolocation (F-05), filters (F-06), multiple dogs per owner (v2),
breeder profiles (persona 3), and photo upload/binary storage.

---

## Introduction

This feature turns the dog record from a five-field stub into the profile that
F-02 describes: the attributes an owner needs to declare for their dog to be
matched sensibly, owned by exactly one owner, and editable over time so that a
profile can be started quickly and completed later.

Three things follow from that and shape every requirement below. Creation stays
deliberately cheap, because *profile creation time* is a success metric, so only
the four fields that exist today are mandatory. Everything new is optional and
added by updating, because *percentage of complete profiles* is the other
metric and it only means something if profiles can in fact be completed
incrementally. And the new descriptive attributes are drawn from fixed
vocabularies rather than free text, because F-04 has to compare them between two
dogs, which free text does not permit.

Ownership in this feature is a **claim, not a guarantee**. The owner identifier
is supplied by the caller and nothing verifies it, because the feature that
would verify it (F-01, passwordless login) does not exist yet. The requirements
below state the one-dog-per-owner rule in terms of that unverified claim and
make the resulting limitation explicit rather than implying a protection that
is not there.

## Boundary Context

- **In scope**: the dog profile's attributes, its photo references, its link to
  an owner identifier, the one-active-profile-per-owner rule, and the full
  create/read/update/remove lifecycle of a profile.

- **Out of scope**: verifying who the owner is (F-01); play and mating modes and
  any rule that makes mating attributes conditionally mandatory (F-03); how the
  new attributes affect the compatibility score (F-04); storing or serving photo
  bytes; supporting more than one dog per owner (v2).

- **Adjacent expectations**:
  - *Matching* consumes dog profiles today and must keep working unchanged.
    This feature expects matching to continue scoring on the attributes it
    already uses, and does not require it to consume the new ones. Removed
    profiles disappear from match results; that is this feature's obligation to
    express, not matching's to discover.
  - *Authentication* will later supply the owner identity. This feature expects
    to keep the identifier as an opaque value so that F-01 can replace its
    source without reshaping the profile.
  - The existing corrupt legacy row stays corrupt. This feature expects the
    profile lifecycle to tolerate stored data that its own validation would
    reject, in the same way matching already does.

## Requirements

### Requirement 1: Descriptive attributes for matching

**Objective:** As a private dog owner, I want to describe what my dog is like,
so that the app can suggest dogs that actually suit mine.

#### Acceptance Criteria

1. The Dog Profile Service shall accept a size for a dog, drawn from a fixed set
   of size values ordered from smallest to largest.
2. The Dog Profile Service shall accept an energy level for a dog, drawn from a
   fixed set of energy values ordered from lowest to highest.
3. The Dog Profile Service shall accept one or more temperament values for a
   dog, each drawn from a fixed set of temperament values.
4. If a request supplies a size, energy level or temperament value that is not
   in the fixed set for that attribute, then the Dog Profile Service shall
   reject the request and identify the offending attribute.
5. When a profile is read, the Dog Profile Service shall return every
   descriptive attribute that has been set, and shall indicate the absence of
   those that have not.
6. The Dog Profile Service shall keep the dog's existing free-text preferences
   distinct from the descriptive attributes, so that setting one never changes
   the other.

### Requirement 2: Self-declared mating attributes

**Objective:** As an owner looking for a mating partner, I want to declare my
dog's pedigree, neutering and health status, so that owners searching for a
partner can judge suitability.

#### Acceptance Criteria

1. The Dog Profile Service shall accept a pedigree declaration, a neutering
   declaration and a health status declaration for a dog.
2. The Dog Profile Service shall treat every mating attribute as optional, and
   shall accept a profile in which none of them is set.
3. When a mating attribute is supplied, the Dog Profile Service shall record it
   as self-declared by the owner.
4. The Dog Profile Service shall not require any supporting document or
   verification for a mating attribute.
5. Where mating attributes are absent, the Dog Profile Service shall still
   accept, store and return the profile.

### Requirement 3: Profile photos

**Objective:** As an owner, I want to show what my dog looks like, so that other
owners recognize and are drawn to my dog.

#### Acceptance Criteria

1. The Dog Profile Service shall accept an ordered list of photo references for
   a dog.
2. The Dog Profile Service shall accept a profile with no photo references.
3. If a request supplies more than the maximum permitted number of photo
   references, then the Dog Profile Service shall reject the request and state
   the maximum.
4. When photo references are returned, the Dog Profile Service shall return them
   in the order they were supplied, with the first reference designated as the
   primary photo.
5. When the photo references of a profile are replaced, the Dog Profile Service
   shall apply the new list in full, including its order.
6. The Dog Profile Service shall not accept, store or serve photo image data
   itself.

### Requirement 4: Creating a profile quickly

**Objective:** As an owner, I want to register my dog with only the essentials,
so that I can get started without filling in a long form.

#### Acceptance Criteria

1. When a create request supplies a name, breed, gender and age, the Dog Profile
   Service shall create the profile even if no other attribute is supplied.
2. If a create request omits the name, breed, gender or age, then the Dog
   Profile Service shall reject the request and identify the missing field.
3. If a create request supplies a negative age, then the Dog Profile Service
   shall reject the request.
4. When a profile is created, the Dog Profile Service shall report the created
   profile together with its identifier.
5. The Dog Profile Service shall treat size, energy level, temperament, photo
   references and all mating attributes as optional at creation.

### Requirement 5: One active profile per owner

**Objective:** As an owner, I want my dog profile bound to me, so that the app
knows whose dog it is and I cannot accidentally create duplicates.

#### Acceptance Criteria

1. The Dog Profile Service shall record an owner identifier for every profile it
   creates.
2. If a create request omits the owner identifier or supplies one that is empty
   or malformed, then the Dog Profile Service shall reject the request.
3. If a create request supplies an owner identifier that already has an active
   profile, then the Dog Profile Service shall reject the request without
   disclosing that the identifier is already in use.
4. When an owner identifier has no active profile, the Dog Profile Service shall
   permit that identifier to create one.
5. The Dog Profile Service shall treat the owner identifier as an unverified
   claim supplied by the caller, and shall not represent it as an authenticated
   identity.
6. Every dog profile that exists before this feature is introduced shall be
   assigned a distinct owner identifier, so that no stored profile is without an
   owner.
7. The Dog Profile Service shall preserve stored profiles whose data predates
   current validation, including a profile with a negative age, rather than
   rejecting or altering them when assigning ownership.

### Requirement 6: Completing a profile over time

**Objective:** As an owner, I want to add to and correct my dog's profile after
creating it, so that I can improve my matches without starting over.

#### Acceptance Criteria

1. When an update request supplies new values for a profile's attributes, the
   Dog Profile Service shall apply them and report the updated profile.
2. If an update request targets a profile that does not exist, then the Dog
   Profile Service shall report that the profile was not found.
3. If an update request supplies a value that would be rejected at creation,
   then the Dog Profile Service shall reject the update on the same grounds.
4. The Dog Profile Service shall not permit an update to change which owner a
   profile belongs to.
5. When an update sets an optional attribute that was previously absent, the Dog
   Profile Service shall record the new value without requiring any other
   absent attribute to be supplied.

### Requirement 7: Profile completeness

**Objective:** As the product owner, I want to know what share of profiles are
complete, so that I can tell whether owners are giving the matching algorithm
enough to work with.

#### Acceptance Criteria

1. The Dog Profile Service shall consider a profile complete when it has a name,
   breed, gender, age, size, energy level, at least one temperament value, and
   at least one photo reference.
2. The Dog Profile Service shall not require any mating attribute for a profile
   to be considered complete.
3. When a profile is read, the Dog Profile Service shall indicate whether that
   profile is complete.
4. When an update causes a profile to satisfy or stop satisfying the
   completeness conditions, the Dog Profile Service shall reflect the change the
   next time the profile is read.

### Requirement 8: Removing a profile

**Objective:** As an owner, I want to take my dog off the app, so that I stop
appearing to other owners when I no longer want to be matched.

#### Acceptance Criteria

1. When an owner removes their dog profile, the Dog Profile Service shall mark
   the profile as removed and retain its stored data.
2. When a profile has been removed, the Dog Profile Service shall exclude it
   from the list of profiles and from consideration as a match candidate.
3. While a profile is removed, the Dog Profile Service shall still return it when
   it is requested directly by its identifier, and shall indicate that it is
   removed.
4. When an owner's only profile has been removed, the Dog Profile Service shall
   permit that owner to create a new profile.
5. If a removal request targets a profile that does not exist, then the Dog
   Profile Service shall report that the profile was not found.
6. If a removal request targets a profile that is already removed, then the Dog
   Profile Service shall leave the profile unchanged.

### Requirement 9: Continuity with existing behaviour

**Objective:** As an owner already using the app, I want matching and profile
reading to keep working as before, so that adding new profile fields does not
break what I rely on.

#### Acceptance Criteria

1. The Dog Profile Service shall continue to return every active profile from
   the profile list, including profiles that predate the new attributes.
2. When a profile lacks the newly introduced attributes, the Dog Profile Service
   shall return it without error.
3. The Dog Profile Service shall not change how compatibility between two dogs
   is scored.
4. While a stored profile holds data that current validation would reject, the
   Dog Profile Service shall continue to return that profile when it is read
   directly.
5. When a profile is requested by an identifier that does not exist, the Dog
   Profile Service shall report that the profile was not found.
