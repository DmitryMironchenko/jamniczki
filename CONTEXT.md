# Jamniczki — Domain Glossary (CONTEXT.md)

The canonical vocabulary for this project. Use these terms exactly in code, issues, and specs. This file is a **glossary only** — no implementation details (those live in `docs/adr/` and `docs/specs/`).

## Core terms

- **User** — a human account holder. Signs in (email+password or Google). Owns zero or more Pets. **One User : many Pets.**
- **Pet** — an animal owned by exactly one User. Has a `species` (DOG or CAT), an optional `breed`, a name, an optional bio, and its **own location**. A Pet is the unit of discovery and social identity.
- **Pet Profile** — the public presentation of a Pet (its posts, info, friends). In MVP this is the public projection of the Pet itself, not a separate entity.
- **Post** — an image (gallery of one or more) plus an optional caption, published on a Pet Profile. The unit that carries Likes and Comments.
- **MediaAsset** — one image (later: video) belonging to a Post; stored in object storage and referenced by key.
- **Like** — a User's single endorsement of a Post. At most one per (User, Post).
- **Comment** — a User's text reply on a Post.
- **Friendship** — a social edge between **two Pets** (Pet ↔ Pet), requested by one owner and accepted by the other. A pure social signal in MVP (grants no gated access). Statuses: `PENDING`, `ACCEPTED`, `DECLINED`.
- **Discovery Feed** — the app's home surface: a reverse-chronological stream of **Updates** answering "what's happening near me", merging a geohash-**nearby** source with (when logged-in) a **friends** source. Distinct from a Pet Profile's own post list, which is Feed composition's concern.
- **Update** — one card in the Discovery Feed: a recent *happening*, not a static entry. **Synthetic** (computed on read, never stored). MVP kinds: `NEW_PET_NEARBY` (a new pet appeared in the area) and `NEW_POST` (a pet published a post).
- **Species** — the kind of animal. MVP defines exactly two: **DOG** and **CAT**. There is no "other" bucket; a new species would be added deliberately.
- **Breed** — a named breed belonging to one Species (e.g. *dachshund* is a DOG breed). Optional on a Pet (mixed/unknown breeds exist). Dog breeds and cat breeds are disjoint, split by species.
- **Location** — where a **Pet** lives, stored coarsely as a **geohash** (~1.2 km cell) plus city/region/country. Never exact coordinates. Drives "nearby" discovery. A User may keep an optional **home area** used only to pre-fill a new Pet's location.
- **geohash** — a short string encoding an approximate area; "nearby" is a shared-prefix match. The finer the geohash, the smaller the area.

## Language

- The UI is **bilingual (PL + EN)**; translations live in the app i18n layer, not the database.
- **User-generated content** (captions, comments, bios) is free text in whatever language the User writes; it is not translated. It may carry an optional `locale` tag.
- **Reference data** (Species, Breed) is stored as language-agnostic **codes**; their display names are localized in the app i18n layer, keyed by code.
