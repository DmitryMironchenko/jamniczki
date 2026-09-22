# Spec: Content streams & the Pet Profile post list

Resolves [Feed composition — #4](https://github.com/DmitryMironchenko/jamniczki/issues/4) (map [#1](https://github.com/DmitryMironchenko/jamniczki/issues/1)). Data model in [data-model.md](./data-model.md); the home feed in [discovery.md](./discovery.md); vocabulary in [CONTEXT.md](../../CONTEXT.md).

This is an **information-architecture** decision, not screen design. It fixes *which content streams exist in the MVP* and the *behavioural contract* of the profile post list. How any of it is laid out or rendered (grid vs feed, post-detail interaction, owner controls) is **screen/UI design — out of scope for this planning map**.

## MVP content streams

The MVP has exactly **two** content streams:

1. **Discovery Feed** — the app's home surface: a merged nearby + friends stream of **Updates**. Owned by [discovery.md](./discovery.md) ([#3](https://github.com/DmitryMironchenko/jamniczki/issues/3)).
2. **Pet Profile post list** — the reverse-chronological list of a single Pet's own **Posts** (this spec).

No other streams ship in MVP. A ranked/personalised home feed and a "what happened to my friends since last visit" activity feed are **Phase 2**.

## Pet Profile post list — contract

- **Contents:** the `Post` rows for one Pet (`Post.petId = <pet>`, `deletedAt IS NULL`). Each post carries its `MediaAsset` gallery, caption, and like/comment counts.
- **Ordering:** reverse-chronological by `Post.createdAt` (newest first).
- **Pagination:** cursor-based on `(createdAt, id)`, infinite scroll. No pinning, no ranking.
- **Index:** backed by the existing `Post (petId, createdAt DESC) WHERE deletedAt IS NULL` (data-model.md) — no new tables or indexes.
- **Visibility:** **public-read** — logged-out viewers see the full list and like/comment **counts**. Any write action (Like, Comment) prompts login, consistent with the Discovery Feed.
- **Link-in:** the Discovery Feed's `NEW_POST` Updates deep-link to the corresponding post within its Profile post list.

## Out of scope (screen/UI design — not this map)

Grid-vs-feed presentation, profile screen layout, the post-detail view/interaction, owner edit/delete affordances, and the profile's **friends list** rendering. These are UI concerns for a separate design track, not load-bearing specs or architecture decisions.
