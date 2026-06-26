# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- `bun run dev` — start dev server on port 3000 (loads `.env.local`, instruments Sentry). Use this, not bare `vite dev`, so env vars and Sentry are wired up.
- `bun run test` — run Vitest once. Single file: `bun run test src/foo.test.ts`. Single test: `bun run test -t "name"`.
- `bun run check` — Biome lint + format (the combined gate). `bun run lint` / `bun run format` run them individually.
- `bun run generate-routes` — regenerate `src/routeTree.gen.ts` from `src/routes/` via `tsr generate` (also runs automatically during dev/build).
- `bun run build` — Vite build (also copies the Sentry instrument file into `.output/server`).
- `bun run deploy` — build then `wrangler deploy` to Cloudflare Workers.
- `bunx convex dev` — run the Convex dev server; required alongside `bun run dev` for any Convex queries/mutations to work.

## Architecture

**TanStack Start** (full-stack React 19, SSR) deployed to **Cloudflare Workers**. The Worker entry is `@tanstack/react-start/server-entry` (set in `wrangler.jsonc`); the Cloudflare Vite plugin runs the SSR environment.

**Routing is file-based** under `src/routes/`. `src/routeTree.gen.ts` is generated — never edit it by hand. `src/routes/__root.tsx` is the app shell (HTML document, providers, devtools). API routes use the `server.handlers` pattern (see `src/routes/api/auth/$.ts`).

**Data layer is split between two systems:**
- **Convex** (`convex/`) is the application database. `convex/schema.ts` defines tables; queries/mutations live in files like `convex/todos.ts` and are exposed through generated `convex/_generated/api`. The client is provided app-wide via `src/integrations/convex/provider.tsx` (needs `VITE_CONVEX_URL`). `@convex-dev/react-query` bridges Convex into TanStack Query.
- **TanStack Query** is the client cache. The `QueryClient` is created in `src/integrations/tanstack-query/root-provider.tsx#getContext`, passed as router context in `src/router.tsx`, and SSR-integrated via `setupRouterSsrQueryIntegration`.

**Auth is Better Auth** (`src/lib/auth.ts`), mounted at `/api/auth/$` as a catch-all handler. Server config uses `tanstackStartCookies()`; the client (`src/lib/auth-client.ts`) exposes `authClient` (e.g. `authClient.useSession()`, `authClient.signOut()`).

**Sentry** is initialized in `instrument.server.mjs`, loaded via `NODE_OPTIONS --import` in dev and copied into the build output. It no-ops without `VITE_SENTRY_DSN`.

## Conventions

- **Import alias:** `#/*` and `@/*` both map to `src/*` (e.g. `import { auth } from "#/lib/auth"`).
- **Formatting (Biome):** tabs for indentation, double quotes, organize-imports on. `src/routeTree.gen.ts` and `src/styles.css` are excluded from Biome.
- **TypeScript is strict**, including `noUnusedLocals`/`noUnusedParameters` — unused symbols fail the build.
- **UI:** Tailwind CSS v4 (config-less, via `@tailwindcss/vite`; styles in `src/styles.css`) and shadcn (new-york style, zinc base). Add components with `pnpm dlx shadcn@latest add <name>`; they land in `src/components/ui`. Icons from `lucide-react`.
- Files prefixed `demo` are scaffolding and safe to delete.
- Secrets go to Cloudflare via `wrangler secret put`; non-secret vars go in `wrangler.jsonc` under `vars`. Local env lives in `.env.local`.
