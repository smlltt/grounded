# Cloudflare Workers — Deployment Plan

Wire Grounded for production deploy on Cloudflare Workers with auto-deploy on push to `main` via GitHub Actions (`cloudflare/wrangler-action@v3`). Provisions a cloud Supabase project, fixes repo hygiene (MIAPI key typo, worker name, env schema), then runs a manual smoke deploy before turning on the push-to-`main` deploy workflow and verifying rollback.

## Constraints from [`context/foundation/infrastructure.md`](../../foundation/infrastructure.md) and [`AGENTS.md`](../../../AGENTS.md)

- Target is **Cloudflare Workers** (not Pages). Adapter `@astrojs/cloudflare@^13.5.0`, runtime workerd, `output: "server"` — already in place.
- Server secrets are read via `astro:env/server` (schema in [`astro.config.mjs`](../../../astro.config.mjs)), never `import.meta.env` or `process.env`.
- MIAPI calls are server-side only via `src/lib/services/miapi.ts`; under-5s response is an NFR gate.
- Local Supabase (`http://127.0.0.1:54321`) is not reachable from production Workers — a hosted Supabase project is required.

## Deploy topology (decided)

- Auto-deploy on push to `main` is driven by [`.github/workflows/deploy.yml`](../../../.github/workflows/deploy.yml), which checks out, builds, then runs `cloudflare/wrangler-action@v3`.
- [`.github/workflows/ci.yml`](../../../.github/workflows/ci.yml) stays as a **PR-time lint+build guardrail only**, no deploy step.
- Manual `npx wrangler deploy` from a laptop remains supported.
- Production URL: `grounded.<account>.workers.dev` (no custom domain in this plan).
- No PR preview deploys (per your choice).

```mermaid
flowchart LR
  Dev[Developer merges PR to main] --> GH[GitHub main]
  GH -->|push event| Deploy[deploy.yml GH Actions job]
  Deploy -->|npm ci + npm run build| Built[dist/ + worker entry]
  Built -->|wrangler-action v3 wrangler deploy| Worker[grounded Worker]
  Dev2[Developer laptop] -->|npx wrangler deploy manual| Worker
  Worker -->|HTTPS| Users[Users on workersdotdev]
  Worker -->|@supabase/ssr| Supabase[Cloud Supabase]
  Worker -->|server-side fetch| MIAPI[MIAPI v1/answer]
  GH -->|PR| CI[ci.yml lint+build guardrail]
```

## Pre-existing state (confirmed via read)

- [`wrangler.jsonc`](../../../wrangler.jsonc): `nodejs_compat`, `observability.enabled`, `assets` binding, `compatibility_date: 2026-05-08`. Worker name still `10x-astro-starter` — will rename to `grounded`.
- [`astro.config.mjs`](../../../astro.config.mjs): env schema declares `SUPABASE_URL`, `SUPABASE_KEY` only — `MIAPI_API_KEY` is missing.
- [`.env.example`](../../../.env.example) and [`.dev.vars`](../../../.dev.vars): contain `MIAMI_API_KEY` typo (must be `MIAPI_API_KEY` per [`AGENTS.md`](../../../AGENTS.md) hard rule).
- [`src/lib/supabase.ts`](../../../src/lib/supabase.ts) + [`src/middleware.ts`](../../../src/middleware.ts) already use `@supabase/ssr` server client correctly for the workerd cookie pattern.

## External integrations — risks and where they appear in the plan

- **Supabase Auth on Workers**: chunked-cookie corruption for sessions >4KB was fixed in `@supabase/ssr` PR #210 (May 2026). Current `^0.10.3` should include it — Phase 3 confirms the resolved version and Phase 5 includes a long-session smoke check.
- **MIAPI**: external HTTP from workerd. Subrequest budget is 50/request on the Free plan, 1000 on Paid — for a single `POST /v1/answer` this is fine but the smoke step measures it. The under-5s NFR is wall-clock; Workers free CPU-time cap (10 ms initial, bursts allowed) applies only to JS execution, not the waiting time for MIAPI.
- **GitHub Actions deploy auth**: production deploys authenticate via `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` GitHub repo secrets (Phase 6). Mint a dedicated `grounded-gha-deploy` token scoped to the **Edit Cloudflare Workers** template on a single account — do not reuse your laptop token. Rotate by creating a new token, updating the repo secret, and revoking the old one; GitHub redacts secret values in logs but never commit or echo them elsewhere.
- **Cloudflare account**: a single account/zone — no DNS work because we stay on `*.workers.dev`.

## Progress overview

- [ ] Prerequisites — accounts, local toolchain, CLI auth, network checks
- [x] Phase 0 — Pre-flight audit (read-only)
- [x] Phase 1 — Repo hygiene
- [x] Phase 2 — Provision cloud Supabase project
- [x] Phase 3 — Local smoke against cloud Supabase
- [x] Phase 4 — First manual deploy from laptop
- [ ] Phase 5 — Post-deploy smoke (NFR gates)
- [x] Phase 6 — Set up GitHub Actions deploy workflow
- [ ] Phase 7 — Workflow role separation (`ci.yml` vs `deploy.yml`)
- [ ] Phase 8 — Rollback drill
- [ ] Phase 9 — Docs sync

## Prerequisites

Everything below should be true **before starting Phase 0**. Each checkbox is a one-time setup; if you're returning to this plan after a break, walk the list to make sure nothing has rotated out.

### Accounts (free tier is enough for MVP)

- [ ] **Cloudflare account** — sign up at <https://dash.cloudflare.com/sign-up>. Verify the email; no payment method required for the Workers Free plan.
- [ ] **Supabase account** — sign up at <https://supabase.com/dashboard/sign-up>. Signing in with GitHub is the smoothest path. Free tier limits: 2 free projects, 500 MB DB, 50k MAU.
- [ ] **GitHub access to the `grounded` repo** as Admin — required in Phase 6 to create the `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` repository secrets.
- [ ] **MIAPI API key** — already present in [`.dev.vars`](../../../.dev.vars) as `MIAMI_API_KEY` (typo fixed in Phase 1). If the key has rotated, mint a new one at <https://miapi.uk/>.

### Local toolchain

Versions are pinned by [`.nvmrc`](../../../.nvmrc) and [`package.json`](../../../package.json) `devDependencies`. No global installs are needed — everything runs via `npx`.

- [ ] **Node.js v22.14.0** — exact version in [`.nvmrc`](../../../.nvmrc). With `nvm`: `nvm install && nvm use`. Verify: `node --version` → `v22.14.0`.
- [ ] **npm** — bundled with Node 22. Verify: `npm --version`.
- [ ] **Git** — verify: `git --version`.
- [ ] `npm ci` completed in repo root — confirms `wrangler@^4.90.0` and `supabase@^2.23.4` resolve under `node_modules/.bin`. Verify: `npx wrangler --version` prints `4.x.x`.
- [ ] **(Optional) Docker** — only needed if you ever want local Supabase via `npx supabase start` (~7 GB RAM). Not required for this plan; we move to cloud Supabase in Phase 2.

### Repository state

- [ ] Repo is pushed to GitHub at `<org-or-user>/grounded`. The **default branch is `main`** (already the case in this repo) — Phase 6 wires `deploy.yml` to `main`.
- [ ] You can push to the default branch (or merge into it) — the deploy workflow triggers on **any push to `main`**, not on PR merge specifically.
- [ ] [`.gitignore`](../../../.gitignore) excludes `.env` and `.dev.vars` — confirm this stays the case so secrets never leak into commits.

### Wrangler CLI authentication

Wrangler ships as a dev dependency; do **not** install it globally. All commands use `npx wrangler …`.

- [ ] **Authenticate**: `npx wrangler login` — opens your browser to OAuth into Cloudflare. Wrangler starts a local callback server on `http://localhost:8976`.
- [ ] **Verify**: `npx wrangler whoami` — prints email + Account ID. If multiple accounts, the active one is shown; switch with `npx wrangler logout && npx wrangler login`.
- [ ] Credentials persist at `~/.config/.wrangler/config/default.toml` (macOS/Linux) or `%APPDATA%\.wrangler\` (Windows). They survive shell sessions but not machines — re-run `wrangler login` if you switch laptops.

**Laptop API token** (headless / corporate VPN — loopback blocked, no browser):

1. Create an API token at <https://dash.cloudflare.com/profile/api-tokens> using the **Edit Cloudflare Workers** template (or a custom token with `Account → Workers Scripts: Edit`, `Account → Account Settings: Read`, `Zone → Workers Routes: Edit`).
2. Export it before any `wrangler` command: `export CLOUDFLARE_API_TOKEN=...`
3. Wrangler picks the env var up automatically and skips OAuth. This token is for your laptop only — Phase 6 mints a **separate** `grounded-gha-deploy` token for GitHub Actions. The two are independent; rotating one does not affect the other.

### Supabase CLI (optional for this plan)

The `supabase` CLI is installed as a dev dependency. Phase 2 happens entirely in the dashboard, so you only need the CLI if you want to manage migrations or run local Supabase later.

- [ ] **(Optional) Authenticate**: `npx supabase login` — opens browser, generates an access token stored at `~/.supabase/access-token`.
- [ ] **(Optional) Link the local repo to the cloud project** (after Phase 2 creates it): `npx supabase link --project-ref <ref>` where `<ref>` is the slug in the project URL.

### Cloudflare account sanity checks (60-second precheck)

- [ ] **Workers plan**: dashboard → Workers & Pages → Plans confirms **Workers Free** (100k requests/day, 10 ms CPU initial, 50 subrequests/req, 1000 invocations/min). Comfortable headroom for MVP.
- [ ] **GitHub Actions minutes quota**: confirm the repo has enough monthly Actions minutes for a build+deploy of ~2–3 minutes per merge (Free tier on personal repos is 2,000 min/month; public repos are unlimited).
- [ ] **Account ID**: visible in the dashboard URL (`https://dash.cloudflare.com/<account-id>`) and in the right sidebar of any Workers page. Needed for the `CLOUDFLARE_ACCOUNT_ID` GitHub secret in Phase 6 and for `wrangler` errors that ask for it.

### Network / firewall

- [ ] Outbound HTTPS open to: `api.cloudflare.com`, `dash.cloudflare.com`, `workers.cloudflare.com`, `<project-ref>.supabase.co` (set in Phase 2), `miapi.uk`, `registry.npmjs.org`, `github.com`. On corporate networks, get these allow-listed first.
- [ ] Loopback `http://localhost:8976` reachable for the `wrangler login` OAuth callback. If blocked, use the laptop API-token path above.

### Secrets handling discipline

- [ ] You have somewhere safe (password manager / 1Password / Bitwarden) to store: Supabase DB password (set during Phase 2 project creation), Supabase anon key, MIAPI key, and the `grounded-gha-deploy` Cloudflare API token (Phase 6). Never paste these into chat, commits, or shared docs.
- [ ] Confirm `[.env.example](../../../.env.example)` uses placeholder values only — never real keys.

## Phases

### Phase 0 — Pre-flight audit (read-only)

- [x] `npx wrangler whoami` succeeds; account is the intended one.
- [x] Confirm [`wrangler.jsonc`](../../../wrangler.jsonc) already uses Workers (not Pages): `main: "@astrojs/cloudflare/entrypoints/server"`, `assets.binding: "ASSETS"`, `nodejs_compat` present.
- [x] Confirm [`astro.config.mjs`](../../../astro.config.mjs) has `output: "server"` and `adapter: cloudflare()`.
- [x] `npx astro sync && npm run build` is green locally before touching anything.

### Phase 1 — Repo hygiene (small edits, no deploy yet)

- [x] Rename worker: [`wrangler.jsonc`](../../../wrangler.jsonc) `"name": "10x-astro-starter"` → `"name": "grounded"`.
- [x] Add `secrets.required` declaration to [`wrangler.jsonc`](../../../wrangler.jsonc) (March 2026 feature) so `wrangler deploy` fails fast if a secret is missing:

```jsonc
"secrets": {
  "required": ["SUPABASE_URL", "SUPABASE_KEY", "MIAPI_API_KEY"]
}
```

- [x] Keep `compatibility_date: "2026-05-08"` (already past the `2026-02-24` cutoff that auto-enables `fetch_iterable_type_support` — sidesteps the [withastro/astro#15434](https://github.com/withastro/astro/issues/15434) `[object Object]` regression). If we ever roll back the compat date, add `"fetch_iterable_type_support"` to `compatibility_flags` explicitly.
- [x] Fix the MIAPI key typo in [`.env.example`](../../../.env.example) and [`.dev.vars`](../../../.dev.vars): `MIAMI_API_KEY` → `MIAPI_API_KEY`.
- [x] Add `MIAPI_API_KEY` to the `astro:env/server` schema in [`astro.config.mjs`](../../../astro.config.mjs):

```js
env: {
  schema: {
    SUPABASE_URL: envField.string({ context: "server", access: "secret", optional: true }),
    SUPABASE_KEY: envField.string({ context: "server", access: "secret", optional: true }),
    MIAPI_API_KEY: envField.string({ context: "server", access: "secret", optional: true }),
  },
},
```

- [x] **(Optional cleanup)** Remove the `SUPABASE_URL` / `SUPABASE_KEY` `env:` pass-through on the `npm run build` step in [`.github/workflows/ci.yml`](../../../.github/workflows/ci.yml) — the env schema marks them `optional: true`, so CI no longer needs GitHub secrets for a green build.
- [x] Run `npx astro sync && npm run lint && npm run build` — all green.
- [x] Note: [`.github/workflows/deploy.yml`](../../../.github/workflows/deploy.yml) is added in Phase 6 — do not add it here.

### Phase 2 — Provision cloud Supabase project (external integration)

- [x] Create a new Supabase project (free tier, EU region — closer to Cloudflare's EU PoPs).
- [x] Disable **Email Confirmation** in Auth settings (MVP commitment in PRD FR-001).
- [x] Copy `Project URL` and `anon` public key from Settings → API.
- [x] (Future) Note the SQL editor location for FR-006-008 history tables with RLS — out of scope for this plan, but document the URL.

Edge-case support steps:

- If signup later silently fails on the deployed Workers URL but works locally, re-check that **Email Confirmation** is off and that no Auth redirect URL whitelist blocks the `*.workers.dev` origin.
- If you see CORS errors from Supabase, verify the Worker is using the `@supabase/ssr` **server** client (not the browser client) — [`src/lib/supabase.ts`](../../../src/lib/supabase.ts) already does.

### Phase 3 — Local smoke against cloud Supabase

- [x] Update [`.dev.vars`](../../../.dev.vars) to the cloud Supabase URL/key (replace the `127.0.0.1` values).
- [x] `npm run dev` → sign up + sign in + visit `/dashboard` + sign out — all work.
- [x] Confirm `@supabase/ssr` resolved version: `npm ls @supabase/ssr` shows `≥ 0.10.x` (PR #210 cookie-chunk fix landed in this line). If pre-fix, run `npm update @supabase/ssr`. — resolves to `0.10.3`.
- [x] `npm run build` produces a successful workerd bundle.

### Phase 4 — First manual deploy from laptop (the smoke-test gate)

- [x] `npx wrangler login` (if not already authed).
- [ ] Set secrets (GitHub Actions deploy will reuse these — they live on the Worker, not in the workflow):
  - [x] `npx wrangler secret put SUPABASE_URL` (paste cloud URL).
  - [x] `npx wrangler secret put SUPABASE_KEY` (paste anon key).
  - [x] `npx wrangler secret put MIAPI_API_KEY` (paste MIAPI key — strip the typo'd one).
- [x] `npx wrangler deploy` succeeds; note the `https://grounded.<account>.workers.dev` URL. — live at `https://grounded.samuel-liotta.workers.dev` (version `55f26b4f-b649-4a71-8091-7efcde5b85ac`).
- [x] `npx wrangler tail` streams logs without `[object Object]` or workerd warnings. — `GET /`, `GET /signin` (404), and `GET /dashboard` (302) all logged as `Ok` with no runtime exceptions.

Notes from this deploy:

- Wrangler provisioned `grounded-session` KV namespace automatically on first deploy.
- Account subdomain `samuel-liotta.workers.dev` had to be registered once in the dashboard before publishing succeeded (Wrangler v4 dropped the standalone `subdomain` command, so this is dashboard-only).
- Wrangler warned that `workers_dev` and `preview_urls` are not explicitly set in [`wrangler.jsonc`](../../../wrangler.jsonc) — both default to `true`. Leaving as-is for MVP; Phase 9 can decide whether to pin `preview_urls = false` to keep surface area tight per the "no PR preview deploys" decision in the deploy topology.

Edge-case support steps:

- If `wrangler deploy` errors on **secrets validation** (the new `secrets.required` check), the error will name the missing secret — set it and retry.
- If you see `[object Object]` response bodies, add `"disable_nodejs_process_v2"` to `wrangler.jsonc` `compatibility_flags` and redeploy ([withastro/astro#15434](https://github.com/withastro/astro/issues/15434)).
- If a dependency fails with `Cannot find module 'node:fs'` or similar at runtime, treat the dep as workerd-incompatible — do not chase it, find a Web-API alternative (per Risk register row 2 of [`context/foundation/infrastructure.md`](../../foundation/infrastructure.md)).

### Phase 5 — Post-deploy smoke (the NFR gates)

- [x] **Auth on deployed URL**: signup → signin → `/dashboard` → signout on the `*.workers.dev` URL. Cookie set/clear works across requests.
- [x] **Cookie chunk sanity**: stay signed in across a hard refresh and a tab close/reopen — verifies large `sb-*` cookie chunks survive the workerd cookie path.
- [ ] **MIAPI latency** (when `src/lib/services/miapi.ts` lands): wire a minimal `POST /api/_diag/miapi` test route (`export const prerender = false`), call `POST /v1/answer` with `citations: true` and `include_sources: true`, log `Date.now()` deltas to `wrangler tail`. Confirm p50 well under 5 s wall-clock. Remove the diag route after measurement. — deferred: `src/lib/services/miapi.ts` is not present yet.
- [ ] **Subrequest budget**: in `wrangler tail`, confirm a single answer request makes 1-2 outbound fetches to MIAPI (well under Free plan's 50/req cap). — deferred until the MIAPI service/route exists.
- [x] **Observability**: open Cloudflare dashboard → Workers → grounded → Logs; confirm `observability.enabled` is recording invocations.

Automated Phase 5 route/log probe: `GET /`, `GET /auth/signin`, and `GET /auth/signup` returned 200; unauthenticated `GET /dashboard` returned 302; `wrangler tail` logged all as `Ok` with no `[object Object]` or workerd warnings. Fresh automated signup was blocked by Supabase Auth `email rate limit exceeded`, so deployed auth/cookie checks were completed manually with an existing test account.

Edge-case support steps:

- If MIAPI p95 occasionally exceeds 5 s, enforce a per-request timeout in `src/lib/services/miapi.ts` (`AbortController` with ~4500 ms) before this risk reaches users (Risk register row 3 of [`context/foundation/infrastructure.md`](../../foundation/infrastructure.md)).
- If Supabase Auth behaves differently on the deployed URL versus locally, the smoke checklist above must pass before progressing to Phase 6 (Risk register row 4).

### Phase 6 — Set up GitHub Actions deploy workflow (auto-deploy on push to main)

Per [Cloudflare GitHub Actions docs](https://developers.cloudflare.com/workers/ci-cd/external-cicd/github-actions/).

**Create the Cloudflare API token**

- [ ] Dashboard → **Account API tokens** → **Create Token**.
- [ ] Template: **Edit Cloudflare Workers** (or custom: `Account → Workers Scripts: Edit` + `Account → Account Settings: Read` + `Zone → Workers Routes: Edit` if/when a custom domain lands).
- [ ] Scope: **only the Cloudflare account that owns `grounded`** (do not grant All accounts).
- [ ] Name it `grounded-gha-deploy` so it's distinguishable from your laptop token.
- [ ] Store in a password manager immediately; the dashboard only shows the value once.

**Find the Cloudflare Account ID**

- [x] Dashboard URL `https://dash.cloudflare.com/<account-id>` or the right sidebar of any Workers page (also returned by `npx wrangler whoami`).

**Add GitHub repo secrets**

GitHub → repo → Settings → Secrets and variables → Actions → New repository secret:

- [x] `CLOUDFLARE_API_TOKEN` = the token from above.
- [x] `CLOUDFLARE_ACCOUNT_ID` = the account ID from above.
- [x] Verify neither leaks into commits or logs (GitHub redacts known secret values in workflow logs automatically). — verified with `gh secret list`, which shows names and update times only.

**Author [`.github/workflows/deploy.yml`](../../../.github/workflows/deploy.yml)**

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm
      - run: npm ci
      - run: npx astro sync
      - run: npm run build
      - name: Deploy Worker
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
```

- [x] Added [`.github/workflows/deploy.yml`](../../../.github/workflows/deploy.yml) and validated the YAML locally.

Workflow notes:

- `node-version-file: .nvmrc` keeps Node pinned to `v22.14.0` from a single source of truth.
- No lint step — PR-time `ci.yml` is the lint gate, backed by branch protection on `main`.
- `wrangler-action@v3` defaults to `wrangler deploy` with the Worker's [`wrangler.jsonc`](../../../wrangler.jsonc) — no extra `command:` needed.
- `actions/checkout` and `actions/setup-node` are pinned at `@v4` to match [`ci.yml`](../../../.github/workflows/ci.yml); the Cloudflare doc currently shows `@v6` for checkout — bump both workflows together in a future PR.
- `timeout-minutes: 10` — build + deploy completes in ~2–3 minutes; a longer cap just hides hangs.

**Branch protection on `main`**

- [x] GitHub → repo → Settings → Branches → add a rule for `main` requiring (a) PRs before merge, (b) `ci` workflow checks to pass. Without this, anyone with write access can push directly to `main` and skip lint.

Status: enforced. `gh api repos/smlltt/grounded/branches/main/protection` now reports `required_status_checks.checks: [{ context: "ci" }]` and PR reviews required.

**Verification deploy**

- [x] Push a trivial commit to `main` (e.g., bump a comment in [`README.md`](../../../README.md)); confirm the workflow run is green, a new deployment ID appears in `npx wrangler deployments list`, and the workers.dev URL serves the new version. — `CI` and `Deploy` both succeeded for commit `4b7e922d3b1cac8aa83dc1f7739f079dc493c617`; Wrangler deployment version `860f1468-8894-4811-a2c6-8cf9c1d8e0a7`; `https://grounded.samuel-liotta.workers.dev/` returns HTTP 200.

Edge-case support steps:

- If `wrangler-action` fails with `Authentication error [code: 10000]`, the API token is missing the **Workers Scripts: Edit** permission — recreate with the **Edit Cloudflare Workers** template.
- If the deploy fails with `Missing required secret: ...` (from Phase 1's `secrets.required` declaration), the secret was never set on the Worker — run `npx wrangler secret put <NAME>` locally and rerun the workflow.
- If the build step fails on `Cannot find module 'astro:env/server'`, the workflow skipped `npx astro sync` — confirm that step is present and ran successfully.
- If you ever need to redeploy without a code change, push an empty commit (`git commit --allow-empty -m "redeploy"`) or rerun the failed workflow from the GitHub Actions UI; the action is idempotent.

### Phase 7 — Workflow role separation (`ci.yml` vs `deploy.yml`)

- [ ] [`.github/workflows/ci.yml`](../../../.github/workflows/ci.yml): triggers on push and PR to `main`. Runs `npm ci`, `npx astro sync`, `npm run lint`, `npm run build`. **Does not deploy.** Catches regressions before merge.
- [ ] [`.github/workflows/deploy.yml`](../../../.github/workflows/deploy.yml): triggers only on push to `main`. Runs `npm ci`, `npx astro sync`, `npm run build`, then `cloudflare/wrangler-action@v3`. **Does not lint** — trusts PR-time CI + branch protection.
- [ ] Both workflows are independent; either can fail without blocking the other. If a deploy fails, PRs can still pass CI and stack up safely.
- [ ] Document in [`AGENTS.md`](../../../AGENTS.md) under "Commit & Pull Request Guidelines": deploy to Cloudflare is driven by `.github/workflows/deploy.yml` on push to `main`; do not add a Wrangler step to `ci.yml`.

### Phase 8 — Rollback drill (do not skip)

- [ ] `npx wrangler deployments list` — confirm at least 2 deployment IDs exist (the manual one from Phase 4 and the GitHub Actions one from Phase 6).
- [ ] `npx wrangler rollback <previous-deployment-id>` — verify the workers.dev URL serves the older version.
- [ ] Push a trivial commit to `main` to let the deploy workflow roll forward again, restoring head.
- [ ] Document that Worker rollback and Supabase migrations are independent (Risk register row 7 of [`context/foundation/infrastructure.md`](../../foundation/infrastructure.md)): rollback does **not** revert any schema/data changes.

### Phase 9 — Docs sync

- [ ] Update the **Deployment** section of [`README.md`](../../../README.md): production deploys happen automatically on push to `main` via [`.github/workflows/deploy.yml`](../../../.github/workflows/deploy.yml) (`cloudflare/wrangler-action@v3`); `npx wrangler deploy` from a laptop remains the manual escape hatch.
- [ ] Note the rollback command in the same section.
- [ ] Update [`AGENTS.md`](../../../AGENTS.md) "Commit & Pull Request Guidelines": CI runs on push/PR to `main`; deploy is driven by `deploy.yml` on push to `main` — do not add a Wrangler step to `ci.yml`.
- [ ] (Optional) Add a one-line addendum to [`AGENTS.md`](../../../AGENTS.md) under "Security & Configuration" pointing to the `secrets.required` declaration as the source of truth for required Worker secrets.

## What this plan deliberately does NOT do

- No Cloudflare Workers Builds (repo-connected) integration — GitHub Actions owns deploy.
- No custom domain / DNS work (out of scope; revisit when MVP graduates).
- No PR preview deploys (you opted out; can be added later with a separate workflow on `pull_request`).
- No new Cloudflare services bound (no D1, KV, Queues, R2, Durable Objects) — scope-discipline per Risk register row 5 of [`context/foundation/infrastructure.md`](../../foundation/infrastructure.md).
- No history/MIAPI feature work — those land in their own plans; this one only makes deploys safe.
