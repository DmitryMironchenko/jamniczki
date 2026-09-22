# ADR-0002: Compute the Discovery Feed on read (no materialised feed table)

- **Status:** Accepted
- **Date:** 2026-09-15
- **Deciders:** Dmitry Mironchenko
- **Context ticket:** [Discovery / "nearby feed" mechanics — #3](https://github.com/DmitryMironchenko/jamniczki/issues/3) (map [#1](https://github.com/DmitryMironchenko/jamniczki/issues/1))

## Context

The [Discovery Feed](../specs/discovery.md) is a reverse-chronological stream of **Updates** (`NEW_PET_NEARBY`, `NEW_POST`) merged from a geohash-**nearby** source and a **friends** source, over a rolling 14-day window. Every MVP Update kind is fully derivable from rows that already exist: `NEW_PET_NEARBY` from `Pet.createdAt` + `Pet.geohash`, `NEW_POST` from `Post.createdAt` (+ its pet's geohash/friend edges). There is no per-viewer read/seen state and no ranking in MVP. The obvious alternative — writing an activity/event row whenever something happens — would introduce a new write path and a table to keep consistent.

## Decision

Build the feed by **querying existing tables on each read** and unioning the results; **add no feed/activity/event table** for MVP.

- `NEW_PET_NEARBY` = `Pet` rows within the window whose `geohash` matches the viewer's nearby cell-set (see spec's [Nearby query](../specs/discovery.md#nearby-query)).
- `NEW_POST` = `Post` rows within the window whose pet is nearby **or** an accepted friend of one of the viewer's pets.
- Union, filter to the 14-day window, sort by `occurredAt DESC`, cursor-paginate on `(occurredAt, kind, sourceId)`.

This leans on indexes already specified in [data-model.md](../specs/data-model.md): `Pet (speciesCode, geohash) WHERE deletedAt IS NULL AND geohash IS NOT NULL`, `Post (petId, createdAt DESC)`, and the `Friendship` accepted-edge indexes. **No new tables** — consistent with data-model.md's rule that the Discovery/Feed sub-wayfinders must not contradict the core schema.

## Alternatives considered

- **Materialised feed table** (fan-out-on-write / activity log) — a row per happening, feed = a single indexed read. Rejected for MVP: adds a write path and a consistency surface for only two derivable event types at modest scale, and would bake in choices (fan-out strategy, retention) better made under real load. It stays the natural Phase-2 move.
- **Fan-out-on-write per follower** — rejected outright: friendship is a small mutual graph, not a follower firehose; the read-time union is cheap here.

## Consequences

- **Positive:** no new tables, no dual-write/backfill, no feed-vs-source drift; new Update kinds derivable from existing rows are trivial to add; deletes/soft-deletes reflect instantly (the source row simply stops matching).
- **Negative / trade-offs:** each feed request runs a union query rather than one indexed read; the friends-anywhere source is not geohash-bounded, so its cost scales with a viewer's accepted-friend count. **Phase-2 trigger:** revisit materialisation when feed-query latency or friend-graph fan-out becomes a hot path, or when a non-derivable Update kind (or per-viewer seen-state) is introduced.
