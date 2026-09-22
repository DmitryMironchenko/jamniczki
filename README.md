# jamniczki

A social network for dog owners.

## Monorepo

pnpm workspace with two apps:

- `apps/backend` — NestJS API
- `apps/frontend` — React + Vite

## Getting started

```bash
pnpm install
pnpm dev          # runs all apps in parallel
```

Or run a single app:

```bash
pnpm --filter @jamniczki/backend dev    # http://localhost:3000
pnpm --filter @jamniczki/frontend dev   # http://localhost:5173
```

## Scripts (root)

| Command             | Description                   |
| ------------------- | ----------------------------- |
| `pnpm dev`          | Run all apps in watch mode    |
| `pnpm build`        | Build all apps                |
| `pnpm lint`         | Lint all apps                 |
| `pnpm typecheck`    | Type-check all apps           |
| `pnpm format`       | Format the repo with Prettier |
| `pnpm format:check` | Verify formatting             |
