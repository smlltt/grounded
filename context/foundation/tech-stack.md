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

Grounded is a solo, after-hours web MVP in three weeks with email auth, private Q&A history, and AI-generated cited answers. The recommended JavaScript default — Astro 6 with React islands, TypeScript, Tailwind CSS 4, Supabase auth/database, and Cloudflare deploy — ships auth and data out of the box while keeping the stack agent-friendly and convention-based. You asked for shadcn/ui; this starter already uses React and Tailwind, so shadcn can be added during bootstrap (the external MIAPI grounding layer is wired separately). Deployment targets Cloudflare Pages; CI uses GitHub Actions with auto-deploy on merge to main.
