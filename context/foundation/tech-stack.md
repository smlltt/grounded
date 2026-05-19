---
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
---

## Why this stack

Grounded is a solo, after-hours web MVP in three weeks: cited Q&A with email auth, private history, and external MIAPI grounding. The recommended JavaScript default — Astro + Supabase + Cloudflare — ships TypeScript, auth, PostgreSQL, and edge deploy together, which matches auth and persistence FRs without spinning up separate services. Cloudflare Pages is the starter default; GitHub Actions with auto-deploy on merge keeps CI simple. AI answer generation stays an integration concern (MIAPI spike per PRD); the edge runtime is fine for request/response flows but not long background jobs — acceptable for MVP scope.
