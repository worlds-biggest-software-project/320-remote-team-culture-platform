# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Remote Team Culture Platform · Created: 2026-05-25

## Philosophy

A remote team culture platform manages several interlocking programs: matching rounds pair colleagues for virtual coffees, recognition moments let peers give kudos tied to company values, celebrations mark birthdays and work anniversaries, pulse surveys measure engagement, and analytics track relationship density across the organisation. A normalized relational model gives each program its own set of tables with explicit foreign keys, enabling database-level enforcement of matching constraints (no repeat pairings within N rounds), recognition budget limits (daily giving caps), and survey response integrity.

This mirrors how People Ops teams think about culture programs: a matching program has rounds, rounds produce pairs, pairs result in meetings (or not). Recognition has a giver, a receiver, a value, and optional points. Celebrations have a person, a date, and a card with signers. Surveys have questions, respondents, and scores. Each maps cleanly to a table.

The relationship graph — who has connected with whom, how often, and how recently — is modeled as an explicit `connections` table derived from matching pairs and recognition interactions, enabling network density queries and isolation-risk detection via SQL.

**Best for:** Teams building a multi-program culture platform where matching fairness constraints need enforcement, where recognition budgets and reward redemptions require transactional integrity, and where network analysis (isolation detection, relationship density) is a first-class feature.

**Trade-offs:**
- **Pro:** Database-enforced matching constraints (no duplicate pairs within window)
- **Pro:** Recognition with explicit points enables budget tracking and leaderboards
- **Pro:** Connections table enables network graph queries and isolation detection
- **Pro:** Survey responses as typed rows enable per-question trend analysis
- **Pro:** Celebration cards with signers as rows enable nudge workflows
- **Con:** 24 tables — moderate-to-high complexity
- **Con:** Network graph queries require self-joins on connections table
- **Con:** Matching configuration varies by program type but uses shared columns
- **Con:** High join count for "full engagement dashboard" view

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 30414:2018 | HR reporting taxonomy for engagement metrics |
| ISO/IEC 27001 | Security controls for sensitive employee data |
| ISO/IEC 27701 | Privacy management for engagement/wellness data |
| RFC 6749 (OAuth 2.0) | Slack, Teams, HRIS integration auth |
| RFC 7643/7644 (SCIM 2.0) | Enterprise user provisioning |
| OIDC / SAML 2.0 | Enterprise SSO |
| Slack Block Kit | Match notifications, recognition messages |
| MS Adaptive Cards v1.5 | Teams-native delivery |
| RFC 4791 (CalDAV) | Calendar integration for meeting scheduling |
| GDPR | Consent for engagement tracking; right-to-erasure |
| OpenAPI 3.1 | REST API specification |

---

## Organisations, Users & Teams

```sql
CREATE TABLE organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    billing_plan TEXT NOT NULL DEFAULT 'free',
    timezone TEXT NOT NULL DEFAULT 'UTC',
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
    department TEXT,
    location TEXT,
    hire_date DATE,
    birthday DATE,
    job_title TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    consent_engagement_tracking BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_org ON users(org_id);
CREATE INDEX idx_users_birthday ON users(org_id, birthday) WHERE birthday IS NOT NULL;
CREATE INDEX idx_users_hire_date ON users(org_id, hire_date) WHERE hire_date IS NOT NULL;

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    slug TEXT NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE TABLE team_members (
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('manager', 'member')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, user_id)
);
```

---

## Matching Programs & Rounds

```sql
CREATE TABLE matching_programs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    program_type TEXT NOT NULL DEFAULT 'random' CHECK (program_type IN (
        'random', 'cross_department', 'onboarding_buddy', 'mentoring',
        'affinity', 'dei_targeted'
    )),
    group_size INT NOT NULL DEFAULT 2,
    cadence TEXT NOT NULL DEFAULT 'weekly' CHECK (cadence IN (
        'weekly', 'biweekly', 'monthly'
    )),
    channel_id TEXT,
    channel_platform TEXT CHECK (channel_platform IN ('slack', 'teams')),
    matching_rules JSONB NOT NULL DEFAULT '{}',
    -- matching_rules: {
    --   "avoid_same_department": true,
    --   "avoid_same_location": false,
    --   "avoid_repeat_within_rounds": 4,
    --   "prefer_different_tenure": true,
    --   "dei_balance": {"target": "gender_balance", "tolerance": 0.2}
    -- }
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_programs_org ON matching_programs(org_id);

CREATE TABLE matching_rounds (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    program_id UUID NOT NULL REFERENCES matching_programs(id) ON DELETE CASCADE,
    round_number INT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'matched', 'notified', 'completed'
    )),
    matched_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (program_id, round_number)
);

CREATE INDEX idx_rounds_program ON matching_rounds(program_id, round_number DESC);

CREATE TABLE match_pairs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    round_id UUID NOT NULL REFERENCES matching_rounds(id) ON DELETE CASCADE,
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'accepted', 'declined', 'met', 'no_response'
    )),
    meeting_scheduled_at TIMESTAMPTZ,
    meeting_completed BOOLEAN,
    feedback_score INT CHECK (feedback_score BETWEEN 1 AND 5),
    feedback_comment TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE match_pair_members (
    pair_id UUID NOT NULL REFERENCES match_pairs(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    accepted BOOLEAN,
    responded_at TIMESTAMPTZ,
    PRIMARY KEY (pair_id, user_id)
);

CREATE INDEX idx_pair_members_user ON match_pair_members(user_id);
```

---

## Recognition

```sql
CREATE TABLE company_values (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT,
    emoji TEXT,
    sort_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, name)
);

CREATE TABLE recognitions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    giver_id UUID NOT NULL REFERENCES users(id),
    receiver_id UUID NOT NULL REFERENCES users(id),
    value_id UUID REFERENCES company_values(id),
    message TEXT NOT NULL,
    points INT NOT NULL DEFAULT 1,
    channel_id TEXT,
    channel_platform TEXT CHECK (channel_platform IN ('slack', 'teams')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (giver_id != receiver_id)
);

CREATE INDEX idx_recognitions_receiver ON recognitions(receiver_id, created_at DESC);
CREATE INDEX idx_recognitions_giver ON recognitions(giver_id, created_at DESC);
CREATE INDEX idx_recognitions_org ON recognitions(org_id, created_at DESC);
CREATE INDEX idx_recognitions_value ON recognitions(value_id);

CREATE TABLE recognition_reactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recognition_id UUID NOT NULL REFERENCES recognitions(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    reaction TEXT NOT NULL DEFAULT '👏',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (recognition_id, user_id)
);

CREATE TABLE reward_catalogue (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT,
    points_cost INT NOT NULL,
    reward_type TEXT NOT NULL CHECK (reward_type IN (
        'gift_card', 'experience', 'time_off', 'custom'
    )),
    provider_config JSONB,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE reward_redemptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    reward_id UUID NOT NULL REFERENCES reward_catalogue(id),
    points_spent INT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'fulfilled', 'cancelled'
    )),
    fulfilled_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_redemptions_user ON reward_redemptions(user_id, created_at DESC);
```

---

## Celebrations

```sql
CREATE TABLE celebrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    celebrant_id UUID NOT NULL REFERENCES users(id),
    celebration_type TEXT NOT NULL CHECK (celebration_type IN (
        'birthday', 'work_anniversary', 'promotion', 'new_hire', 'custom'
    )),
    celebration_date DATE NOT NULL,
    years INT,
    channel_id TEXT,
    channel_platform TEXT CHECK (channel_platform IN ('slack', 'teams')),
    card_message TEXT,
    is_posted BOOLEAN NOT NULL DEFAULT FALSE,
    posted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_celebrations_org ON celebrations(org_id, celebration_date);
CREATE INDEX idx_celebrations_upcoming ON celebrations(org_id, celebration_date)
    WHERE NOT is_posted;

CREATE TABLE celebration_signers (
    celebration_id UUID NOT NULL REFERENCES celebrations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    message TEXT NOT NULL,
    signed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (celebration_id, user_id)
);
```

---

## Pulse Surveys

```sql
CREATE TABLE pulse_surveys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    cadence TEXT NOT NULL DEFAULT 'weekly' CHECK (cadence IN (
        'weekly', 'biweekly', 'monthly', 'quarterly', 'one_time'
    )),
    is_anonymous BOOLEAN NOT NULL DEFAULT TRUE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE survey_questions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id UUID NOT NULL REFERENCES pulse_surveys(id) ON DELETE CASCADE,
    question_text TEXT NOT NULL,
    question_type TEXT NOT NULL DEFAULT 'likert' CHECK (question_type IN (
        'likert', 'nps', 'text', 'multiple_choice'
    )),
    options TEXT[],
    sort_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_survey_questions ON survey_questions(survey_id, sort_order);

CREATE TABLE survey_rounds (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id UUID NOT NULL REFERENCES pulse_surveys(id) ON DELETE CASCADE,
    sent_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    closes_at TIMESTAMPTZ,
    response_count INT NOT NULL DEFAULT 0,
    total_recipients INT NOT NULL DEFAULT 0
);

CREATE TABLE survey_responses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    round_id UUID NOT NULL REFERENCES survey_rounds(id) ON DELETE CASCADE,
    question_id UUID NOT NULL REFERENCES survey_questions(id),
    user_id UUID REFERENCES users(id),
    score INT CHECK (score BETWEEN 1 AND 10),
    text_response TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (round_id, question_id, user_id)
);

CREATE INDEX idx_survey_responses ON survey_responses(round_id);
```

---

## Connection Network & Engagement Analytics

```sql
CREATE TABLE connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_a_id UUID NOT NULL REFERENCES users(id),
    user_b_id UUID NOT NULL REFERENCES users(id),
    interaction_count INT NOT NULL DEFAULT 1,
    last_interaction_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    interaction_types TEXT[] NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, user_a_id, user_b_id),
    CHECK (user_a_id < user_b_id)
);

CREATE INDEX idx_connections_user_a ON connections(user_a_id);
CREATE INDEX idx_connections_user_b ON connections(user_b_id);
CREATE INDEX idx_connections_org ON connections(org_id);

CREATE TABLE engagement_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    snapshot_date DATE NOT NULL,
    matches_participated INT NOT NULL DEFAULT 0,
    recognitions_given INT NOT NULL DEFAULT 0,
    recognitions_received INT NOT NULL DEFAULT 0,
    surveys_completed INT NOT NULL DEFAULT 0,
    celebrations_signed INT NOT NULL DEFAULT 0,
    connection_count INT NOT NULL DEFAULT 0,
    cross_team_connections INT NOT NULL DEFAULT 0,
    isolation_risk_score REAL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, snapshot_date)
);

CREATE INDEX idx_engagement_user ON engagement_snapshots(user_id, snapshot_date DESC);
CREATE INDEX idx_engagement_isolation ON engagement_snapshots(org_id, isolation_risk_score DESC)
    WHERE isolation_risk_score IS NOT NULL;
```

---

## HRIS Sync & Integrations

```sql
CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    provider TEXT NOT NULL CHECK (provider IN (
        'slack', 'teams', 'bamboohr', 'workday', 'google_calendar',
        'outlook', 'zoom'
    )),
    status TEXT NOT NULL DEFAULT 'connected',
    credentials_enc TEXT,
    config JSONB NOT NULL DEFAULT '{}',
    last_synced_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, provider)
);
```

---

## AI Analyses

```sql
CREATE TABLE ai_analyses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL CHECK (analysis_type IN (
        'culture_narrative', 'isolation_detection', 'engagement_anomaly',
        'matching_optimization', 'recognition_pattern', 'survey_insight'
    )),
    scope_type TEXT NOT NULL CHECK (scope_type IN ('org', 'team', 'user')),
    scope_id UUID NOT NULL,
    content TEXT NOT NULL,
    score REAL,
    details JSONB,
    model_version TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_org ON ai_analyses(org_id, analysis_type, created_at DESC);
```

---

## Example Queries

### Network density for a team

```sql
SELECT u.name,
       COUNT(DISTINCT c.id) AS total_connections,
       COUNT(DISTINCT c.id) FILTER (
           WHERE tm2.team_id != tm.team_id OR tm2.team_id IS NULL
       ) AS cross_team_connections
FROM users u
JOIN team_members tm ON tm.user_id = u.id AND tm.team_id = 'team-uuid'
LEFT JOIN connections c ON (c.user_a_id = u.id OR c.user_b_id = u.id)
LEFT JOIN team_members tm2 ON tm2.user_id = CASE
    WHEN c.user_a_id = u.id THEN c.user_b_id ELSE c.user_a_id END
GROUP BY u.id, u.name
ORDER BY total_connections;
```

### Isolation-risk users (low engagement)

```sql
SELECT u.name, u.department, u.hire_date,
       es.connection_count, es.matches_participated,
       es.recognitions_received, es.isolation_risk_score
FROM engagement_snapshots es
JOIN users u ON u.id = es.user_id
WHERE es.org_id = 'org-uuid'
  AND es.snapshot_date = CURRENT_DATE - 1
  AND es.isolation_risk_score > 0.7
ORDER BY es.isolation_risk_score DESC;
```

### Recognition leaderboard with values

```sql
SELECT u.name, cv.name AS value, cv.emoji,
       COUNT(*) AS times_recognized,
       SUM(r.points) AS total_points
FROM recognitions r
JOIN users u ON u.id = r.receiver_id
LEFT JOIN company_values cv ON cv.id = r.value_id
WHERE r.org_id = 'org-uuid'
  AND r.created_at >= CURRENT_DATE - 30
GROUP BY u.id, u.name, cv.name, cv.emoji
ORDER BY total_points DESC
LIMIT 20;
```

### Matching history — avoid repeat pairings

```sql
SELECT mpm.user_id AS partner_id, COUNT(*) AS times_paired,
       MAX(mr.matched_at) AS last_paired_at
FROM match_pair_members mpm
JOIN match_pairs mp ON mp.id = mpm.pair_id
JOIN matching_rounds mr ON mr.id = mp.round_id
WHERE mpm.pair_id IN (
    SELECT pair_id FROM match_pair_members WHERE user_id = 'user-uuid'
)
AND mpm.user_id != 'user-uuid'
GROUP BY mpm.user_id
ORDER BY last_paired_at DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Users | 4 | organisations, users, teams, team_members |
| Matching | 4 | matching_programs, matching_rounds, match_pairs, match_pair_members |
| Recognition | 4 | company_values, recognitions, recognition_reactions, reward_catalogue + reward_redemptions |
| Celebrations | 2 | celebrations, celebration_signers |
| Surveys | 4 | pulse_surveys, survey_questions, survey_rounds, survey_responses |
| Network & Analytics | 2 | connections, engagement_snapshots |
| Integrations | 1 | integrations |
| AI | 1 | ai_analyses |
| **Total** | **22** | |

---

## Key Design Decisions

1. **Match pairs with member junction** — `match_pairs` stores the pair status while `match_pair_members` stores individual responses. This supports both 1:1 pairings and small groups (3-4 people) and tracks individual acceptance independently.

2. **Connections as undirected edges** — `connections` stores one row per pair with `CHECK (user_a_id < user_b_id)` to prevent duplicates. `interaction_count` and `interaction_types` accumulate over time, enabling "this pair has met 5 times via matching and recognition" queries.

3. **Engagement snapshots for isolation detection** — `engagement_snapshots` stores a daily materialized view of each user's engagement metrics. The `isolation_risk_score` is computed from declining participation, low connection count, and absence from recognition flows. This enables managers to see at-risk employees without querying across multiple program tables.

4. **Recognition with company values** — `recognitions` links to `company_values`, enabling "how often is 'innovation' being recognized vs 'teamwork'?" analytics that align recognition patterns with stated cultural priorities.

5. **Celebration signers as rows** — `celebration_signers` stores each teammate's contribution to a digital birthday/anniversary card. This enables nudge workflows ("4 of 8 teammates have signed — remind the others").

6. **Survey responses per question** — `survey_responses` stores one row per respondent per question per round. This enables per-question trend analysis ("trust in leadership declined from 4.2 to 3.8 over 6 months") and supports mixed question types (Likert, NPS, text).

7. **Matching rules as JSONB** — `matching_programs.matching_rules` stores the configuration for the matching algorithm (avoid same department, DEI balancing, repeat avoidance window). This is the one area where JSONB flexibility is used in the normalized model, because matching rule schemas vary by program type.

8. **HRIS fields on users** — `users` includes `department`, `location`, `hire_date`, `birthday`, and `job_title` synced from BambooHR/Workday. These are first-class columns rather than JSONB because they drive matching rules, celebration dates, and engagement analytics directly.
