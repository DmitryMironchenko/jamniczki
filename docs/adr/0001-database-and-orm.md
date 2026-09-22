# ADR-0001: PostgreSQL + Prisma for persistence

- **Status:** Accepted
- **Date:** 2026-09-14
- **Deciders:** Dmitry Mironchenko
- **Context ticket:** [Persistence & data model — #2](https://github.com/DmitryMironchenko/jamniczki/issues/2) (map [#1](https://github.com/DmitryMironchenko/jamniczki/issues/1))

## Context

The Jamniczki MVP needs a persistence layer for a relational, location-aware social graph: Users, Pets, Posts, Media, Likes, Comments, and Pet↔Pet Friendships. Requirements:

- Relational integrity across a moderately connected graph.
- "Nearby" discovery by **coarse geohash prefix** (no exact coordinates), which is a plain indexed range scan — not true spatial search.
- Typed data access and low-friction migrations for a small team optimising velocity.
- Bilingual (PL/EN) UI, but the **database should stay language-agnostic**.

## Decision

1. **PostgreSQL** as the database engine. Fits the relational graph, gives btree-indexed geohash-prefix queries for "nearby", JSON where needed, and a clean upgrade path to PostGIS if precise geo ever lands (Phase 2). No PostGIS in MVP.
2. **Prisma** as the ORM. Chosen for its typed client, schema-as-source-of-truth, and migration workflow (`prisma migrate dev` / `deploy`, migrations committed). Wrapped as a NestJS provider.
3. **UUIDv7 primary keys** everywhere — non-enumerable on public resources, yet time-sortable so composite indexes stay append-friendly (avoids UUIDv4 index fragmentation).
4. **No PostgreSQL native enums.** Closed sets are modelled as:
   - **Code-only lookup tables** for user-facing / extensible sets (`Species`), giving FK integrity without storing localized names.
   - **TypeScript string unions** validated at the API boundary (+ optional `CHECK`) for internal, stable sets (`FriendshipStatus`, `MediaType`, `AuthProvider`, `Locale`).
5. **Localization stays out of the database.** Reference tables store stable codes; PL/EN display names live in the app i18n layer, keyed by code. Exception reserved for future large, data-sourced sets (e.g. a Cities dataset), where names are data.
6. **Local development** runs Postgres via **Docker Compose** (`postgres:17`), committed to the repo.

## Alternatives considered

- **TypeORM** — NestJS's "official" integration with decorator entities wired into Nest DI, but a weaker reliability/maintenance reputation. Pick this only if decorator-native DI matters more than Prisma's DX.
- **Drizzle** — SQL-first, minimal runtime, excellent types; rejected for MVP as more manual with a younger ecosystem.
- **MikroORM** — strong Nest integration + Unit of Work; smaller community.
- **PostgreSQL native enums** — rejected: painful to evolve (renaming/removing a value rewrites the type; adding has transaction caveats) and cannot carry localization.
- **bigint auto-increment PKs** — rejected: enumerable, leaking resource counts and enabling scraping on public profiles.

## Consequences

- **Positive:** end-to-end type safety, fast schema iteration, reproducible local DB, language-agnostic schema, enum evolution with zero migration pain, non-enumerable public resources.
- **Negative / trade-offs:** Prisma is not decorator-native, so entities are accessed through a Prisma service rather than Nest-DI-managed repositories; Prisma adds some runtime weight. App-level validation (not the DB) enforces the string-union sets, backed by optional `CHECK` constraints.
