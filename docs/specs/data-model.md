# Spec: MVP Core Data Model

Resolves [Persistence & data model — #2](https://github.com/DmitryMironchenko/jamniczki/issues/2). Engine & ORM rationale in [ADR-0001](../adr/0001-database-and-orm.md); vocabulary in [CONTEXT.md](../../CONTEXT.md).

This is the **core** MVP schema. The Discovery ([#3](https://github.com/DmitryMironchenko/jamniczki/issues/3)) and Feed ([#4](https://github.com/DmitryMironchenko/jamniczki/issues/4)) sub-wayfinders may append tables (e.g. swipe history, feed tables); they should not contradict this core.

## Conventions

- **PK:** `id UUIDv7` on every entity table unless noted.
- **Timestamps:** `createdAt`, `updatedAt` on every table.
- **Soft-delete:** `deletedAt` (nullable) on User, Pet, Post, Comment. Hot indexes on these are partial: `WHERE deletedAt IS NULL`.
- **Code sets in code:** `FriendshipStatus = 'PENDING'|'ACCEPTED'|'DECLINED'`, `MediaType = 'IMAGE'|'VIDEO'` (MVP: IMAGE only), `AuthProvider = 'GOOGLE'`, `Locale = 'pl'|'en'`. Validated at the API boundary; optional `CHECK` constraints.

## Entities

### User
| field | type | notes |
|---|---|---|
| id | UUIDv7 PK | |
| email | citext | **UNIQUE** |
| passwordHash | text? | null for OAuth-only accounts |
| displayName | text | |
| locale | text | `'pl'`/`'en'` — preferred UI language |
| homeGeohash | char(6)? | optional; only pre-fills a new Pet's location |
| homeCity | text? | |
| createdAt / updatedAt / deletedAt | timestamptz | deletedAt nullable |

### AuthIdentity (Google OAuth)
| field | type | notes |
|---|---|---|
| id | UUIDv7 PK | |
| userId | FK → User | |
| provider | text | `'GOOGLE'` |
| providerAccountId | text | |
| — | | **UNIQUE (provider, providerAccountId)** |

### Species (code-only lookup)
| field | type | notes |
|---|---|---|
| code | text PK | seeded: `DOG`, `CAT` (no OTHER) |

### Breed (reference; names localized in app i18n by `code`)
| field | type | notes |
|---|---|---|
| id | UUIDv7 PK | |
| code | text | **UNIQUE**; i18n key (e.g. `DACHSHUND`) |
| speciesCode | FK → Species | |
| isActive | bool | |
| — | | **UNIQUE (id, speciesCode)** to back Pet's composite FK guard |

### Pet
| field | type | notes |
|---|---|---|
| id | UUIDv7 PK | |
| ownerId | FK → User | |
| name | text | |
| speciesCode | FK → Species | **required** |
| breedId | FK → Breed | **nullable** (mixed/unknown) |
| bio | text? | free text, user's language |
| geohash | char(6)? | **pet's** location; coarse (~1.2 km); null ⇒ not discoverable |
| city / region / countryCode | text? | |
| createdAt / updatedAt / deletedAt | timestamptz | |

Optional DB-level guard: composite FK `(breedId, speciesCode) → Breed(id, speciesCode)` so a Pet's breed must match its species.

### Post
| field | type | notes |
|---|---|---|
| id | UUIDv7 PK | |
| petId | FK → Pet | the profile it's on |
| authorUserId | FK → User | who posted (audit for future co-management) |
| caption | text? | |
| locale | text? | optional content-language tag |
| createdAt / updatedAt / deletedAt | timestamptz | |

### MediaAsset
| field | type | notes |
|---|---|---|
| id | UUIDv7 PK | |
| postId | FK → Post | |
| type | text | `MediaType`; MVP IMAGE only |
| storageKey | text | object-storage key (not a URL); read via signed URL |
| mimeType / width / height / sizeBytes | | |
| position | int | gallery ordering |

### Like
| field | type | notes |
|---|---|---|
| id | UUIDv7 PK | |
| postId | FK → Post | |
| userId | FK → User | |
| — | | **UNIQUE (postId, userId)** |

### Comment
| field | type | notes |
|---|---|---|
| id | UUIDv7 PK | |
| postId | FK → Post | |
| authorUserId | FK → User | |
| body | text | |
| createdAt / updatedAt / deletedAt | timestamptz | |

### Friendship (Pet ↔ Pet)
| field | type | notes |
|---|---|---|
| id | UUIDv7 PK | |
| petAId / petBId | FK → Pet | **canonical order** (petA.id < petB.id) so a pair is unique regardless of direction |
| requestedByPetId | FK → Pet | who initiated |
| status | text | `PENDING`/`ACCEPTED`/`DECLINED` |
| requestedAt | timestamptz | |
| respondedAt | timestamptz? | set on accept/decline |
| — | | **UNIQUE (petAId, petBId)** |

**MVP has no friendship deletion** — rows are retained and status-driven. Cancel-request and unfriend are Phase 2.

## Indexes (hot paths)

- **Discovery** — `Pet (speciesCode, geohash) WHERE deletedAt IS NULL AND geohash IS NOT NULL`. "Dogs/cats near here" = single-table index range scan, **no join, no denormalization** (location is native to Pet).
- **My pets** — `Pet (ownerId) WHERE deletedAt IS NULL`.
- **Profile feed** — `Post (petId, createdAt DESC) WHERE deletedAt IS NULL`.
- **Comments** — `Comment (postId, createdAt) WHERE deletedAt IS NULL`.
- **Likes** — UNIQUE `(postId, userId)` (also the count index).
- **Friend lists** — `Friendship (petAId) WHERE status='ACCEPTED'` and `(petBId) WHERE status='ACCEPTED'`.
- **Auth** — User UNIQUE `(email)`; AuthIdentity UNIQUE `(provider, providerAccountId)`.
- **Breed pickers** — `Breed (speciesCode)`; UNIQUE `(code)`.
