---
project: Grounded
context_type: greenfield
product_type: web-app
target_scale:
  users: medium
created: 2026-05-19
updated: 2026-05-19
timeline_budget:
  mvp_weeks: 3
  hard_deadline: null
  after_hours_only: true
checkpoint:
  current_phase: 8
  phases_completed: [1, 2, 3, 4, 5, 6, 7]
  frs_drafted: 9
  gray_areas_resolved:
    - topic: pain category
      decision: workflow friction (manual search) plus agent overconfidence / weak sourcing
    - topic: insight
      decision: citations and visible sources make trust auditable
    - topic: primary persona
      decision: solo knowledge worker / researcher
    - topic: auth model
      decision: email + password login; register and login
    - topic: role model
      decision: flat user model — each user sees only their own history
  frs_drafted: 0
  quality_check_status: accepted
---

# Shape notes — Grounded

## Vision & Problem Statement

Solo knowledge workers and researchers waste hours getting to answers they can trust. Manual web search means opening many tabs and filtering commercial and spam content before real work can start. Using AI agents alone is no better: models answer with overconfidence, under-emphasize sources, and leave users unable to verify what they were told.

Grounded exists because auditable answers beat persuasive ones. Every response ties claims to visible, openable sources so users judge evidence themselves instead of trusting a black box.

## User & Persona

**Primary persona — Solo researcher**

A knowledge worker or independent researcher who regularly needs current, reliable information — for work decisions, deep learning, or fact-checking before committing to a view. They are comfortable with the web but distrust both raw search results and chat-style agents that sound right without proof. The moment they reach for Grounded is when a question needs verified, up-to-date sources, not a quick trivia lookup: they want an answer they can cite and defend, with evidence one click away.

## Access Control

- **Authentication:** Email + password — users register and log in.
- **Authorization:** Flat user model. No admin, teams, or shared workspaces in MVP. Each authenticated user accesses only their own question/answer history.
- **Unauthenticated use:** None in MVP — login required to ask questions and access history.

## Success Criteria

### Primary

- End-to-end MVP flow works: register/login → ask question → cited answer with source list → saved to private history → re-view from cache without new MIAPI call.
- ≥90% of API-generated answers include at least 2 cited sources.
- User receives an answer in under 5 seconds (submit to display).
- ≥80% of test users rate answers as sufficiently accurate to begin further research.

### Secondary

- (none locked in Phase 3)

### Guardrails

- Private history: authenticated users never see another user's questions or answers.

## Functional Requirements

### Authentication

- FR-001: User can register with email and password. Priority: must-have
  > Socrates: Counter — email verification and password reset balloon auth scope before Q&A works. Resolution: MVP registers without email verification; verification deferred to v2.
- FR-002: User can log in and log out. Priority: must-have
  > Socrates: No counter-argument; stands as written.

### Q&A core

- FR-003: User can submit a question via a text form. Priority: must-have
  > Socrates: Counter — without rate limits, open ask is an abuse/cost vector. Resolution: simple per-user rate limit added (FR-009).
- FR-004: User receives an answer with inline citations [n] tied to sources. Priority: must-have
  > Socrates: Counter — MIAPI may not return citation-friendly structure; parsing risk. Resolution: validate MIAPI response contract early (spike before build); FR stands.
- FR-005: User can view a source list (title, URL, snippet) for each answer. Priority: must-have
  > Socrates: No counter-argument; stands as written.
- FR-009: System enforces a simple per-user rate limit on question submissions. Priority: must-have

### History

- FR-006: User's question and answer are saved to their private history after each ask. Priority: must-have
  > Socrates: No counter-argument; stands as written.
- FR-007: User can browse their past questions and answers. Priority: must-have
  > Socrates: No counter-argument; stands as written.
- FR-008: User can re-open a past answer from server cache without triggering a new MIAPI call. Priority: must-have
  > Socrates: No counter-argument; stands as written.

## User Stories

### US-01: Researcher gets a cited answer and finds it again later

- **Given** a registered, logged-in user on the ask page
- **When** they submit a question and the answer is generated
- **Then** they see an answer with inline citations, a source list (title, URL, snippet), and the Q&A appears in their history
- **And when** they later open that history entry
- **Then** they see the same cached answer and sources with no new API request

## Business Logic

Grounded synthesizes a web-grounded answer and attaches every claim to auditable sources so the user can verify rather than trust model confidence.

The user supplies a natural-language question. The system retrieves current web-grounded content via MIAPI and produces a single answer text where claims reference numbered sources. The user sees inline citation markers in the answer body and a parallel source list with title, URL, and snippet for each reference — they can open any source to judge credibility themselves. After generation, the question, answer, and sources are persisted to that user's private history; revisiting a history entry replays the stored response without a new generation call.

## Non-Functional Requirements

- **Latency:** For typical questions, the full answer (with citations and source list) is visible within 5 seconds of submit — measured from submit action to complete display.
- **Privacy:** A user's questions and answers are never shared with other users; Q&A is not used for model training or sold/shared to third parties (MVP commitment at product boundary).
- **Device support:** Responsive web UI readable on mobile browsers; native mobile apps out of scope for MVP.

## Non-Goals

- **Custom web crawling / indexing** — no owned crawl pipeline; all web grounding via MIAPI.
- **Advanced search filters** — no per-domain include/exclude controls in MVP.
- **Knowledge mode** — no answering from user-uploaded documents; web-only.
- **Social features** — no sharing, comments, or voting on answers.
- **Native mobile apps** — responsive web only.
- **Streaming responses** — full answer displayed when ready, not token-by-token.

## Quality cross-check

All soft-gate elements present. No gaps recorded.
