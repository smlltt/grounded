---
bootstrapped_at: 2026-05-19T13:42:14Z
starter_id: 10x-astro-starter
starter_name: "10x Astro Starter (Astro + Supabase + Cloudflare)"
project_name: grounded
language_family: js
package_manager: npm
cwd_strategy: git-clone
bootstrapper_confidence: first-class
phase_3_status: ok
audit_command: "npm audit --json"
---

## Hand-off

```yaml
starter_id: 10x-astro-starter
package_manager: npm
project_name: grounded
hints:
  language_family: js
  team_size: solo
  deployment_target: cloudflare-pages
  ci_provider: github-actions
  ci_default_flow: auto-deploy-on-merge
  bootstrapper_confidence: first-class
  path_taken: standard
  quality_override: false
  self_check_answers: null
  has_auth: true
  has_payments: false
  has_realtime: false
  has_ai: true
  has_background_jobs: false
```

## Why this stack

Grounded is a solo, after-hours web MVP in three weeks with email auth, private Q&A history, and AI-generated cited answers. The recommended JavaScript default — Astro 6 with React islands, TypeScript, Tailwind CSS 4, Supabase auth/database, and Cloudflare deploy — ships auth and data out of the box while keeping the stack agent-friendly and convention-based. You asked for shadcn/ui; this starter already uses React and Tailwind, so shadcn can be added during bootstrap (the external MIAPI grounding layer is wired separately). Deployment targets Cloudflare Pages; CI uses GitHub Actions with auto-deploy on merge to main.

## Pre-scaffold verification

| Signal             | Value                                              | Severity | Notes                                      |
| ------------------ | -------------------------------------------------- | -------- | ------------------------------------------ |
| npm package        | not run                                            | —        | `cmd_template` uses `git clone`; npm skipped |
| GitHub repo        | przeprogramowani/10x-astro-starter pushed 2026-05-17 | fresh    | from card `docs_url` via GitHub API        |

## Scaffold log

**Resolved invocation**: `git clone https://github.com/przeprogramowani/10x-astro-starter .bootstrap-scaffold && cd .bootstrap-scaffold && npm install`

**Strategy**: git-clone

**Exit code**: 0

**Files moved**: 31442 (includes `node_modules/`)

**Conflicts (.scaffold siblings)**: none

**.gitignore handling**: moved silently

**.bootstrap-scaffold cleanup**: deleted

Upstream `.git/` removed before move-up. Existing `context/` preserved.

## Post-scaffold audit

**Tool**: npm audit --json

**Summary**: 0 CRITICAL, 1 HIGH, 10 MODERATE, 0 LOW

**Direct vs transitive**: 0/0/3/0 direct of total 0/1/10/0 (npm `isDirect` flag)

#### CRITICAL findings

None.

#### HIGH findings

- **devalue** (transitive) — Svelte devalue: DoS via sparse array deserialization ([GHSA-77vg-94rm-hx3p](https://github.com/advisories/GHSA-77vg-94rm-hx3p)). Range: 5.6.3–5.8.0. Fix available: yes.

#### MODERATE findings

- **@astrojs/check** (direct) — via `@astrojs/language-server` → `volar-service-yaml`. Fix: downgrade to 0.9.2 (semver-major).
- **@astrojs/cloudflare** (direct) — via `@cloudflare/vite-plugin`, `wrangler`. Fix: upgrade to 12.6.13 (semver-major).
- **wrangler** (direct) — via `miniflare` → `ws`. Fix: downgrade to 3.107.3 (semver-major).
- **@astrojs/language-server** (transitive) — via `volar-service-yaml`.
- **@cloudflare/vite-plugin** (transitive) — via `miniflare`, `wrangler`, `ws`.
- **miniflare** (transitive) — via `ws`.
- **volar-service-yaml** (transitive) — via `yaml-language-server`.
- **ws** (transitive) — Uninitialized memory disclosure ([GHSA-58qx-3vcg-4xpx](https://github.com/advisories/GHSA-58qx-3vcg-4xpx)).
- **yaml** (transitive) — Stack overflow via deeply nested YAML ([GHSA-48c2-rrv3-qjmp](https://github.com/advisories/GHSA-48c2-rrv3-qjmp)).
- **yaml-language-server** (transitive) — via `yaml`.

#### LOW / INFO findings

None.

## Remediation (2026-05-19)

Addressed all 11 bootstrap audit findings. Post-remediation: **0 vulnerabilities** (`npm audit`), `npm run build` passes.

| Finding | Action |
| ------- | ------ |
| **devalue** (HIGH) | `npm audit fix` — bumped transitive `devalue` to 5.8.1 |
| **ws** (MODERATE) | Added npm override `"ws": "^8.20.1"` (wrangler/miniflare chain was on 8.18.0) |
| **yaml** (MODERATE) | Added npm override `"yaml": "^2.8.3"` (`yaml-language-server` nested dep was on 2.7.1) |
| **@astrojs/check** chain | Resolved via `yaml` override; no downgrade of `@astrojs/check` |

`package.json` overrides now:

```json
"overrides": {
  "vite": "^7.3.2",
  "ws": "^8.20.1",
  "yaml": "^2.8.3"
}
```

Did **not** run `npm audit fix --force` (would have downgraded `@astrojs/cloudflare` to 12.x and `@astrojs/check` to 0.9.2).

## Hints recorded but not acted on

| Hint                    | Value                    |
| ----------------------- | ------------------------ |
| bootstrapper_confidence | first-class              |
| quality_override        | false                    |
| path_taken              | standard                 |
| self_check_answers      | null                     |
| team_size               | solo                     |
| deployment_target       | cloudflare-pages         |
| ci_provider             | github-actions           |
| ci_default_flow         | auto-deploy-on-merge     |
| has_auth                | true                     |
| has_payments            | false                    |
| has_realtime            | false                    |
| has_ai                  | true                     |
| has_background_jobs     | false                    |

## Next steps

Next: a future skill will set up agent context (CLAUDE.md, AGENTS.md). For now, your project is scaffolded and verified — happy hacking.

Useful manual steps in the meantime:
- `git init` (if you have not already) to start your own repo history.
- Review any `.scaffold` siblings the conflict policy created and decide which version of each file to keep.
- ~~Address audit findings~~ — remediated 2026-05-19 (see **Remediation** section).
