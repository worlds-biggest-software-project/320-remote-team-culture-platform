# Remote Team Culture Platform — Phased Development Plan

> Project: 320-remote-team-culture-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three data-model suggestions. The database design adopts **Data Model Suggestion 1 (Entity-Centric Normalized Relational)** as its foundation, because the platform's differentiating features — matching-fairness constraints (no repeat pairings within N rounds), recognition budgets/leaderboards, an explicit connection graph, and isolation-risk detection — all benefit from database-level integrity and indexed relational queries. Polymorphic JSONB (Model 2) is used selectively for matching rules and AI output payloads where shape genuinely varies.

---

## Product Summary

**What it does.** An AI-native, open-source platform that builds remote-team culture by delivering virtual-coffee matching, peer recognition, birthday/anniversary celebrations, pulse surveys, and engagement analytics directly inside Slack and Microsoft Teams, with an admin web portal for HR/People-Ops configuration and reporting.

**Who uses it.** HR/People-Ops leaders (configure programs, read analytics), engineering managers (team health), DEI leaders (connection equity), and employees (participate entirely within their chat tool — zero separate app).

**Key differentiators.** (1) First-class cross-platform parity (Slack *and* Teams); (2) AI matching beyond randomness (skills/goals/DEI-aware); (3) isolation-risk detection from a persistent connection graph before attrition; (4) AI monthly culture narrative; (5) self-hostable and affordable versus fragmented incumbents.

**Deployment model.** Self-hosted (Docker Compose) and cloud SaaS — same codebase, multi-tenant by `org_id`.

**MVP scope (from features.md).** Slack + Teams bots; configurable matching engine (random, rules-based, basic affinity); birthday/anniversary automation with HRIS sync (BambooHR, Workday); participation analytics dashboard; OAuth 2.0 + GDPR-compliant data handling.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | **Python 3.12** | The AI-native core (matching optimisation, isolation detection, narrative generation) is the differentiator; Python has the strongest LLM SDK and data-tooling ecosystem. Chat SDKs (Slack Bolt, Bot Framework) all ship first-class Python support. |
| API / web framework | **FastAPI** | Async-native (chat webhooks + long LLM calls are I/O-bound), auto-generates the OpenAPI 3.1 spec required by `standards.md`, integrates with Pydantic for request/response validation (JSON Schema 2020-12). |
| ASGI server | **Uvicorn** (behind Gunicorn workers) | Standard production ASGI runner for FastAPI. |
| Data validation | **Pydantic v2** | Config, webhook payloads, survey bodies, and matching rules become typed models; emits JSON Schema for the OpenAPI doc. |
| Database | **PostgreSQL 16** | Model 1 needs FK constraints, partial indexes, `JSONB` for matching rules, array columns, and recursive/self-join queries for the connection graph. SQLite cannot express the required partial indexes and `JSONB` operators cleanly. |
| ORM / migrations | **SQLAlchemy 2.0 (async) + Alembic** | Async ORM matches FastAPI; Alembic gives versioned migrations (Definition of Done requires migrations per phase). |
| Task queue | **Celery + Redis** (broker + result backend) | Matching rounds, HRIS sync, celebration posting, survey sends, and LLM calls are async/scheduled. Celery Beat drives cadence (weekly/biweekly/monthly) scheduling. |
| Cache / locks / rate-limit | **Redis** | Doubles as Celery broker, recognition daily-cap counters, OAuth state store, and idempotency keys for webhook de-duplication. |
| Slack integration | **slack-bolt (Python) + slack-sdk** | Official framework; handles Events API signature verification, Block Kit, slash commands, interactivity, OAuth installation store. |
| Teams integration | **botbuilder-core / botbuilder-integration-aiohttp (Bot Framework SDK)** + **Microsoft Graph SDK** | Official Teams bot path; Adaptive Cards v1.5 for parity with Slack Block Kit; Graph for roster/calendar. |
| LLM access | **Provider-agnostic gateway** via the `litellm` client (defaults to OpenAI/Anthropic) | Keeps the AI layer swappable; the same interface serves matching scoring, isolation narratives, and culture reports. All prompts versioned in-repo. |
| HRIS sync | **Pluggable connectors** with optional **Merge unified API** adapter | `standards.md` notes HRIS fragmentation; ship native BambooHR + Workday connectors for MVP, behind a `HRISConnector` interface that a Merge adapter can later satisfy for 50+ providers. |
| Calendar | **Google Calendar API + Microsoft Graph Calendar** | One-click scheduling respecting timezones; OAuth 2.0 + PKCE per `standards.md`. |
| Admin frontend | **Next.js 15 (App Router) + TypeScript + shadcn/ui + Tailwind** | HR/People-Ops portal for configuration and analytics dashboards; SSR for auth-gated pages; network-graph and trend charts. Employees never use it — they stay in chat. |
| Graph viz | **Cytoscape.js** | Renders the relationship network graph (force-directed, isolation highlighting) in the admin portal. |
| Charts | **Recharts** | Participation/trend/pulse dashboards. |
| Auth (admin portal) | **OIDC + SAML 2.0** via **Authlib** (server) / NextAuth (client) | Enterprise SSO (Okta, Azure AD, Google). Employee identity comes from the chat platform, not a password. |
| Enterprise provisioning | **SCIM 2.0 server** (RFC 7643/7644) | Auto provision/deprovision from IdPs; implemented as a FastAPI sub-router. |
| Containerisation | **Docker + docker-compose** | Self-hosted deployment mode; one command brings up api, worker, beat, postgres, redis, frontend. |
| Testing | **pytest + pytest-asyncio + httpx.AsyncClient + testcontainers + Playwright** | Unit/integration for backend (real Postgres/Redis via testcontainers), Playwright for admin-portal E2E. |
| Code quality | **ruff (lint+format), mypy (strict), pre-commit** | Single fast toolchain; mypy enforces the typed design. |
| Package manager | **uv** (backend), **pnpm** (frontend) | Fast, reproducible installs; lockfiles committed. |
| API doc / wire format | **OpenAPI 3.1**, **SARIF-style not needed**; **CloudEvents-lite** envelope for internal engagement events | Aligns with `standards.md`; engagement events use a CloudEvents-shaped envelope inside an `engagement_events` audit table (lightweight nod to Model 3 for GDPR auditability without full event sourcing). |

### Project Structure

```
remote-team-culture-platform/
├── pyproject.toml                 # uv-managed, backend deps + tool config (ruff, mypy, pytest)
├── uv.lock
├── Dockerfile                     # multi-stage: api / worker share image
├── docker-compose.yml             # api, worker, beat, postgres, redis, frontend
├── docker-compose.dev.yml         # hot-reload overrides
├── .env.example
├── alembic.ini
├── README.md
├── openapi/                       # generated + curated OpenAPI 3.1 artefacts
│   └── openapi.json
├── migrations/                    # Alembic versions
│   └── versions/
├── prompts/                       # versioned LLM prompt templates (jinja2)
│   ├── matching_score_v1.jinja
│   ├── isolation_narrative_v1.jinja
│   ├── culture_narrative_v1.jinja
│   └── watercooler_prompt_v1.jinja
├── src/
│   └── culture/
│       ├── __init__.py
│       ├── main.py                # FastAPI app factory, router mounting
│       ├── config.py              # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── base.py            # async engine, session factory
│       │   ├── models/            # SQLAlchemy models (one file per domain)
│       │   │   ├── org.py
│       │   │   ├── user.py
│       │   │   ├── matching.py
│       │   │   ├── recognition.py
│       │   │   ├── celebration.py
│       │   │   ├── survey.py
│       │   │   ├── network.py
│       │   │   ├── integration.py
│       │   │   ├── ai.py
│       │   │   └── events.py
│       │   └── repositories/      # query objects per aggregate
│       ├── schemas/               # Pydantic request/response models
│       ├── api/                   # FastAPI routers (REST + SCIM)
│       │   ├── deps.py            # auth, tenant, db-session dependencies
│       │   ├── v1/
│       │   │   ├── programs.py
│       │   │   ├── recognition.py
│       │   │   ├── celebrations.py
│       │   │   ├── surveys.py
│       │   │   ├── analytics.py
│       │   │   ├── users.py
│       │   │   └── integrations.py
│       │   └── scim/              # SCIM 2.0 server (Users, Groups)
│       ├── chat/                  # platform-agnostic delivery layer
│       │   ├── interface.py       # ChatGateway protocol
│       │   ├── blocks.py          # canonical message model -> Block Kit / Adaptive Card
│       │   ├── slack/             # slack-bolt app, events, OAuth install store
│       │   └── teams/             # Bot Framework adapter, Adaptive Cards
│       ├── matching/
│       │   ├── engine.py          # round generation, constraint solver
│       │   ├── strategies.py      # random / rules / affinity / ai
│       │   └── ai_scorer.py       # LLM-assisted pair scoring
│       ├── recognition/
│       ├── celebrations/
│       ├── surveys/
│       ├── network/               # connection graph + isolation scoring
│       ├── analytics/             # snapshots, dashboards aggregation
│       ├── ai/
│       │   ├── llm.py             # litellm wrapper, retries, cost capture
│       │   ├── narratives.py
│       │   └── isolation.py
│       ├── integrations/
│       │   ├── hris/              # HRISConnector + bamboohr/workday/merge
│       │   └── calendar/          # google + msgraph
│       ├── auth/                  # OAuth/OIDC/SAML, PKCE, JWT sessions
│       ├── mcp/                   # MCP server exposing analytics (backlog phase)
│       └── tasks/                 # Celery tasks + beat schedule
├── tests/
│   ├── conftest.py                # testcontainers Postgres/Redis fixtures
│   ├── unit/
│   ├── integration/
│   ├── fixtures/                  # sample HRIS payloads, Slack events, rosters
│   └── e2e/
└── frontend/
    ├── package.json               # pnpm
    ├── app/                       # Next.js App Router
    │   ├── (auth)/
    │   ├── dashboard/
    │   ├── programs/
    │   ├── analytics/
    │   └── settings/
    ├── components/                # shadcn/ui + charts + cytoscape graph
    └── lib/api.ts                 # typed client generated from openapi.json
```

The structure is additive: each phase fills in modules without restructuring. Group by concern (domain modules + chat delivery + integrations), not by phase.

---

## Phase 1: Foundation — Project Skeleton, Config, Multi-Tenant Data Core

### Purpose
Establish the runnable backbone: typed configuration, async Postgres connectivity, the core multi-tenant entities (organisations, users, teams), migrations, the FastAPI app factory, health checks, and the Docker Compose stack. After this phase the app boots, connects to Postgres/Redis, exposes `/healthz` and an OpenAPI doc, and has the tables every later phase depends on.

### Tasks

#### 1.1 — Project scaffolding & tooling

**What**: Create the `uv` project, tool configuration, pre-commit, and the FastAPI app factory.

**Design**:
- `pyproject.toml` defines deps (`fastapi`, `uvicorn[standard]`, `sqlalchemy[asyncio]`, `asyncpg`, `alembic`, `pydantic-settings`, `celery[redis]`, `redis`, `httpx`, `litellm`, `slack-bolt`, `botbuilder-core`, `authlib`, `jinja2`) and dev deps (`pytest`, `pytest-asyncio`, `httpx`, `testcontainers`, `ruff`, `mypy`, `pre-commit`).
- Tool config: `ruff` (line length 100, select E/F/I/UP/B), `mypy` (`strict = true`), `pytest` (`asyncio_mode = "auto"`).
- App factory:
```python
def create_app() -> FastAPI:
    app = FastAPI(title="Remote Team Culture Platform", version="0.1.0",
                  openapi_url="/openapi.json")
    app.include_router(health_router)
    register_exception_handlers(app)
    return app
```
- `Settings` (Pydantic `BaseSettings`) reads env: `DATABASE_URL`, `REDIS_URL`, `SECRET_KEY`, `LLM_PROVIDER`, `LLM_API_KEY`, `SLACK_CLIENT_ID/SECRET/SIGNING_SECRET`, `TEAMS_APP_ID/PASSWORD`, `ENV` (`dev|prod`). Required fields raise `ValidationError` on startup.

**Testing**:
- `Unit: Settings loads from env dict → all fields typed correctly, defaults applied`
- `Unit: missing DATABASE_URL → ValidationError naming the field`
- `Integration: GET /healthz → 200 {"status":"ok"}`
- `Integration: GET /openapi.json → valid OpenAPI 3.1 document (version field == "3.1.0")`

#### 1.2 — Async DB layer & migration tooling

**What**: Async SQLAlchemy engine/session, declarative base, Alembic wired to async.

**Design**:
- `db/base.py`: `create_async_engine(settings.DATABASE_URL)`, `async_sessionmaker`, FastAPI dependency `get_session()` yielding a session per request with commit/rollback.
- All models inherit a `Base` with `id: Mapped[UUID]` default `gen_random_uuid()`, `created_at`, `updated_at` mixins.
- Alembic `env.py` configured for async + autogenerate from metadata.

**Testing**:
- `Integration (testcontainers Postgres): engine connects, SELECT 1 succeeds`
- `Integration: alembic upgrade head on empty DB → all tables exist; downgrade base → clean`

#### 1.3 — Core entities: organisations, users, teams, team_members

**What**: Implement the foundational tables from Data Model 1.

**Design** (SQL DDL adopted verbatim from `data-model-suggestion-1.md`):
```sql
CREATE TABLE organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL, slug TEXT NOT NULL UNIQUE,
    billing_plan TEXT NOT NULL DEFAULT 'free',
    timezone TEXT NOT NULL DEFAULT 'UTC',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    email TEXT NOT NULL UNIQUE, name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('admin','manager','member')),
    avatar_url TEXT, timezone TEXT NOT NULL DEFAULT 'UTC',
    department TEXT, location TEXT, hire_date DATE, birthday DATE, job_title TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    consent_engagement_tracking BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_org ON users(org_id);
CREATE INDEX idx_users_birthday ON users(org_id, birthday) WHERE birthday IS NOT NULL;
CREATE INDEX idx_users_hire_date ON users(org_id, hire_date) WHERE hire_date IS NOT NULL;
-- teams, team_members per Model 1
```
- Add `external_ids JSONB DEFAULT '{}'` on `users` to hold `{"slack":"U123","teams":"...","bamboohr":"123"}` (needed by chat + HRIS phases).
- Repositories: `OrgRepository`, `UserRepository`, `TeamRepository` with typed methods (`get_by_slack_id`, `list_active_for_org`, etc.).
- Multi-tenancy: every query is scoped by `org_id`; a `TenantContext` dependency resolves the org from the auth principal.

**Testing**:
- `Unit: user role check constraint rejects 'superuser'`
- `Integration: create org + users, idx_users_birthday partial index used (EXPLAIN) for birthday lookup`
- `Integration: deleting org cascades to users/teams`
- `Integration: UserRepository.get_by_slack_id resolves via external_ids JSONB`

#### 1.4 — Docker Compose stack

**What**: Containerise api + worker + beat + postgres + redis + frontend.

**Design**:
- `Dockerfile` multi-stage (builder installs via uv, runtime slim). Same image runs api (`uvicorn`), worker (`celery worker`), beat (`celery beat`) via command override.
- `docker-compose.yml` services with healthchecks; `depends_on` ordering; named volume for Postgres.
- `.env.example` documents every variable from 1.1.

**Testing**:
- `E2E (CI): docker compose up → /healthz returns 200 within 60s`
- `Integration: worker container starts, registers tasks (celery inspect ping)`

---

## Phase 2: Chat Delivery Layer — Slack & Teams Parity

### Purpose
Build the platform-agnostic delivery layer so every later feature (matches, recognition, celebrations, surveys) writes to a single canonical message model that renders to both Slack Block Kit and Teams Adaptive Cards. This phase delivers OAuth installation, signed-event ingestion, slash-command/interaction handling, and DM/channel sending for both platforms. After this phase the platform can install into a Slack workspace and a Teams tenant and exchange messages.

### Tasks

#### 2.1 — Canonical message model & renderers

**What**: A `Message` abstraction that renders to Block Kit and Adaptive Card v1.5.

**Design**:
```python
@dataclass
class Action:
    id: str; label: str; style: Literal["primary","default","danger"] = "default"
    value: str | None = None

@dataclass
class Message:
    text: str                       # fallback / notification text
    sections: list[str] = field(default_factory=list)  # markdown blocks
    actions: list[Action] = field(default_factory=list)
    fields: dict[str, str] | None = None  # key/value facts (e.g. celebration card)

class ChatGateway(Protocol):
    async def send_dm(self, user_external_id: str, msg: Message) -> str: ...      # returns ts/msg id
    async def post_channel(self, channel_id: str, msg: Message) -> str: ...
    async def update(self, channel_id: str, msg_id: str, msg: Message) -> None: ...
def to_block_kit(msg: Message) -> list[dict]: ...
def to_adaptive_card(msg: Message) -> dict: ...   # AdaptiveCard v1.5 schema
```
- `Action.id` becomes Slack `action_id` / Teams `Action.Submit` `data.action`; interaction routing maps it back to a handler.

**Testing**:
- `Unit: Message with 2 actions → Block Kit has actions block with 2 buttons, correct action_ids`
- `Unit: same Message → Adaptive Card v1.5 with ActionSet, version "1.5"`
- `Unit: fields dict → Block Kit fields / Adaptive Card FactSet`
- `Fixture: golden-file comparison of rendered JSON for a sample match-notification message`

#### 2.2 — Slack app: OAuth install, events, interactivity

**What**: slack-bolt app mounted into FastAPI; OAuth install store persisting bot tokens per org.

**Design**:
- OAuth Authorisation Code flow (RFC 6749); installation store writes to `integrations` table (`provider='slack'`, `credentials_enc`).
- Request signature verification via `SLACK_SIGNING_SECRET` (rejects stale/forged requests).
- Routes: `/slack/install`, `/slack/oauth_redirect`, `/slack/events`, `/slack/interactivity`, `/slack/commands`.
- Slash commands registered: `/coffee optin|optout`, `/kudos @user message`, `/pulse`.
- Idempotency: dedupe Slack `event_id` via Redis SETNX (Slack retries on timeout).

**Testing**:
- `Integration (mocked Slack API): valid signed events payload → 200, handler invoked`
- `Integration: payload with bad signature → 401, handler NOT invoked`
- `Integration: duplicate event_id within window → second call no-ops`
- `Integration: OAuth redirect with code → token exchanged (mocked), integration row written`

#### 2.3 — Teams bot: Bot Framework adapter & Graph auth

**What**: Bot Framework message endpoint + Adaptive Card delivery + Graph app auth.

**Design**:
- `/teams/messages` endpoint using `CloudAdapter`; JWT validation of Bot Framework tokens (RFC 7519).
- `TeamsChatGateway(ChatGateway)` sends proactive DMs/channel messages via conversation references stored on first contact.
- Graph app-only OAuth (client credentials) for roster/calendar; per-org tenant credentials in `integrations` (`provider='teams'`).

**Testing**:
- `Integration (mocked Bot Framework): activity with valid auth header → 200, turn handler runs`
- `Integration: invalid bearer token → 401`
- `Unit: TeamsChatGateway maps Message → activity attachment with Adaptive Card`

#### 2.4 — Interaction router & opt-in/opt-out

**What**: Route platform-agnostic actions (button clicks, submits) to handlers; persist matching opt-in.

**Design**:
- `InteractionRouter` keyed by `Action.id` prefix (`match:accept`, `match:decline`, `card:sign`, `survey:answer`).
- Opt-in stored as `users.consent_engagement_tracking` plus a `matching_opt_in BOOLEAN` column added here; slash command toggles it and replies ephemerally.

**Testing**:
- `Unit: action_id "match:accept:<pair_id>" → routed to match accept handler with parsed pair_id`
- `Integration: /coffee optout → matching_opt_in=false, ephemeral confirmation sent`
- `Integration: unknown action_id → logged, graceful no-op (no 500)`

---

## Phase 3: Matching Engine — Rounds, Constraints, Delivery

### Purpose
Ship the heart of the product: configurable matching programs that generate pairing rounds honouring fairness constraints (no repeat pairings within N rounds, avoid-same-department, group sizing), deliver match notifications to both platforms, and track accept/decline/met lifecycle. This is the core value proposition and the first thing buyers evaluate.

### Tasks

#### 3.1 — Matching schema

**What**: Implement `matching_programs`, `matching_rounds`, `match_pairs`, `match_pair_members` (Model 1).

**Design**: DDL from `data-model-suggestion-1.md` (Matching Programs & Rounds section) adopted verbatim, including `matching_rules JSONB` and the `match_pair_members` junction supporting groups of 2–4. Pair status state machine: `pending → accepted|declined|no_response`, and on confirmation `→ met`. Round status: `pending → matched → notified → completed`.

```python
class MatchingRules(BaseModel):
    avoid_same_department: bool = False
    avoid_same_location: bool = False
    avoid_repeat_within_rounds: int = 4
    prefer_different_tenure: bool = False
    dei_balance: dict | None = None
```

**Testing**:
- `Unit: MatchingRules parses partial JSON with defaults`
- `Integration: unique (program_id, round_number) enforced`
- `Integration: match_pair_members composite PK prevents duplicate member in pair`

#### 3.2 — Pairing algorithm (random + rules-based)

**What**: Deterministic, testable round generator producing valid pairs from the opted-in pool.

**Design**:
- `generate_round(program, pool, history) -> list[Pair]`. Algorithm:
  1. Filter pool to `matching_opt_in AND is_active`.
  2. Build a forbidden-pairs set from the last `avoid_repeat_within_rounds` rounds (query via `match_pair_members`).
  3. Apply rule penalties (same department/location) as soft constraints; hard-forbid only repeats.
  4. Greedy + randomised matching with a seeded RNG (seed accepted for tests); odd member out forms a triad (group_size respected).
  5. Return pairs; raise `InsufficientPoolError` if pool < 2.
- Pure function (no I/O) for unit-testability; persistence handled by caller.

**Testing**:
- `Unit (seeded): pool of 6, no history → 3 distinct pairs, everyone matched once`
- `Unit: pool of 5, group_size 2 → 1 triad + 2 pairs (or 2 pairs + 1 triad), all matched`
- `Unit: avoid_repeat_within_rounds=4 with prior pairing → that pair not reproduced`
- `Unit: avoid_same_department soft constraint → cross-dept pairs preferred when feasible`
- `Unit: pool of 1 → InsufficientPoolError`

#### 3.3 — Round orchestration & delivery (Celery + Beat)

**What**: Scheduled task that creates a round, persists pairs, and DMs each participant via the correct gateway.

**Design**:
- Celery Beat schedule derived from each active program's `cadence`; a dispatcher task enqueues `run_matching_round(program_id)`.
- `run_matching_round`: load pool + history → `generate_round` → persist round/pairs/members → for each member, render match `Message` (intro + Accept/Schedule buttons) → `gateway.send_dm`. Round status `pending→matched→notified`.
- Per-org gateway resolved from `integrations.provider`.

**Testing**:
- `Integration (mocked gateways): run_matching_round on seeded org → round+pairs persisted, send_dm called per member`
- `Integration: program with no opted-in users → round skipped, warning logged, no crash`
- `Integration: Beat schedule for weekly program enqueues exactly one dispatch per week (frozen clock)`

#### 3.4 — Match lifecycle handlers

**What**: Handle accept/decline interactions and meeting-completion feedback.

**Design**:
- `match:accept:<pair_id>` → set member.accepted=true; when all accepted, pair.status=`accepted`, send scheduling prompt.
- `match:decline` → member.accepted=false, pair.status=`declined`.
- `match:met:<pair_id>` (follow-up DM after scheduled time) → pair.meeting_completed, optional 1–5 `feedback_score`.
- Follow-up DM enqueued via Celery `eta` based on meeting time.

**Testing**:
- `Integration: both members accept → pair.status accepted, scheduling prompt sent`
- `Integration: one declines → pair.status declined, no scheduling prompt`
- `Integration: met handler with score 4 → meeting_completed=true, feedback_score=4`
- `Integration: feedback_score 7 → 400 (CHECK 1..5)`

---

## Phase 4: Recognition & Celebrations

### Purpose
Add the second and third culture surfaces: peer recognition with company values, points, and reactions; and automated birthday/work-anniversary celebrations with collaborative digital cards and HRIS-sourced dates. Both run inside chat, deepening the platform beyond matching into an all-in-one suite that replaces HeyTaco + Doozy.

### Tasks

#### 4.1 — Recognition schema & giving flow

**What**: Implement `company_values`, `recognitions`, `recognition_reactions`; wire the `/kudos` flow with daily caps.

**Design**: DDL from Model 1 Recognition section. Daily giving cap (default 5, per `data-model-suggestion-2` org settings) enforced via Redis counter `kudos:{org}:{user}:{yyyymmdd}` with TTL to midnight (user tz).
```python
async def give_recognition(giver_id, receiver_id, message, value_id=None) -> Recognition
# raises DailyLimitExceeded, SelfRecognitionError (giver != receiver)
```
- On success, post a public Block Kit / Adaptive Card shoutout in the configured channel; others can react (👏) which inserts `recognition_reactions`.

**Testing**:
- `Unit: giver == receiver → SelfRecognitionError (mirrors CHECK constraint)`
- `Integration: 6th kudos same day → DailyLimitExceeded, no row written`
- `Integration: valid kudos → row + public message; reaction insert unique per (recognition,user)`
- `Integration: value tagging links recognition to company_values row`

#### 4.2 — Recognition leaderboard & values analytics

**What**: Read endpoints powering leaderboard and value-distribution.

**Design**: Endpoints `GET /v1/recognition/leaderboard?period=30d` and `GET /v1/recognition/values-breakdown`, backed by the leaderboard query from Model 1's Example Queries.

**Testing**:
- `Integration: seed 10 recognitions → leaderboard ordered by total_points desc`
- `Integration: values-breakdown returns count per value`

#### 4.3 — Celebrations schema & detection

**What**: Implement `celebrations`, `celebration_signers`; daily Beat task to detect upcoming dates.

**Design**: DDL from Model 1 Celebrations section. Daily task scans `users` (using `idx_users_birthday` / `idx_users_hire_date`) for events `lead_days` ahead, creates `celebrations` rows (`is_posted=false`), and opens a digital card (DMs teammates to sign).
- Anniversary computes `years = current_year - hire_year`.

**Testing**:
- `Integration: user with birthday in lead_days window → celebration row created once (idempotent on re-run)`
- `Integration: anniversary years computed correctly`
- `Integration: no upcoming events → no rows`

#### 4.4 — Digital card signing & posting

**What**: Collaborative card with sign nudges, posted on the celebration date.

**Design**:
- `card:sign:<celebration_id>` opens a modal/submit capturing a message → inserts `celebration_signers`.
- Nudge: if signers < threshold the day before, DM remaining teammates.
- On the day, Beat task posts the assembled card (FactSet of signer messages) to the channel and sets `is_posted=true, posted_at`.

**Testing**:
- `Integration: sign action inserts signer row; duplicate sign by same user → upsert, not duplicate`
- `Integration: posting task on date → channel post, is_posted true`
- `Integration: idempotent — already-posted celebration not re-posted`

---

## Phase 5: HRIS Sync & Calendar Scheduling

### Purpose
Connect the platform to systems of record so rosters stay current and meetings get booked without leaving chat. HRIS sync (BambooHR, Workday) keeps the matching pool and celebration dates accurate; calendar integration provides one-click, timezone-aware scheduling for matched pairs. This phase can be developed in parallel with Phase 6.

### Tasks

#### 5.1 — HRISConnector interface & BambooHR connector

**What**: Pluggable HRIS abstraction with a BambooHR implementation.

**Design**:
```python
class EmployeeRecord(BaseModel):
    external_id: str; email: str; name: str
    department: str | None; location: str | None
    hire_date: date | None; birthday: date | None; job_title: str | None
    is_active: bool = True

class HRISConnector(Protocol):
    async def list_employees(self) -> list[EmployeeRecord]: ...
    async def changed_since(self, since: datetime) -> list[EmployeeRecord]: ...
```
- BambooHR: HTTP Basic auth + API key per subdomain (per `standards.md`); uses the Changed Employees endpoint for incremental sync.
- Sync upserts into `users` keyed by `external_ids.bamboohr`; deactivations set `is_active=false`.

**Testing**:
- `Unit: BambooHR JSON fixture → list[EmployeeRecord] correctly mapped`
- `Integration (mocked HTTP): full sync upserts new users, updates changed, deactivates removed`
- `Integration: incremental changed_since only touches changed rows`

#### 5.2 — Workday connector & unified-API seam

**What**: Workday REST connector; document the Merge adapter seam.

**Design**: Workday connector authenticates via OAuth 2.0 ISU credentials (per `standards.md`), maps the worker REST resource to `EmployeeRecord`. A `MergeHRISConnector` stub satisfies the same protocol for future 50+ provider coverage.

**Testing**:
- `Unit: Workday worker fixture → EmployeeRecord`
- `Integration (mocked): OAuth token fetched then employees listed`

#### 5.3 — Sync scheduling, integration config & encryption

**What**: Scheduled sync task; encrypted credential storage; `integrations` management endpoints.

**Design**: `integrations` table from Model 1. Credentials encrypted at rest (Fernet via `SECRET_KEY`-derived key) in `credentials_enc`. Beat task `sync_hris(org_id)` runs nightly; `GET/POST/DELETE /v1/integrations`.

**Testing**:
- `Unit: credentials round-trip encrypt/decrypt; ciphertext != plaintext`
- `Integration: nightly sync updates last_synced_at`
- `Integration: POST integration with bad provider → 422`

#### 5.4 — Calendar scheduling

**What**: One-click meeting booking for accepted match pairs across Google Calendar and MS Graph.

**Design**:
```python
class CalendarProvider(Protocol):
    async def find_slot(self, attendees: list[str], duration_min: int,
                        within_days: int) -> datetime | None: ...
    async def create_event(self, attendees, start, duration_min, title) -> str: ...
```
- Timezone-aware: candidate slots intersect attendees' working hours using each user's `timezone`. OAuth 2.0 + PKCE per `standards.md`. On `match:schedule`, find a slot, create the event, DM confirmation with the join link.

**Testing**:
- `Unit: find_slot intersects two users' working hours across timezones → slot inside both windows`
- `Integration (mocked Google API): create_event returns event id, confirmation DM sent`
- `Integration: no mutual slot in window → user offered manual scheduling fallback`

---

## Phase 6: Connection Network & Engagement Analytics

### Purpose
Turn raw participation into the analytics that incumbents lack: a persistent connection graph, daily engagement snapshots, and the participation dashboards HR needs. This phase builds the data substrate for isolation detection (Phase 8) and the AI narrative (Phase 8). Can be developed in parallel with Phase 5.

### Tasks

#### 6.1 — Connection graph maintenance

**What**: Maintain the undirected `connections` table from match meetings and recognitions.

**Design**: DDL from Model 1 Network section (`CHECK (user_a_id < user_b_id)`, unique per pair). A `record_interaction(org_id, u1, u2, source)` helper upserts: increments `interaction_count`, appends `source` to `interaction_types`, bumps `last_interaction_at`. Called from match `met` and recognition handlers.

**Testing**:
- `Unit: ordering normalises (b,a) → stored as (a,b)`
- `Integration: two interactions same pair → interaction_count=2, both sources recorded`
- `Integration: network-density query returns per-user connection + cross-team counts`

#### 6.2 — Engagement snapshots

**What**: Daily per-user `engagement_snapshots` aggregation.

**Design**: DDL from Model 1 Analytics section. Nightly Beat task computes matches_participated, recognitions_given/received, surveys_completed, celebrations_signed, connection_count, cross_team_connections for each consenting user; `isolation_risk_score` left null until Phase 8. Upsert keyed `(user_id, snapshot_date)`.

**Testing**:
- `Integration: snapshot reflects seeded activity counts`
- `Integration: re-running same day upserts (no duplicate)`
- `Integration: users without consent excluded`

#### 6.3 — Analytics API & admin dashboard

**What**: REST endpoints + Next.js dashboard for participation metrics and the network graph.

**Design**: Endpoints `GET /v1/analytics/participation`, `/v1/analytics/network`, `/v1/analytics/trends`. Frontend: Recharts trend cards + Cytoscape force-directed graph (node size = connection_count, colour = team, dashed/red = low connectivity). Manager role sees only own team (role-gated, OWASP A01 broken-access-control mitigation).

**Testing**:
- `Integration: participation endpoint returns match-accept/meeting-complete/recognition counts`
- `Integration: manager principal cannot read another team's data → 403`
- `E2E (Playwright): admin loads dashboard → trend chart + graph render with seeded data`

---

## Phase 7: Pulse Surveys & SCIM Provisioning

### Purpose
Add lightweight pulse measurement (own question set — no Gallup Q12 licensing) with trend dashboards, and enterprise SCIM provisioning so large customers auto-manage rosters. Together these move the product up-market.

### Tasks

#### 7.1 — Survey schema & delivery

**What**: Implement `pulse_surveys`, `survey_questions`, `survey_rounds`, `survey_responses`; deliver questions in chat.

**Design**: DDL from Model 1 Surveys section. Ship a default, independently-authored question set (avoid Q12 per `features.md` legal note). Beat task per survey cadence creates a `survey_round` and DMs Likert/NPS/text questions as interactive cards; responses upsert into `survey_responses` (`UNIQUE (round_id, question_id, user_id)`). Anonymous surveys store `user_id=NULL`.

**Testing**:
- `Integration: round send DMs all consenting users; response insert respects uniqueness`
- `Integration: anonymous survey → response rows have null user_id`
- `Integration: score outside 1..10 → rejected`

#### 7.2 — Survey trends API & dashboard

**What**: Per-question trend analysis and dashboard cards.

**Design**: `GET /v1/surveys/{id}/trends` returns avg score per question per round (Model 1 per-question rows enable this directly). Dashboard line charts show e.g. "team connection 3.8→3.4 over 6 rounds".

**Testing**:
- `Integration: 3 rounds seeded → trend series ordered by round`
- `E2E: trend chart renders declining series`

#### 7.3 — SCIM 2.0 server

**What**: RFC 7643/7644-compliant `/scim/v2/Users` and `/Groups`.

**Design**: FastAPI sub-router implementing GET/POST/PUT/PATCH/DELETE for Users and Groups per SCIM Core Schema. Bearer-token auth per org. SCIM `userName`→email, `active`→`is_active`, group membership→`team_members`. Pagination via SCIM `startIndex`/`count`; filtering on `userName eq`.

**Testing**:
- `Integration: POST /scim/v2/Users (SCIM payload) → user created, SCIM response shape with schemas/meta`
- `Integration: PATCH active:false → is_active false`
- `Integration: GET with filter userName eq "x@y.com" → single result`
- `Integration: missing bearer token → 401`

---

## Phase 8: AI-Native Layer — Matching Intelligence, Isolation Detection, Culture Narrative

### Purpose
Deliver the differentiators that justify "AI-native": affinity/goal/DEI-aware match scoring, isolation-risk detection over the connection graph and engagement snapshots, AI-generated watercooler prompts, and the monthly plain-language culture narrative for leadership. This is what no lightweight incumbent offers.

### Tasks

#### 8.1 — LLM gateway & prompt management

**What**: A single, retry-safe, cost-capturing LLM client with versioned prompts.

**Design**:
```python
class LLMClient:
    async def complete(self, prompt_name: str, vars: dict,
                       *, json_schema: dict | None = None) -> dict | str: ...
```
- Backed by `litellm`; prompts loaded from `prompts/*.jinja`; structured output validated against a JSON Schema; retries with backoff; token/cost recorded. Provider/model from settings.

**Testing**:
- `Unit (mocked LLM): complete renders template with vars, validates JSON output against schema`
- `Unit: malformed LLM JSON → one retry then ValidationError`

#### 8.2 — AI match scoring

**What**: Affinity/goal/DEI-aware pair scoring augmenting Phase 3's constraint solver.

**Design**: `ai_scorer.score_pairs(candidates, profiles) -> dict[pair, float]` builds a prompt from interests/skills/goals/department and returns a 0–1 affinity score; the engine uses scores as soft-preference weights in `program_type IN ('affinity','dei_targeted','mentoring')`. Falls back to rules-based if AI disabled/unavailable. Persists rationale to `ai_analyses` (`analysis_type='matching_optimization'`).

**Testing**:
- `Unit (mocked LLM): complementary skills → higher score than unrelated pair`
- `Integration: AI program produces pairs; AI failure → graceful fallback to rules-based`

#### 8.3 — Isolation-risk detection

**What**: Compute `isolation_risk_score` and flag at-risk employees.

**Design**: Nightly job computes a score per user from declining participation, low/declining `connection_count`, absence from recognition flows, and skipped surveys (signal list mirrors `data-model-suggestion-3` `engagement.isolation_flagged.signals`). Writes `engagement_snapshots.isolation_risk_score` and, above threshold (0.7), an `ai_analyses` row (`analysis_type='isolation_detection'`, `scope_type='user'`) plus a manager alert. An `engagement_events` audit row records the flag for GDPR traceability.

**Testing**:
- `Unit: user with zero matches + zero recognitions over window → score > 0.7`
- `Unit: highly active user → low score`
- `Integration: above-threshold user → ai_analyses row + manager alert; below → none`

#### 8.4 — Watercooler prompts & monthly culture narrative

**What**: AI watercooler prompts into channels; monthly leadership narrative.

**Design**: Watercooler Beat task generates a prompt (`watercooler_prompt_v1.jinja`) per channel cadence and posts it. Monthly task assembles org metrics (participation, recognition volume, survey averages, connection growth, isolation count) and generates a narrative with highlights/concerns/recommendations (`culture_narrative_v1.jinja`), stored in `ai_analyses` (`analysis_type='culture_narrative'`, `scope_type='org'`) and surfaced in the dashboard + emailed to admins.

**Testing**:
- `Integration (mocked LLM): monthly task → ai_analyses narrative row with highlights/concerns`
- `Integration: watercooler task posts generated prompt to configured channel`
- `Unit: narrative prompt includes the computed metric summary block`

---

## Phase 9: Enterprise Auth, GDPR & Security Hardening

### Purpose
Make the platform enterprise-deployable and compliant: admin SSO (OIDC + SAML), consent management, right-to-erasure, data minimisation, and an OWASP Top 10 pass. These are gating requirements for the HR/People-Ops buyers identified in research.

### Tasks

#### 9.1 — Admin SSO (OIDC + SAML 2.0)

**What**: SSO login for the admin portal.

**Design**: Authlib OIDC (Authorisation Code + PKCE, RFC 7636) for Google/Okta/Azure; SAML 2.0 for IdPs preferring it. Issues a short-lived JWT session (RFC 7519). Admin/manager roles mapped from IdP claims.

**Testing**:
- `Integration (mocked IdP): OIDC callback → JWT session issued, role mapped`
- `Integration: SAML assertion validated; tampered assertion → 401`
- `Unit: expired JWT → 401`

#### 9.2 — Consent, erasure & data minimisation (GDPR/CCPA)

**What**: Consent records, right-to-erasure, retention.

**Design**: Engagement processing gated on `consent_engagement_tracking` (lawful basis per GDPR; `standards.md`). `DELETE /v1/users/{id}/data` erases/anonymises personal data across recognitions (anonymise giver/receiver), survey responses, snapshots, connections, and `engagement_events` (crypto-erasure of the audit envelope). A retention task purges raw signals beyond the configured window.

**Testing**:
- `Integration: user without consent excluded from snapshots/matching`
- `Integration: erasure request anonymises recognitions, deletes snapshots, leaves aggregate counts intact`
- `Integration: consent revoke stops future engagement processing`

#### 9.3 — Security hardening (OWASP Top 10)

**What**: Multi-tenant access control, input validation, rate limiting, secrets handling.

**Design**: Enforce `org_id` scoping on every query (A01); parameterised queries via ORM (A03); object-level authorisation on all `/v1` routes (IDOR); per-IP/per-token rate limiting via Redis; security headers; webhook signature verification (Phases 2/5); encrypted credentials (Phase 5). Add a CI security job (`pip-audit`, `bandit`).

**Testing**:
- `Integration: user A cannot fetch user B's data in another org → 404/403`
- `Integration: IDOR attempt on /v1/recognition/{id} cross-org → 403`
- `Integration: rate limit exceeded → 429`
- `CI: bandit + pip-audit pass with no high findings`

---

## Phase 10: Packaging, MCP Server & Rewards (Backlog/Differentiators)

### Purpose
Round out the product with backlog items that extend reach: an MCP server exposing analytics to AI assistants, a peer-rewards catalogue with redemption, and production packaging/observability. Lowest priority; depends on the analytics and AI layers.

### Tasks

#### 10.1 — Rewards catalogue & redemption

**What**: Implement `reward_catalogue`, `reward_redemptions`; points spend flow.

**Design**: DDL from Model 1 Recognition section. Points balance derived from recognitions received; redemption decrements available points transactionally (`status: pending→fulfilled|cancelled`). Gift-card provider config in `provider_config JSONB`.

**Testing**:
- `Integration: redeem within balance → redemption row, points deducted`
- `Integration: redeem over balance → rejected, no row`
- `Integration: concurrent redemptions don't double-spend (row lock)`

#### 10.2 — MCP server for engagement analytics

**What**: MCP server exposing read-only culture metrics to AI assistants.

**Design**: MCP server (per `standards.md` modelcontextprotocol.io) exposing tools: `get_engagement_summary(org)`, `get_isolation_risks(org)`, `get_recognition_patterns(org)`, `get_culture_narrative(org, period)`. Read-only, auth via API token, org-scoped.

**Testing**:
- `Integration: MCP tool list advertises 4 tools`
- `Integration: get_isolation_risks returns flagged users for authorised org only`

#### 10.3 — Production packaging & observability

**What**: Production Compose/Helm-ready config, structured logging, metrics, OpenAPI publish.

**Design**: Structured JSON logging with correlation IDs; Prometheus metrics (match rounds, deliveries, LLM cost, task failures); healthcheck/readiness endpoints; `openapi.json` published and the frontend's typed client regenerated from it; production Docker Compose with Gunicorn workers + separate worker/beat services.

**Testing**:
- `Integration: /metrics exposes counters`
- `E2E: production compose up → full smoke flow (install → match round → recognition → dashboard) passes`
- `CI: generated frontend client compiles against published openapi.json`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (skeleton, config, core entities, Docker)   ── required by everything
    │
Phase 2: Chat Delivery Layer (Slack + Teams parity)             ── requires 1
    │
Phase 3: Matching Engine (core value proposition)               ── requires 2
    │
    ├── Phase 4: Recognition & Celebrations                     ── requires 2 (parallel with 5,6)
    ├── Phase 5: HRIS Sync & Calendar                           ── requires 1,3 (parallel with 4,6)
    └── Phase 6: Connection Network & Analytics                 ── requires 3,4 (parallel with 5)
         │
    ┌────┴───────────────────────────────────────────────┐
Phase 7: Pulse Surveys & SCIM        ── requires 2,6      │
Phase 8: AI Layer (matching/isolation/narrative)  ── requires 6 (uses 3,4,7 data)
         │
Phase 9: Enterprise Auth, GDPR & Security           ── requires 6 (hardens all prior)
         │
Phase 10: Packaging, MCP, Rewards (backlog)         ── requires 8,9
```

**Parallelism opportunities**
- Phases **4, 5, 6** can be developed concurrently once Phase 3 lands (4 and 6 share the chat layer; 5 is integration-only).
- Phase **7** (surveys) and the start of Phase **8** (LLM gateway 8.1) can begin alongside Phase 6.
- Frontend dashboard work (6.3, 7.2) can proceed in parallel with backend once the analytics endpoints are stubbed.

**Scope:** large (10 phases, multi-platform chat, multiple integrations, AI layer, enterprise compliance).

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`pytest`); new mocked-integration tests cover external dependencies.
3. Linting and formatting pass (`ruff check`, `ruff format --check`).
4. Type checking passes (`mypy --strict`).
5. Docker build succeeds and `docker compose up` reaches healthy state.
6. The phase's headline capability works end-to-end against a seeded org (verified by an integration or E2E test).
7. New configuration options are documented in `.env.example` and the README.
8. New/changed API endpoints appear in the generated `openapi.json` (OpenAPI 3.1) and, where consumed by the frontend, the typed client regenerates cleanly.
9. Alembic migrations are created, apply cleanly on an empty DB, and downgrade without error.
10. Security-relevant phases (2, 5, 7, 9) pass `bandit` and `pip-audit` with no high-severity findings, and webhook/SCIM endpoints verify signatures/tokens.
```
