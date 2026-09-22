# Spec: Discovery Feed

Resolves [Discovery / "nearby feed" mechanics — #3](https://github.com/DmitryMironchenko/jamniczki/issues/3) (map [#1](https://github.com/DmitryMironchenko/jamniczki/issues/1)). Architecture rationale in [ADR-0002](../adr/0002-compute-on-read-discovery-feed.md); data model in [data-model.md](./data-model.md); vocabulary in [CONTEXT.md](../../CONTEXT.md).

The **Discovery Feed** is the app's home surface: a visual, reverse-chronological stream of recent **Updates** — "what's happening near me" — closer to Instagram Stories than to a pet directory or a Tinder swipe deck. It is about surfacing **new/nearby pets and their fresh posts**; the post list *on a single Pet Profile* belongs to Feed composition ([#4](https://github.com/DmitryMironchenko/jamniczki/issues/4)).

## Scope

**In:** the Discovery Feed surface — its Updates, ordering, the geohash "nearby" query, freshness, empty-state, and how social actions attach to a card.

**Out:** per-Pet-Profile post lists and friends-activity *as a distinct screen* (→ #4); mutual-match / swipe-gating / chat (Phase 2); ranking/personalisation (Phase 2); per-viewer read/seen state (Phase 2).

## The Update

An **Update** is one card in the feed — a *recent happening*, not a static directory entry. It is **synthetic**: computed on read from existing rows, never stored (see [ADR-0002](../adr/0002-compute-on-read-discovery-feed.md)). Two kinds in MVP:

| kind | source row | `occurredAt` | card shows |
|---|---|---|---|
| `NEW_PET_NEARBY` | `Pet` | `Pet.createdAt` | pet avatar/best photo, name, breed, coarse distance band, "new nearby" badge |
| `NEW_POST` | `Post` (+ its `MediaAsset`s) | `Post.createdAt` | first image, caption, pet name, coarse distance band |

An Update's identity for pagination/dedup is `(kind, sourceId)`. A single pet may legitimately produce **both** a `NEW_PET_NEARBY` and one or more `NEW_POST` Updates within the window — they are distinct happenings and both appear.

## Audience & sources

The feed merges up to two **sources**; which apply depends on the viewer:

- **Nearby source** — Updates whose pet is within the viewer's nearby set (see [Nearby query](#nearby-query)).
- **Friends source** *(logged-in only)* — Updates from any Pet that has an **ACCEPTED** `Friendship` with any Pet the viewer owns, **regardless of location**.

| viewer | sources | origin of "nearby" |
|---|---|---|
| Logged-out, no location | global latest-N (fallback path) | — |
| Logged-out, area chosen | nearby | the chosen area's geohash |
| Logged-in | nearby **+** friends (merged) | union of all the viewer's pets' geohashes |

A logged-in viewer who owns **no pets** (or whose pets have `geohash = null`) has an empty nearby origin and falls to the [empty-state](#empty--cold-start) path; their friends source is likewise empty. Location is **coarse geohash only** — never exact coordinates, in the query or on the card.

### Viewer location

- **Logged-in:** the nearby origin is the **union of all owned pets' `geohash`** values (`char(6)`, `deletedAt IS NULL`, `geohash IS NOT NULL`). Multiple pets in different places simply mix their areas together. `User.homeGeohash` is **not** a feed origin — it only pre-fills a new pet's location. A manual area override is allowed.
- **Logged-out:** a manual **area picker** (city/area → geohash). Browser geolocation may be *offered* but is never required, and there is **no silent IP geolocation**.

## Ordering

A single **merged stream, pure reverse-chronological** by `occurredAt` (newest first). Nearby and friends Updates interleave purely by time — **no ranking, no weighting, no personalisation** in MVP. Logged-out is the same stream with the friends source absent. Ties broken by `(occurredAt DESC, kind, sourceId)` for a stable cursor.

## Nearby query

Location lives on the **Pet** as `geohash char(6)` (~1.2 km cell); the discovery index `Pet (speciesCode, geohash) WHERE deletedAt IS NULL AND geohash IS NOT NULL` already backs this (data-model.md). "Nearby" is a **shared-prefix / cell-set match**, not spatial search:

1. **Tight ring (length 6, ~1.2 km):** for each origin geohash, build the 9-cell set = the cell itself **+ its 8 neighbours** (standard geohash-neighbour computation, to avoid cell-edge blind spots). Match `Pet.geohash = ANY(<cells>)`.
2. **Widen on sparsity:** if the candidate Update count is below the target (**~20**), **truncate the prefix** by one char and match `Pet.geohash LIKE '<prefix>%'` — ~1.2 km → ~5 km (len 5) → ~39 km (len 4). Each Update is tagged with the **distance band** it was found at (used for the card label, not for ordering).
3. **Stop** at the first level meeting the target **or** at the **max radius (~40 km / prefix length 4)**, whichever comes first. Still short → [empty-state](#empty--cold-start).

When the viewer has several origin cells (multi-pet), union their cell-sets and de-duplicate pets by id. All numbers (target ~20, max ~40 km) are **config, not contract**.

## Freshness

- The feed is a **rolling 14-day window**: only Updates with `occurredAt >= now() - 14 days`.
- A pet is shown as **"new nearby"** (i.e. a `NEW_PET_NEARBY` Update is emitted) for its **first 7 days** after `Pet.createdAt`. After 7 days the pet still appears via its `NEW_POST` Updates, just without the "new" framing.
- **Pagination:** cursor-based within the window, cursor = `(occurredAt, kind, sourceId)` of the last item; infinite scroll. Soft-deleted rows (`deletedAt`) are always excluded.
- Both windows (14 days, 7 days) are **config, not contract**.

## Social actions on a card

| action | from the card | notes |
|---|---|---|
| **Like** | inline (one tap) | toggles a `Like` on the Update's Post; only on `NEW_POST` cards |
| **Comment** | taps through to the Post | full context; not composed inline |
| **Befriend** | **not** on the card — only from the Pet Profile | it's a Pet↔Pet request needing fuller context |

Cards render like/comment **counts** for everyone. For **logged-out** viewers any action (Like, Comment, Befriend) **prompts login** rather than proceeding.

## Empty / cold-start

The feed is **never shown empty**. When the nearby query (after widening to max radius) plus the friends source yield too few items — including the logged-out-no-location case and the no-pets logged-in case — fall back to the **global latest-N** Updates (most recent across the whole community, same 14-day window). Layer a gentle **"add your pet" / "set your area"** prompt on top. The global-latest path is the single shared fallback for every thin-origin case.

## Architecture

**Compute-on-read** — no feed/event/activity table is added for MVP. Each request builds the Update stream by querying `Pet` (for `NEW_PET_NEARBY`) and `Post` (for `NEW_POST`) with the window + nearby/friends predicates, unioning and sorting by `occurredAt`. Rationale, trade-offs, and the Phase-2 materialisation trigger are in [ADR-0002](../adr/0002-compute-on-read-discovery-feed.md). This adds **no new tables** to the core data model — consistent with data-model.md's note that sub-wayfinders "should not contradict this core."

## Phase-2 fog (deferred)

Ranked/personalised ordering · per-viewer read/seen state · additional Update kinds (new friendship, milestones, events) · a materialised feed/activity table · swipe history · silent IP geolocation.
