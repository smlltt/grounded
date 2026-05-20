# Frontend Conventions — `src/`

Scope: rules for writing `.astro` pages/components and React islands under `src/`. Inherits everything in `@AGENTS.md` (project root). This file only adds rules specific to the UI layer.

## Hard rules (frontend)

- Default to `.astro` components. Reach for `.tsx` only when the markup needs interactivity (state, event handlers, effects, focus management). Static layout and data-only rendering must stay in `.astro`.
- Server data is fetched in `.astro` frontmatter (top-level `await` is fine) and passed to islands as serializable props. Never call `astro:db`, `astro:content`, Supabase server clients, or anything reading `Astro.locals` from inside a `.tsx` file — those imports leak server code into the client bundle.
- Do not default to `client:load`. Pick the cheapest directive that still works (see "Client directives" below). Every `client:*` ships its component's JS, so an island without a `client:*` directive is intentional (server-rendered, zero JS) — leave it that way.
- Shared state across two or more islands uses [Nano Stores](https://github.com/nanostores/nanostores) (`nanostores` + `@nanostores/react`). Do not introduce Redux, Zustand, Jotai, or React Context across islands — Context does not cross island boundaries.
- Do not lift `useState` into a shared store unless a second island (or an `.astro` `<script>`) reads or writes the same value. Single-island state stays local.
- Tailwind class composition goes through `cn()` (see root `@AGENTS.md`). Conditional classes never use template literals or string concatenation.
- Every interactive control rendered in `.astro` uses semantic HTML (`<button type="...">`, `<a href>`, `<form>`, `<nav>`, `<main>`). Do not put `onclick` on a `<div>`.
- Forms follow the root rule (react-hook-form + zod). The zod schema lives next to the component as `<form>.schema.ts` and is also imported by the matching `src/pages/api/*` handler so client and server validate against the same shape.

## Choosing `.astro` vs React island

- Pure layout, server-rendered data, links, static badges → `.astro`. See `src/components/Topbar.astro`, `src/components/Banner.astro`, `src/pages/dashboard.astro`.
- Anything needing `useState` / `useEffect` / refs / portals → `.tsx` island. See `src/components/auth/SignUpForm.tsx`.
- Mixed page: keep the page `.astro`, pass data as props to a small `.tsx` island, render everything else as `.astro` children around it.
- Never wrap a static React tree in `client:load` just to colocate JSX — convert it to `.astro` instead.

## Client directives (which to pick)

Order of preference, cheapest first:

- No directive — component is server-rendered HTML only. Default.
- `client:visible` — heavy or below-the-fold interactivity (long lists, charts, the question history feed when it lands). Hydrates on scroll into view.
- `client:idle` — non-critical above-the-fold interactivity (menu toggles, dismissible banners).
- `client:load` — must be interactive before first paint (the ask-question form on the main page, the sign-in form). This is the directive auth forms in `src/components/auth/*` should use.
- `client:only="react"` — only when SSR would mismatch (browser-only APIs like `window`, `localStorage`, or UI whose first paint depends on authenticated session state not available at SSR time). Adds CLS risk; prefer SSR + `client:load` when possible.

Rule of thumb: if you reach for `client:only`, leave a one-line comment in the `.astro` consumer explaining why SSR was skipped.

## Sharing state across islands (Nano Stores)

When you actually need it (≥2 islands read/write the same value, or a `.astro` `<script>` needs to push to a React island):

- Install on first use: `npm install nanostores @nanostores/react`.
- Stores live in `src/lib/stores/<feature>.store.ts`, exporting `atom()` / `map()` / `computed()` from `nanostores`. One file per feature, not a global `stores.ts`.
- React reads via `useStore()` from `@nanostores/react`. Inside event handlers (not rendering), call `.get()` directly — `useStore` triggers a re-render on every change, `.get()` doesn't.
- Use `map<T>()` (not `atom<T>()`) when the value is an object with independently-changing keys — `map` re-renders only subscribers of the changed key.
- `computed()` for derived values; never compute derived state inside a React `useEffect`.
- Never store secrets, raw Supabase sessions, or anything you wouldn't put in `window` — Nano Stores live in client memory.
- Cross-framework `.astro` `<script>` consumers import the same store file and use `.subscribe()` / `.set()`.

## React island performance

When writing `.tsx`, apply the React-island subset of `@/Users/samuelliotta/.agents/skills/vercel-react-best-practices/SKILL.md`: `rerender-derived-state-no-effect`, `rerender-functional-setstate`, `rerender-lazy-state-init`, `rerender-use-ref-transient-values`, `rerender-dependencies`, `rerender-move-effect-to-event`, `rendering-conditional-render`, `client-passive-event-listeners`, `bundle-barrel-imports`. Skip the `server-*`, `async-api-routes`, and `next/dynamic` rules — they target Next.js, not Astro islands.

## A11y baseline

- `@eslint.config.js` owns the baseline JSX accessibility checks; do not duplicate lint-enforced rules here.
- Color contrast: never communicate state with color alone (the `FormField` error path already pairs color with the `CircleAlert` icon — follow that pattern).
- shadcn primitives in `src/components/ui/` already ship ARIA — preserve all props when wrapping them; do not strip `aria-*` or `data-*`.

## Naming & layout (frontend-specific)

- `.astro` components: `PascalCase.astro`. React islands: `PascalCase.tsx`, default export when the file is one component.
- Custom hooks: `src/components/hooks/use<Thing>.ts` (already in root `@AGENTS.md`).
- Stores: `src/lib/stores/<feature>.store.ts`.
- Zod schemas for forms: `src/components/<area>/<form>.schema.ts`, imported by both the form island and its `src/pages/api/<area>/<form>.ts` handler.
- Shared DTOs / entity types stay in `src/types.ts` (per root `@AGENTS.md`). Only create a per-feature `types.ts` when all exported types are consumed exclusively inside that feature directory; move any cross-feature or API DTO back to `src/types.ts`.
