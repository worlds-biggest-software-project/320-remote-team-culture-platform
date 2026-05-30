# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Remote Team Culture Platform · Created: 2026-05-25

## Philosophy

A remote team culture platform combines several loosely-coupled programs — matching, recognition, celebrations, surveys, watercooler prompts — each with different configuration shapes and interaction patterns. A hybrid approach keeps the core entities (users, teams, programs) relational while pushing program-specific configuration, interaction details, and AI outputs into JSONB columns. This reduces table count dramatically and aligns with how the platform is consumed: each program is a self-contained feature that the admin enables or disables.

The key insight is that a culture platform is more like a toolkit than a pipeline. Unlike a standup tool (fixed daily cycle) or a retro tool (fixed session flow), a culture platform is a collection of independent programs that share a user base and analytics layer. The JSONB approach lets each program type store its configuration and interaction data in whatever shape makes sense, without a lowest-common-denominator schema.

For analytics, the engagement data is kept in a dedicated relational table because cross-program queries ("show me users who aren't participating in anything") need fast, indexed access across all program types. The network graph is also relational because connection queries require join-based traversal.

**Best for:** Teams building a multi-program culture toolkit where adding new program types (watercooler prompts, team rituals, wellness challenges) should require zero migrations, and where the primary access pattern is "load this user's interactions with this program."

**Trade-offs:**
- **Pro:** New program types (watercooler, wellness, challenges) require zero schema changes
- **Pro:** Program configuration lives on the program row — no config tables
- **Pro:** Match pair interactions and survey responses inline — fewer tables
- **Pro:** 9 tables total — very low complexity
- **Con:** Per-pair analytics require JSONB unpacking
- **Con:** Survey question-level trend analysis requires JSONB array processing
- **Con:** Recognition leaderboards need GIN-indexed queries
- **Con:** Matching history queries are more complex with inline pair data

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 30414:2018 | Engagement metrics taxonomy |
| ISO/IEC 27701 | Privacy management for employee engagement data |
| RFC 6749 (OAuth 2.0) | Slack, Teams, HRIS integration auth |
| RFC 7643/7644 (SCIM 2.0) | Enterprise user provisioning |
| Slack Block Kit | Program interactions rendered from JSONB |
| MS Adaptive Cards v1.5 | Teams-native delivery from JSONB |
| GDPR | Consent tracking; right-to-erasure across JSONB data |
| ISO 8601 | All timestamps as TIMESTAMPTZ |

---

## Organisations & Users

```sql
CREATE TABLE organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    settings JSONB NOT NULL DEFAULT '{}',
    -- settings: {
    --   "billing_plan": "pro",
    --   "timezone": "America/New_York",
    --   "integrations": {
    --     "slack": {"bot_token_enc": "...", "team_id": "T123"},
    --     "teams": {"bot_id": "...", "tenant_id": "..."},
    --     "bamboohr": {"subdomain": "acme", "api_key_enc": "...", "sync_fields": ["birthday", "hire_date", "department"]},
    --     "google_calendar": {"credentials_enc": "..."},
    --     "zoom": {"api_key_enc": "..."}
    --   },
    --   "values": [
    --     {"id": "v1", "name": "Innovation", "emoji": "💡", "description": "Pushing boundaries"},
    --     {"id": "v2", "name": "Teamwork", "emoji": "🤝", "description": "Winning together"},
    --     {"id": "v3", "name": "Integrity", "emoji": "⭐", "description": "Doing the right thing"}
    --   ],
    --   "recognition": {
    --     "daily_give_limit": 5,
    --     "points_per_recognition": 1,
    --     "rewards_enabled": true,
    --     "reward_catalogue": [
    --       {"id": "r1", "name": "$25 Amazon Gift Card", "points_cost": 50, "type": "gift_card"},
    --       {"id": "r2", "name": "Half-day Friday", "points_cost": 100, "type": "time_off"}
    --     ]
    --   },
    --   "celebrations": {
    --     "birthday_enabled": true,
    --     "anniversary_enabled": true,
    --     "lead_days": 3,
    --     "default_channel": "C-celebrations"
    --   },
    --   "branding": {"primary_color": "#4A90D9", "logo_url": "..."}
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('admin', 'manager', 'member')),
    avatar_url TEXT,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    profile JSONB NOT NULL DEFAULT '{}',
    -- profile: {
    --   "department": "Engineering",
    --   "location": "San Francisco",
    --   "hire_date": "2024-03-15",
    --   "birthday": "1990-07-22",
    --   "job_title": "Senior Engineer",
    --   "interests": ["hiking", "board games", "cooking"],
    --   "matching_preferences": {
    --     "opt_in": true,
    --     "prefer_different_department": true,
    --     "availability_hours": [9, 17]
    --   },
    --   "external_ids": {
    --     "slack": "U12345",
    --     "teams": "alice@acme.onmicrosoft.com",
    --     "bamboohr": "12345"
    --   },
    --   "points_balance": 42,
    --   "consent_engagement_tracking": true,
    --   "consent_given_at": "2026-01-15T10:00:00Z"
    -- }
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_org ON users(org_id);
CREATE INDEX idx_users_profile ON users USING GIN (profile jsonb_path_ops);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    slug TEXT NOT NULL,
    description TEXT,
    members JSONB NOT NULL DEFAULT '[]',
    -- members: [
    --   {"user_id": "uuid", "role": "manager"},
    --   {"user_id": "uuid", "role": "member"}
    -- ]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_teams_org ON teams(org_id);
CREATE INDEX idx_teams_members ON teams USING GIN (members jsonb_path_ops);
```

---

## Culture Programs

```sql
CREATE TABLE programs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    program_type TEXT NOT NULL CHECK (program_type IN (
        'matching', 'recognition', 'celebration', 'survey',
        'watercooler', 'wellness_challenge', 'custom'
    )),
    name TEXT NOT NULL,
    config JSONB NOT NULL,
    -- Matching config:
    -- {
    --   "group_size": 2,
    --   "cadence": "biweekly",
    --   "channel_id": "C123",
    --   "channel_platform": "slack",
    --   "matching_type": "cross_department",
    --   "rules": {
    --     "avoid_same_department": true,
    --     "avoid_repeat_within_rounds": 4,
    --     "dei_balance": {"target": "gender_balance"},
    --     "prefer_different_location": true
    --   },
    --   "intro_message": "Hey! You've been matched for a virtual coffee ☕"
    -- }
    --
    -- Survey config:
    -- {
    --   "cadence": "weekly",
    --   "is_anonymous": true,
    --   "questions": [
    --     {"id": "q1", "text": "How connected do you feel to your team?", "type": "likert"},
    --     {"id": "q2", "text": "How would you rate your work-life balance?", "type": "likert"},
    --     {"id": "q3", "text": "Any feedback you'd like to share?", "type": "text"}
    --   ],
    --   "channel_id": "C-pulse"
    -- }
    --
    -- Watercooler config:
    -- {
    --   "cadence": "daily",
    --   "channel_id": "C-watercooler",
    --   "prompt_source": "ai",
    --   "topics": ["weekend", "hobbies", "unpopular_opinions", "travel"]
    -- }
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_programs_org ON programs(org_id, program_type);
```

---

## Interactions

```sql
CREATE TABLE interactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    program_id UUID NOT NULL REFERENCES programs(id) ON DELETE CASCADE,
    interaction_type TEXT NOT NULL CHECK (interaction_type IN (
        'match_round', 'recognition', 'celebration', 'survey_round',
        'watercooler_prompt', 'wellness_entry', 'custom'
    )),
    round_number INT,
    data JSONB NOT NULL,
    -- Match round data:
    -- {
    --   "pairs": [
    --     {
    --       "id": "uuid",
    --       "members": [
    --         {"user_id": "uuid", "accepted": true, "responded_at": "2026-05-25T10:00:00Z"},
    --         {"user_id": "uuid", "accepted": true, "responded_at": "2026-05-25T10:30:00Z"}
    --       ],
    --       "status": "met",
    --       "meeting_scheduled_at": "2026-05-27T14:00:00Z",
    --       "meeting_completed": true,
    --       "feedback": {"score": 4, "comment": "Great conversation!"}
    --     },
    --     ...
    --   ],
    --   "participation": {"total": 20, "matched": 18, "met": 14}
    -- }
    --
    -- Recognition data:
    -- {
    --   "giver_id": "uuid",
    --   "receiver_id": "uuid",
    --   "message": "Thanks for helping debug the production issue last night!",
    --   "value_id": "v2",
    --   "value_name": "Teamwork",
    --   "points": 1,
    --   "channel_id": "C-general",
    --   "reactions": [
    --     {"user_id": "uuid", "emoji": "👏", "at": "2026-05-25T11:00:00Z"}
    --   ]
    -- }
    --
    -- Celebration data:
    -- {
    --   "celebrant_id": "uuid",
    --   "type": "birthday",
    --   "date": "2026-05-25",
    --   "signers": [
    --     {"user_id": "uuid", "message": "Happy birthday! 🎂", "signed_at": "2026-05-24T15:00:00Z"},
    --     {"user_id": "uuid", "message": "Have a great day!", "signed_at": "2026-05-24T16:00:00Z"}
    --   ],
    --   "card_posted": true,
    --   "posted_at": "2026-05-25T09:00:00Z"
    -- }
    --
    -- Survey round data:
    -- {
    --   "round_number": 12,
    --   "sent_at": "2026-05-25T09:00:00Z",
    --   "closes_at": "2026-05-26T09:00:00Z",
    --   "responses": [
    --     {
    --       "user_id": "uuid",
    --       "answers": [
    --         {"question_id": "q1", "score": 4},
    --         {"question_id": "q2", "score": 3},
    --         {"question_id": "q3", "text": "Would love more cross-team events"}
    --       ],
    --       "submitted_at": "2026-05-25T10:15:00Z"
    --     }
    --   ],
    --   "summary": {
    --     "response_rate": 0.85,
    --     "avg_scores": {"q1": 3.8, "q2": 3.4},
    --     "text_themes": ["cross-team", "workload"]
    --   }
    -- }
    ai JSONB,
    -- ai: {
    --   "watercooler_prompt_generated": "What's a skill you'd love to learn that has nothing to do with your job?",
    --   "sentiment": "positive",
    --   "engagement_signal": "high",
    --   "model_version": "gpt-4o-2026-05"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_interactions_program ON interactions(program_id, created_at DESC);
CREATE INDEX idx_interactions_type ON interactions(interaction_type, created_at DESC);
CREATE INDEX idx_interactions_data ON interactions USING GIN (data jsonb_path_ops);
```

---

## Connection Network

```sql
CREATE TABLE connections (
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_a_id UUID NOT NULL REFERENCES users(id),
    user_b_id UUID NOT NULL REFERENCES users(id),
    interaction_count INT NOT NULL DEFAULT 1,
    last_interaction_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    sources TEXT[] NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (org_id, user_a_id, user_b_id),
    CHECK (user_a_id < user_b_id)
);

CREATE INDEX idx_connections_a ON connections(user_a_id);
CREATE INDEX idx_connections_b ON connections(user_b_id);
```

---

## Engagement Analytics

```sql
CREATE TABLE engagement_snapshots (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    snapshot_date DATE NOT NULL,
    metrics JSONB NOT NULL,
    -- metrics: {
    --   "matches_participated": 2,
    --   "recognitions_given": 3,
    --   "recognitions_received": 1,
    --   "surveys_completed": 1,
    --   "celebrations_signed": 2,
    --   "watercooler_participated": 4,
    --   "connection_count": 18,
    --   "cross_team_connections": 7,
    --   "points_balance": 42,
    --   "isolation_risk_score": 0.15,
    --   "engagement_trend": "stable"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, snapshot_date)
);

CREATE INDEX idx_engagement_date ON engagement_snapshots(snapshot_date DESC);
```

---

## AI Culture Narratives

```sql
CREATE TABLE culture_reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    report_type TEXT NOT NULL CHECK (report_type IN (
        'monthly_narrative', 'isolation_alert', 'recognition_pattern',
        'survey_insight', 'matching_optimization', 'engagement_anomaly'
    )),
    scope_type TEXT NOT NULL CHECK (scope_type IN ('org', 'team', 'user')),
    scope_id UUID NOT NULL,
    content JSONB NOT NULL,
    -- monthly_narrative:
    -- {
    --   "narrative": "May saw a 15% increase in cross-team connections driven by the new mentoring program...",
    --   "highlights": ["Recognition up 23% month-over-month", "New matching program reached 85% participation"],
    --   "concerns": ["Engineering team survey scores declined for second consecutive month"],
    --   "recommendations": ["Consider targeted watercooler prompts for engineering"],
    --   "metrics_summary": {"avg_engagement": 3.8, "isolation_risk_users": 3, "recognition_volume": 147}
    -- }
    model_version TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reports_org ON culture_reports(org_id, report_type, created_at DESC);
```

---

## Example Queries

### All matching interactions for a user (who have they been paired with?)

```sql
SELECT i.created_at AS round_date,
       pair->>'status' AS pair_status,
       pair->'feedback'->>'score' AS feedback_score,
       jsonb_path_query_array(pair->'members', '$[*].user_id') AS partner_ids
FROM interactions i,
     jsonb_array_elements(i.data->'pairs') AS pair
WHERE i.interaction_type = 'match_round'
  AND pair->'members' @> '[{"user_id": "target-user-uuid"}]'
ORDER BY i.created_at DESC;
```

### Recognition leaderboard

```sql
SELECT (i.data->>'receiver_id')::UUID AS receiver_id,
       u.name,
       COUNT(*) AS times_recognized,
       SUM((i.data->>'points')::INT) AS total_points
FROM interactions i
JOIN users u ON u.id = (i.data->>'receiver_id')::UUID
WHERE i.interaction_type = 'recognition'
  AND i.program_id IN (SELECT id FROM programs WHERE org_id = 'org-uuid')
  AND i.created_at >= CURRENT_DATE - 30
GROUP BY i.data->>'receiver_id', u.name
ORDER BY total_points DESC
LIMIT 20;
```

### Survey trend over time

```sql
SELECT i.created_at::DATE AS survey_date,
       (i.data->'summary'->'avg_scores'->>'q1')::REAL AS avg_team_connection,
       (i.data->'summary'->>'response_rate')::REAL AS response_rate
FROM interactions i
WHERE i.program_id = 'survey-program-uuid'
  AND i.interaction_type = 'survey_round'
ORDER BY i.created_at;
```

### Isolation-risk users

```sql
SELECT u.name, u.profile->>'department' AS department,
       es.metrics->>'isolation_risk_score' AS risk_score,
       es.metrics->>'connection_count' AS connections,
       es.metrics->>'matches_participated' AS matches
FROM engagement_snapshots es
JOIN users u ON u.id = es.user_id
WHERE es.snapshot_date = CURRENT_DATE - 1
  AND (es.metrics->>'isolation_risk_score')::REAL > 0.7
ORDER BY (es.metrics->>'isolation_risk_score')::REAL DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Users | 3 | organisations (with values, rewards, integrations), users (with profile), teams |
| Programs | 1 | programs (config varies by program_type as JSONB) |
| Interactions | 1 | interactions (data varies by interaction_type as JSONB) |
| Network | 1 | connections (undirected edges) |
| Analytics | 1 | engagement_snapshots (metrics as JSONB) |
| AI | 1 | culture_reports |
| **Total** | **8** | Versus 22 in normalized model |

---

## Key Design Decisions

1. **Programs as polymorphic rows** — `programs` stores all program types (matching, recognition, celebration, survey, watercooler) with type-specific configuration in `config` JSONB. Adding a new program type (wellness challenges, team rituals) means adding a new `program_type` enum value, not a new table.

2. **Interactions as polymorphic rows** — `interactions` stores all program interactions with type-specific data in `data` JSONB. A match round stores pairs with members and feedback. A recognition stores giver, receiver, message, and value. A celebration stores signers. This unified table enables "show me all engagement activity for user X" in one query.

3. **Company values on organisation** — `organisations.settings.values` stores value definitions as JSONB. Recognitions reference values by ID. This avoids a separate values table while keeping the value catalogue centralized.

4. **User profile as JSONB** — `users.profile` stores HRIS-synced fields (department, location, hire_date, birthday), matching preferences, external platform IDs, and points balance. This allows HRIS sync to add new fields without migrations.

5. **Connections kept relational** — Despite the hybrid approach, `connections` remains a relational table because network graph queries (isolation detection, cross-team density) require efficient join-based traversal. The undirected-edge pattern (`user_a_id < user_b_id`) prevents duplicates.

6. **Engagement snapshots with JSONB metrics** — `engagement_snapshots` stores daily per-user engagement metrics as JSONB. This allows adding new metrics (wellness participation, watercooler engagement) without schema changes. The isolation_risk_score is extracted for indexed queries.

7. **Reward catalogue on organisation** — `organisations.settings.recognition.reward_catalogue` stores available rewards as JSONB. Points balance lives on `users.profile.points_balance`. Redemptions are recorded as recognition-type interactions with `interaction_type = 'reward_redemption'`.

8. **Survey responses inline on interaction** — `interactions.data.responses` stores all survey round responses as a JSONB array. This keeps the round self-contained and avoids a separate responses table. The trade-off: per-question trend analysis requires JSONB array processing, but the `data->'summary'->>'avg_scores'` pre-computed summary mitigates this for dashboards.
