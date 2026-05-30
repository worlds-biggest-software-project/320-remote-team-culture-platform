# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Remote Team Culture Platform · Created: 2026-05-25

## Philosophy

A remote team culture platform is fundamentally a system that observes, records, and analyzes human interactions. A matching round produces pairing events; recognitions are social events; celebrations trigger signing events; surveys produce response events; watercooler prompts generate participation signals. The AI layer consumes all of these events to compute engagement scores, detect isolation risk, identify recognition patterns, and generate culture narratives. An event-sourced model makes every interaction an immutable event, creating a unified engagement stream that the AI and analytics layers project into dashboards and alerts.

This architecture is particularly natural for culture platforms because: (1) engagement analytics are temporal by definition — "is this person becoming more or less engaged over time?" is answered by event stream analysis; (2) isolation detection requires tracking the absence of events (no matches accepted, no recognitions given, no survey responses) which event sourcing handles via gap analysis; (3) GDPR requires knowing exactly what engagement data was collected about an employee and when — the event store IS the audit log; (4) the AI culture narrative ("Engineering team's cross-team connections increased 23% this month") is a projection over engagement events.

The connection network graph — which is the most analytically powerful view — becomes a materialized projection that rebuilds whenever match meetings, recognitions, or survey interactions create new connections between people.

**Best for:** Teams building an analytics-first culture platform where engagement trend detection, isolation risk flagging, and AI-generated culture narratives are the primary differentiators, and where GDPR audit trails for employee engagement data are a regulatory requirement.

**Trade-offs:**
- **Pro:** Unified engagement stream for AI — all interaction types in one queryable store
- **Pro:** Isolation detection via event absence analysis
- **Pro:** Complete GDPR audit trail — every engagement data point is an immutable event
- **Pro:** Culture narratives are projections over events — computable on demand for any period
- **Pro:** Network graph rebuilds from events — no stale connection data
- **Con:** "Show the recognition leaderboard" requires a read model (not a direct query)
- **Con:** Event volume grows with org size × program count × interaction frequency
- **Con:** Event schema versioning as new program types are added
- **Con:** 15 tables including read models

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 30414:2018 | Engagement metrics taxonomy for culture reports |
| ISO/IEC 27701 | Privacy management — events are the personal data audit trail |
| CloudEvents 1.0 | Event envelope structure |
| RFC 6749 (OAuth 2.0) | Integration auth events |
| RFC 7643/7644 (SCIM 2.0) | User provisioning events |
| Slack Block Kit | Read models rendered as Block Kit messages |
| MS Adaptive Cards v1.5 | Read models rendered as Adaptive Cards |
| GDPR | Event immutability with crypto-erasure for right-to-deletion |
| ISO 8601 | All timestamps as TIMESTAMPTZ |

---

## Event Store

```sql
CREATE TABLE event_store (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type TEXT NOT NULL,
    stream_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    version BIGINT NOT NULL,
    data JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    -- metadata: {
    --   "actor_id": "uuid",
    --   "org_id": "uuid",
    --   "program_id": "uuid",
    --   "correlation_id": "uuid",
    --   "ce_source": "culture-platform/matching",
    --   "ce_specversion": "1.0",
    --   "platform": "slack",
    --   "gdpr_consent_basis": "legitimate_interest"
    -- }
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, version)
) PARTITION BY RANGE (occurred_at);

CREATE TABLE event_store_2026_q1 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE event_store_2026_q2 PARTITION OF event_store
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE event_store_2026_q3 PARTITION OF event_store
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE event_store_2026_q4 PARTITION OF event_store
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id, version);
CREATE INDEX idx_events_type ON event_store(event_type, occurred_at);
CREATE INDEX idx_events_actor ON event_store((metadata->>'actor_id'), occurred_at);
CREATE INDEX idx_events_org ON event_store((metadata->>'org_id'), occurred_at);
CREATE INDEX idx_events_program ON event_store((metadata->>'program_id'), occurred_at);
```

---

## Event Type Registry

```sql
CREATE TABLE event_types (
    event_type TEXT PRIMARY KEY,
    stream_type TEXT NOT NULL,
    description TEXT NOT NULL,
    schema_version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Matching events
INSERT INTO event_types (event_type, stream_type, description) VALUES
('match.round_created',      'program', 'Matching round initiated for a program'),
('match.pair_created',       'program', 'Two (or more) users paired for connection'),
('match.pair_accepted',      'program', 'User accepted a match pairing'),
('match.pair_declined',      'program', 'User declined a match pairing'),
('match.meeting_scheduled',  'program', 'Meeting time booked for a match pair'),
('match.meeting_completed',  'program', 'Matched pair confirmed they met'),
('match.feedback_submitted', 'program', 'User submitted feedback on a match meeting'),
('match.no_response',        'program', 'Match pair expired without response'),

-- Recognition events
('recognition.given',        'program', 'Peer recognition given with message and optional value'),
('recognition.reacted',      'program', 'Someone reacted to a recognition'),
('recognition.points_earned','user', 'User earned recognition points'),
('recognition.reward_redeemed','user', 'User redeemed points for a reward'),

-- Celebration events
('celebration.upcoming',     'program', 'Celebration date approaching (birthday, anniversary)'),
('celebration.card_created',  'program', 'Digital celebration card created'),
('celebration.card_signed',   'program', 'Teammate signed a celebration card'),
('celebration.card_posted',   'program', 'Celebration card posted to channel'),

-- Survey events
('survey.round_sent',        'program', 'Pulse survey round sent to team'),
('survey.response_submitted','program', 'User submitted survey response'),
('survey.round_closed',      'program', 'Survey round closed and summary generated'),

-- Watercooler events
('watercooler.prompt_posted', 'program', 'Conversation starter posted to channel'),
('watercooler.response',      'program', 'User responded to a watercooler prompt'),

-- Engagement events (AI-generated)
('engagement.snapshot_taken', 'user', 'Daily engagement metrics snapshot computed'),
('engagement.isolation_flagged','user', 'User flagged as isolation risk by AI'),
('engagement.isolation_cleared','user', 'Isolation risk cleared — participation resumed'),
('engagement.anomaly_detected','user', 'Engagement anomaly detected (sudden drop/spike)'),

-- AI culture events
('ai.narrative_generated',   'org', 'Monthly culture narrative generated'),
('ai.matching_optimized',    'program', 'AI optimized matching rules based on outcomes'),
('ai.recognition_pattern',   'org', 'AI detected recognition pattern (value skew, clique)'),
('ai.survey_insight',        'org', 'AI generated insight from survey trends'),

-- Program lifecycle events
('program.created',          'program', 'Culture program created'),
('program.config_updated',   'program', 'Program configuration changed'),
('program.paused',           'program', 'Program temporarily paused'),
('program.resumed',          'program', 'Program resumed'),

-- User lifecycle events
('user.joined',              'user', 'User added to organisation (manual or HRIS sync)'),
('user.profile_updated',     'user', 'User profile fields changed (department, location, etc.)'),
('user.consent_given',       'user', 'User gave consent for engagement tracking'),
('user.consent_revoked',     'user', 'User revoked engagement tracking consent'),
('user.deactivated',         'user', 'User deactivated (left org or offboarded)'),

-- HRIS sync events
('hris.sync_completed',      'org', 'HRIS sync cycle completed'),
('hris.user_created',        'org', 'New employee imported from HRIS'),
('hris.user_updated',        'org', 'Employee data updated from HRIS'),
('hris.celebration_imported', 'org', 'Birthday/anniversary date imported from HRIS');
```

---

## Event Data Examples

```sql
-- match.pair_created
-- data: {
--   "round_number": 14,
--   "pair_id": "uuid",
--   "members": [
--     {"user_id": "uuid", "department": "Engineering", "location": "SF"},
--     {"user_id": "uuid", "department": "Product", "location": "NYC"}
--   ],
--   "matching_reason": "cross_department, different_location",
--   "intro_message": "Hey! You've been matched for a virtual coffee ☕"
-- }

-- recognition.given
-- data: {
--   "giver_id": "uuid",
--   "receiver_id": "uuid",
--   "message": "Thanks for helping debug the production issue last night!",
--   "value_id": "v2",
--   "value_name": "Teamwork",
--   "points": 1,
--   "channel_id": "C-general"
-- }

-- celebration.card_signed
-- data: {
--   "celebrant_id": "uuid",
--   "celebration_type": "birthday",
--   "signer_id": "uuid",
--   "message": "Happy birthday! Hope you have an amazing day! 🎂",
--   "signers_so_far": 5,
--   "team_size": 8
-- }

-- engagement.isolation_flagged
-- data: {
--   "risk_score": 0.82,
--   "signals": [
--     {"signal": "no_matches_accepted", "period": "30_days"},
--     {"signal": "zero_recognitions_given", "period": "30_days"},
--     {"signal": "survey_skipped", "count": 3},
--     {"signal": "declining_mood_proxy", "trend": "response_length_decreasing"}
--   ],
--   "recommendation": "Manager should check in — multiple disengagement signals"
-- }

-- ai.narrative_generated
-- data: {
--   "period": {"start": "2026-05-01", "end": "2026-05-31"},
--   "narrative": "May saw the highest cross-team connection rate in six months...",
--   "highlights": ["Mentoring program reached 92% match completion", "Recognition volume up 23%"],
--   "concerns": ["Engineering pulse survey scores declined for second month"],
--   "metrics": {
--     "total_connections": 147,
--     "cross_team_pct": 0.62,
--     "recognition_volume": 234,
--     "avg_survey_score": 3.7,
--     "isolation_risk_users": 3
--   }
-- }
```

---

## Read Models

```sql
CREATE TABLE rm_users (
    user_id UUID PRIMARY KEY,
    org_id UUID NOT NULL,
    name TEXT NOT NULL,
    email TEXT NOT NULL,
    department TEXT,
    location TEXT,
    hire_date DATE,
    birthday DATE,
    points_balance INT NOT NULL DEFAULT 0,
    total_recognitions_given INT NOT NULL DEFAULT 0,
    total_recognitions_received INT NOT NULL DEFAULT 0,
    total_matches_completed INT NOT NULL DEFAULT 0,
    connection_count INT NOT NULL DEFAULT 0,
    isolation_risk_score REAL,
    consent_engagement BOOLEAN NOT NULL DEFAULT FALSE,
    last_event_version BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_users_org ON rm_users(org_id);
CREATE INDEX idx_rm_users_isolation ON rm_users(org_id, isolation_risk_score DESC)
    WHERE isolation_risk_score IS NOT NULL;

CREATE TABLE rm_connections (
    org_id UUID NOT NULL,
    user_a_id UUID NOT NULL,
    user_b_id UUID NOT NULL,
    interaction_count INT NOT NULL DEFAULT 1,
    sources TEXT[] NOT NULL DEFAULT '{}',
    last_interaction_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (org_id, user_a_id, user_b_id),
    CHECK (user_a_id < user_b_id)
);

CREATE INDEX idx_rm_conn_a ON rm_connections(user_a_id);
CREATE INDEX idx_rm_conn_b ON rm_connections(user_b_id);

CREATE TABLE rm_recognition_board (
    org_id UUID NOT NULL,
    receiver_id UUID NOT NULL,
    period_start DATE NOT NULL,
    recognitions_received INT NOT NULL DEFAULT 0,
    total_points INT NOT NULL DEFAULT 0,
    top_values TEXT[] NOT NULL DEFAULT '{}',
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (org_id, receiver_id, period_start)
);

CREATE INDEX idx_rm_recog ON rm_recognition_board(org_id, period_start, total_points DESC);

CREATE TABLE rm_survey_trends (
    program_id UUID NOT NULL,
    round_number INT NOT NULL,
    question_id TEXT NOT NULL,
    avg_score REAL,
    response_count INT NOT NULL DEFAULT 0,
    total_recipients INT NOT NULL DEFAULT 0,
    surveyed_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (program_id, round_number, question_id)
);

CREATE INDEX idx_rm_survey ON rm_survey_trends(program_id, surveyed_at);

CREATE TABLE rm_engagement_timeline (
    user_id UUID NOT NULL,
    event_date DATE NOT NULL,
    engagement_score REAL NOT NULL DEFAULT 0,
    matches_participated INT NOT NULL DEFAULT 0,
    recognitions_given INT NOT NULL DEFAULT 0,
    recognitions_received INT NOT NULL DEFAULT 0,
    surveys_completed INT NOT NULL DEFAULT 0,
    celebrations_signed INT NOT NULL DEFAULT 0,
    isolation_risk_score REAL,
    PRIMARY KEY (user_id, event_date)
);

CREATE INDEX idx_rm_engagement ON rm_engagement_timeline(user_id, event_date DESC);
```

---

## Reference Tables

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
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_org ON users(org_id);

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_event_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE stream_snapshots (
    stream_type TEXT NOT NULL,
    stream_id UUID NOT NULL,
    version BIGINT NOT NULL,
    state JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id)
);
```

---

## Example Queries

### All engagement events for a user in the last 30 days

```sql
SELECT event_type, data, occurred_at
FROM event_store
WHERE (metadata->>'actor_id') = 'user-uuid'
  AND occurred_at >= now() - INTERVAL '30 days'
ORDER BY occurred_at DESC;
```

### Isolation detection — users with no interaction events in 30 days

```sql
SELECT u.id, u.name, rm.isolation_risk_score
FROM rm_users rm
JOIN users u ON u.id = rm.user_id
WHERE rm.org_id = 'org-uuid'
  AND rm.isolation_risk_score > 0.7
ORDER BY rm.isolation_risk_score DESC;
```

### Match completion rate from events

```sql
SELECT
    COUNT(*) FILTER (WHERE event_type = 'match.pair_created') AS pairs_created,
    COUNT(*) FILTER (WHERE event_type = 'match.meeting_completed') AS meetings_completed,
    COUNT(*) FILTER (WHERE event_type = 'match.meeting_completed') * 100.0 /
        NULLIF(COUNT(*) FILTER (WHERE event_type = 'match.pair_created'), 0) AS completion_pct
FROM event_store
WHERE (metadata->>'program_id') = 'program-uuid'
  AND event_type IN ('match.pair_created', 'match.meeting_completed')
  AND occurred_at >= now() - INTERVAL '90 days';
```

### Recognition pattern — values distribution

```sql
SELECT data->>'value_name' AS value,
       COUNT(*) AS recognition_count,
       COUNT(DISTINCT data->>'giver_id') AS unique_givers
FROM event_store
WHERE event_type = 'recognition.given'
  AND (metadata->>'org_id') = 'org-uuid'
  AND occurred_at >= now() - INTERVAL '30 days'
GROUP BY data->>'value_name'
ORDER BY recognition_count DESC;
```

### Engagement timeline for team trend chart

```sql
SELECT et.event_date, AVG(et.engagement_score) AS avg_engagement,
       AVG(et.isolation_risk_score) AS avg_isolation_risk,
       COUNT(*) AS active_users
FROM rm_engagement_timeline et
JOIN users u ON u.id = et.user_id
JOIN teams t ON t.org_id = u.org_id
WHERE et.event_date >= CURRENT_DATE - 90
  AND u.org_id = 'org-uuid'
GROUP BY et.event_date
ORDER BY et.event_date;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 4 | event_store (partitioned), event_types, projection_checkpoints, stream_snapshots |
| Read Models | 5 | rm_users, rm_connections, rm_recognition_board, rm_survey_trends, rm_engagement_timeline |
| Reference | 2 | organisations, users |
| **Total** | **11** | Plus 4 quarterly partitions on event_store |

---

## Key Design Decisions

1. **Unified engagement stream** — All culture program interactions (matching, recognition, celebrations, surveys, watercooler) flow into a single event store. This enables cross-program analytics ("this user participates in matching but never gives recognition") and powers the AI culture narrative without querying multiple tables.

2. **Isolation detection from event absence** — The `engagement.isolation_flagged` event is triggered by an AI projection that scans for users with declining or absent interaction events. The signals are explicit in the event data (no matches accepted, zero recognitions, survey skipped). This is the inverse of typical event sourcing — detecting what DIDN'T happen.

3. **Connection graph as read model** — `rm_connections` materializes the relationship network from match completion events and recognition events. The graph rebuilds whenever interactions create new connections. This keeps the graph always consistent with actual interactions rather than declared relationships.

4. **Consent as events** — `user.consent_given` and `user.consent_revoked` events create an immutable audit trail of GDPR consent. The current consent status is projected into `rm_users.consent_engagement`. If consent is revoked, the system can identify which events were collected under that consent.

5. **Recognition board as time-windowed read model** — `rm_recognition_board` stores per-period (monthly) recognition counts. This avoids scanning the full event store for leaderboard queries. The board rebuilds from `recognition.given` events for the current period.

6. **HRIS sync events** — Employee data changes from BambooHR/Workday are recorded as events (`hris.user_created`, `hris.user_updated`, `hris.celebration_imported`). This provides full traceability of where employee data came from and when it was last synced.

7. **Engagement timeline for trend charts** — `rm_engagement_timeline` stores daily per-user engagement scores. This enables the manager dashboard to show engagement trends without computing scores from raw events. The timeline is updated by the `engagement.snapshot_taken` event.

8. **Quarterly partitions** — Culture platforms generate moderate event volume (org_size × programs × daily_interactions). A 200-person org with 4 active programs generates roughly 50,000-100,000 events per quarter. Quarterly partitions keep queries efficient while enabling cold storage of older quarters.
