---
project: Grounded
researched_at: 2026-05-20T13:26:00Z
recommended_platform: Cloudflare Workers
runner_up: Vercel
context_type: mvp
tech_stack:
  language: TypeScript
  framework: Astro 6 SSR with React islands
  runtime: Cloudflare Workers
---

## Recommendation

**Deploy on Cloudflare Workers.**

Cloudflare Workers is the best fit for Grounded because the chosen stack already uses `astro@^6.3.1`, `@astrojs/cloudflare@^13.5.0`, and `wrangler@^4.90.0`, and the app is stateless request/response with external Supabase services. It scored strongest against the agent-friendly criteria while matching the existing adapter and keeping MVP traffic in the free tier.

## Platform Comparison

| Platform | CLI-first | Managed/Serverless | Agent-readable docs | Stable deploy API | MCP / Integration | Total |
|---|---|---|---|---|---|---|
| Cloudflare Workers | Pass | Pass | Pass | Pass | Pass | 5 / 5 |
| Vercel | Pass | Pass | Pass | Pass | Partial | 4.5 / 5 |
| Netlify | Partial | Pass | Pass | Partial | Pass | 4 / 5 |
| Railway | Partial | Pass | Pass | Partial | Pass | 4 / 5 |
| Render | Pass | Pass | Pass | Pass | Partial | 4.5 / 5 |
| Fly.io | Pass | Partial | Pass | Pass | Partial | 4 / 5 |

Cloudflare Workers passed all five criteria. Wrangler supports scriptable deploys, rollbacks, and log tailing; Cloudflare publishes agent-readable docs through `llms.txt` and markdown sources; and Cloudflare MCP servers cover docs, Workers, and observability. The main note is to target Workers rather than Cloudflare Pages for this Astro SSR app, since Pages is now the weaker long-term target.

Vercel is a strong runner-up with excellent Astro SSR DX, agent-readable docs, and a mature CLI. It loses points because the MCP surface is beta and the current project would need to switch from the Cloudflare adapter to the Vercel adapter. Its Hobby tier is also non-commercial, so a monetized MVP would need Pro.

Netlify is viable for Astro 6 SSR with `@astrojs/netlify`, strong docs, an official MCP server, and low MVP cost. It is weaker for agent maintenance because rollback is not fully CLI-first, and adopting it would require an adapter/runtime switch from the current Cloudflare setup.

Railway is a good container-style PaaS with agent-readable docs and MCP support, but it would require switching to `@astrojs/node`, binding to `0.0.0.0`, and accepting dashboard-only rollback. Its co-located databases are useful but not important because Grounded already uses Supabase externally.

Render is a comfortable Node-hosting option for Astro SSR and avoids edge-runtime constraints, but the free tier's cold starts are incompatible with the product's under-5-second response target. The paid Starter instance is reasonable, yet still adds cost and adapter migration without enough benefit for this MVP.

Fly.io provides the most control and supports persistent processes, but Grounded does not need that capability. The platform is more operationally exposed than serverless targets, has no true free tier, and would require switching to `@astrojs/node`.

### Shortlisted Platforms

#### 1. Cloudflare Workers (Recommended)

Cloudflare Workers won because it matches the already-selected runtime, keeps the deployment path smallest, has a strong CLI-first operational loop, and easily handles the expected 10k-100k monthly requests at MVP scale. It also fits the interview constraints: no persistent server process, no strict co-location requirement, and Supabase as an external managed data layer.

#### 2. Vercel

Vercel scored second because it has excellent Astro support, strong preview deployments, readable docs, and a very polished developer experience. The gap is that it requires changing adapters and may force a paid plan earlier if Grounded becomes commercial.

#### 3. Netlify

Netlify scored third because it has good Astro 6 support, low MVP costs, and strong AI tooling. It falls behind Vercel and Cloudflare because rollback is less scriptable and the project would still need to leave the Cloudflare runtime it already uses.

## Anti-Bias Cross-Check: Cloudflare Workers

### Devil's Advocate — Weaknesses

1. The starter's Cloudflare direction can hide an important product choice: Workers is the right target, while Pages is now a weaker choice for a new SSR app.
2. Astro 6 on workerd can expose edge-runtime limits, including no native Node modules, limited filesystem access during prerender, and the need for `nodejs_compat` when dependencies expect Node APIs.
3. The MIAPI answer endpoint has an under-5-second product target, so upstream latency plus Workers runtime limits must be tested early.
4. Supabase from Workers means cross-provider secrets, cookies, redirects, and network behavior need careful verification.
5. Cloudflare has many tempting services, and adding D1, Queues, Durable Objects, or R2 before the MVP needs them would increase scope.

### Pre-Mortem — How This Could Fail

Six months after launch, Cloudflare Workers turned into a bad decision because the team treated Pages and Workers as interchangeable and discovered too late that the Astro SSR deployment path needed Workers-specific configuration. The answer endpoint depended on a package that worked locally but failed in workerd because it expected native Node APIs or filesystem access. The team also missed early latency testing, so MIAPI calls sometimes exceeded the product's response target and were hard to debug across Astro, Wrangler, Workers logs, and Supabase. To compensate, the project added Cloudflare services it did not need, creating more configuration, more secrets, and more moving parts. The result was not a platform failure so much as a scope failure: the MVP stopped being a small stateless web app and became a Cloudflare architecture exercise.

### Unknown Unknowns

- Astro 6's Cloudflare adapter uses workerd behavior for SSR and prerendering; local `astro dev` can feel normal while deployment exposes runtime constraints.
- Cloudflare Workers and Cloudflare Pages should not be treated as equivalent targets for this project; Workers is the safer recommendation for a new SSR deployment.
- `nodejs_compat` can unblock many packages, but it is not a guarantee that every Node-oriented dependency will work on Workers.
- Cloudflare rollback and logs are scriptable through Wrangler, but runtime diagnosis still requires the team to know whether an issue is build-time, adapter-time, or request-time.
- Supabase Auth cookie behavior should be tested on the deployed Workers URL before custom domains and preview deploys are considered done.

## Operational Story

- **Preview deploys**: Use Wrangler versioned deployments and GitHub Actions previews for branches/PRs; protect preview URLs with Cloudflare Access if real user data or production Supabase credentials are ever exposed.
- **Secrets**: Store local development values in `.dev.vars`; set production values with `npx wrangler secret put SUPABASE_URL`, `npx wrangler secret put SUPABASE_KEY`, and later `npx wrangler secret put MIAPI_API_KEY`; rotate by writing the new secret and redeploying.
- **Rollback**: Use `npx wrangler deployments list` to find a prior deployment and `npx wrangler rollback <deployment-id>` to revert the Worker; database migrations in Supabase do not roll back automatically.
- **Approval**: An agent may run read-only checks, build verification, preview deploys, and log inspection; a human should approve production deploys, secret rotation, and Supabase schema changes that affect persisted data.
- **Logs**: Use `npx wrangler tail` for runtime logs and GitHub Actions logs for CI/build failures; keep log access read-only for routine agent diagnosis.

## Risk Register

| Risk | Source | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| Accidentally targeting Cloudflare Pages instead of Workers | Devil's advocate | M | M | Document Workers as the deployment target and keep Wrangler/Workers config as the source of truth. |
| Node-only dependency fails in workerd | Devil's advocate / Unknown unknowns | M | H | Run `npm run build` and deploy a smoke test early; avoid native modules and verify any AI/API SDKs under Workers. |
| MIAPI latency breaks the under-5-second goal | Pre-mortem | M | H | Add an early spike for `POST /v1/answer`, measure deployed latency, and enforce request timeouts in code. |
| Supabase auth behaves differently after deployment | Unknown unknowns | M | H | Test register/login/logout and private history on the deployed Workers URL before considering auth complete. |
| Cloudflare service sprawl expands MVP scope | Devil's advocate / Pre-mortem | M | M | Keep the MVP to Workers plus external Supabase; add D1, Queues, Durable Objects, or R2 only when a concrete requirement appears. |
| Secrets drift between local, preview, and production | Research finding | M | M | Maintain a minimal secret inventory and set production values only through Wrangler secrets or Cloudflare dashboard-controlled secrets. |
| Rollback hides data incompatibility | Research finding | L | H | Treat Worker rollback and Supabase migrations separately; require human approval for irreversible schema/data changes. |

## Getting Started

1. Keep the current Astro configuration: `output: "server"` with `adapter: cloudflare()` in `astro.config.mjs`.
2. Add the missing MIAPI secret to Astro's env schema and Cloudflare secrets before implementing the answer endpoint.
3. Run `npx astro sync` and `npm run build` locally to validate Astro 6 + `@astrojs/cloudflare@^13.5.0`.
4. Deploy with the pinned toolchain using `npx wrangler deploy`; inspect runtime behavior with `npx wrangler tail`.
5. After the first deployed build, smoke-test auth, private history, and the MIAPI answer request against the deployed Workers URL.

## Out of Scope

The following were not evaluated in this research:

- Docker image configuration
- CI/CD pipeline setup
- Production-scale architecture (multi-region, HA, DR)
