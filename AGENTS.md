# Repository Guidelines

Astro 6 SSR app (React 19 islands, Tailwind 4, shadcn/ui, Supabase Auth) deployed to Cloudflare Workers. See `@README.md` (setup) and `@CLAUDE.md` (architecture).

## Hard Rules

- Every endpoint in `src/pages/api/` must export `const prerender = false` — prerendering breaks Cloudflare runtime.
- Read server secrets via `astro:env/server` (schema in `@astro.config.mjs`), never `import.meta.env` or `process.env`.
- Never concatenate Tailwind class strings — use `cn()` from `@/lib/utils` (clsx + tailwind-merge).
- New React forms use `react-hook-form` + `zod` resolver — no bare `useState` validators (existing `src/components/auth/*` predates this).
- New Supabase tables must enable RLS with granular per-operation, per-role policies.
- MIAPI calls go server-side via `src/lib/services/miapi.ts` — only `POST /v1/answer` with `citations: true` and `include_sources: true`; zod-validate the response (docs: https://miapi.uk/#docs).
- No Next.js directives (`"use client"`, etc.) — Astro, not Next.

## Project Structure & Module Organization

- `src/pages/` — Astro routes; `src/pages/api/` endpoints export uppercase `GET`/`POST` and zod-validate input.
- `src/components/` — `*.astro` for static/layout, React `*.tsx` only when interactive. shadcn primitives in `src/components/ui/` (new-york variant; add via `npx shadcn@latest add <name>`); hooks in `src/components/hooks/`.
- `src/lib/` — services/helpers (`supabase.ts`, `utils.ts`); business logic in `src/lib/services/`. Path alias `@/*` → `./src/*`.
- `src/middleware.ts` — auth gate; add protected paths to `PROTECTED_ROUTES`.
- Shared entity/DTO types in `src/types.ts`. Supabase migrations: `supabase/migrations/YYYYMMDDHHmmss_short_description.sql`.

## Build, Test, and Development Commands

- `npm run dev` — dev server on Cloudflare workerd.
- `npm run build` — production SSR build for Cloudflare.
- `npm run lint` / `npm run lint:fix` — ESLint, type-checked rules (`@eslint.config.js`).
- `npm run format` — Prettier (Astro + Tailwind plugins).
- `npx astro sync` — regenerate `.astro/types.d.ts`; run before lint.
- `npx supabase start` — local Supabase stack (Docker, ~7 GB RAM).
- `npx wrangler deploy` — Cloudflare Workers deploy.

## Coding Style & Naming Conventions

Two-space indent, double quotes, semicolons, trailing commas, 120-col print width (`@.prettierrc.json`). TypeScript strict via `@tsconfig.json`. Pre-commit (husky + lint-staged) auto-fixes `*.{ts,tsx,astro}` and formats `*.{json,css,md}` — don't bypass.

## Commit & Pull Request Guidelines

Branch off `main` as `feature/<slug>` and merge via PR. Commit subjects: short, sentence-case, imperative (e.g. *"tech stack bootstrap"*); no Conventional Commits prefix. CI (`@.github/workflows/ci.yml`) runs `astro sync`, `lint`, and `build` on push/PR to `master` — all must pass. `SUPABASE_URL`, `SUPABASE_KEY`, and `MIAPI_API_KEY` are required GitHub secrets.

## Security & Configuration

Local env: `.env` (Node) or `.dev.vars` (Cloudflare, gitignored); production secrets via `npx wrangler secret put`. Node pinned in `@.nvmrc` (v22.14.0).
