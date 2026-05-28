# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Language Learning Platform · Created: 2026-05-19

## Philosophy

This model treats every learning interaction as an immutable event stored in an append-only event store. The event store is the single source of truth; all queryable state (learner progress, SRS card parameters, proficiency estimates, pronunciation history) is derived from replaying or projecting events into materialised read models. This is the Command Query Responsibility Segregation (CQRS) pattern: commands write events, queries read from projections.

The event-sourced approach is particularly well-suited to a language learning platform because the domain is fundamentally temporal. Learners progress over time, forget and relearn, and the entire value proposition of spaced repetition is modelling memory decay as a function of time. An event-sourced system can answer temporal queries naturally ("what was this learner's estimated CEFR level on March 15th?", "show the complete pronunciation improvement trajectory for /r/ over the past 6 months") by replaying events up to a point in time. This is exactly the kind of audit trail that institutional and enterprise buyers require for compliance reporting.

Event sourcing also provides a natural foundation for FSRS parameter optimisation: the FSRS optimizer needs the complete review history (every rating, every interval, every state transition) to train personalised memory model parameters. In a traditional CRUD model this requires a separate audit table; in an event-sourced model it is the primary data structure.

**Best for:** Platforms where full audit trails, temporal queries, FSRS optimisation, and institutional compliance reporting are primary requirements.

**Trade-offs:**
- (+) Complete, immutable audit trail of every learning interaction — nothing is ever lost
- (+) Natural temporal queries: replay to any point in time to reconstruct past state
- (+) FSRS parameter optimisation uses the event log directly — no separate audit table needed
- (+) xAPI statement generation is trivial: each event maps 1:1 to an xAPI Actor-Verb-Object statement
- (+) Supports offline-first mobile: events are collected locally and synced when online
- (-) Higher complexity: developers must understand event sourcing, projections, and eventual consistency
- (-) Read model projections must be maintained and rebuilt when projection logic changes
- (-) Simple queries (e.g., "current CEFR level") require reading from a projection, not the event store
- (-) Storage grows indefinitely; event compaction or snapshotting strategies are needed at scale
- (-) Debugging is harder: the "current state" of an entity requires understanding which events produced it

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CEFR (A1-C2) | Events carry CEFR level context; proficiency projection tracks level transitions over time |
| ACTFL Proficiency Guidelines | Mapped via projection from CEFR events for US institutional reporting |
| FSRS Algorithm (MIT) | Review events store rating and are replayed to compute FSRS parameters; optimizer runs directly on event stream |
| xAPI / IEEE 9274.1.1 | Each learning event maps directly to an xAPI statement; the event store IS the Learning Record Store |
| SCORM 2004 | SCORM tracking data generated from event projections on export |
| IMS LTI 1.3 | LTI launch and grade passback events stored as first-class events |
| ISO 639-1/639-3 | Reference data in static lookup tables (not event-sourced) |
| W3C Web Speech API | Pronunciation assessment events include phoneme-level detail |
| OpenID Connect / SAML 2.0 | Authentication events tracked; SSO configuration in reference tables |

---

## Event Store

```sql
-- The core event store: append-only, immutable
CREATE TABLE learning_event (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,           -- aggregate root ID (user_id + language_id composite concept)
    stream_type     VARCHAR(50) NOT NULL,     -- e.g. 'learner_journey', 'review_card', 'conversation', 'course_authoring'
    event_type      VARCHAR(100) NOT NULL,    -- e.g. 'ExerciseCompleted', 'ReviewRated', 'PronunciationAssessed'
    event_version   INTEGER NOT NULL DEFAULT 1, -- schema version for this event type
    sequence_number BIGINT NOT NULL,          -- monotonically increasing within stream
    user_id         UUID NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {"device": "ios", "app_version": "2.1.0", "session_id": "abc-123", "ip_country": "DE"}
    payload         JSONB NOT NULL,
    -- payload varies by event_type (see Event Catalog below)
    UNIQUE(stream_id, sequence_number)
);

-- Write-optimised: append by stream, ordered by sequence
CREATE INDEX idx_event_stream ON learning_event(stream_id, sequence_number);
-- Query by user across all streams
CREATE INDEX idx_event_user ON learning_event(user_id, occurred_at);
-- Query by event type for projection rebuilds
CREATE INDEX idx_event_type ON learning_event(event_type, occurred_at);
-- GIN index on payload for ad-hoc JSON queries
CREATE INDEX idx_event_payload ON learning_event USING GIN(payload);

-- Snapshots to avoid full replay for long-lived streams
CREATE TABLE stream_snapshot (
    stream_id       UUID NOT NULL,
    snapshot_version BIGINT NOT NULL,         -- sequence_number at snapshot time
    stream_type     VARCHAR(50) NOT NULL,
    state           JSONB NOT NULL,           -- projected state at this point
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

## Event Catalog

The `event_type` field determines the structure of `payload`. Below are the core event types:

```sql
-- Event catalog (documentation table, not enforced at DB level)
-- This serves as a schema registry for event types.

/*
EVENT: LearnerRegistered
stream_type: learner_journey
payload: {
    "email": "user@example.com",
    "display_name": "Maria Garcia",
    "native_language": "en",
    "organisation_id": "uuid-or-null",
    "auth_provider": "local"
}

EVENT: CourseEnrolled
stream_type: learner_journey
payload: {
    "course_id": "uuid",
    "target_language": "es",
    "source_language": "en",
    "cefr_level_start": "A1"
}

EVENT: LessonStarted
stream_type: learner_journey
payload: {
    "lesson_id": "uuid",
    "unit_id": "uuid",
    "course_id": "uuid",
    "lesson_type": "vocabulary",
    "cefr_level": "A1"
}

EVENT: ExerciseCompleted
stream_type: learner_journey
payload: {
    "exercise_id": "uuid",
    "exercise_type": "translate_to_target",
    "is_correct": true,
    "user_answer": "Yo como manzanas",
    "correct_answer": "Yo como manzanas",
    "response_time_ms": 4200,
    "vocabulary_item_id": "uuid",
    "cefr_level": "A1"
}

EVENT: LessonCompleted
stream_type: learner_journey
payload: {
    "lesson_id": "uuid",
    "exercises_completed": 12,
    "exercises_correct": 10,
    "duration_seconds": 420,
    "xp_earned": 15
}

EVENT: ReviewRated
stream_type: review_card
payload: {
    "card_id": "uuid",
    "vocabulary_item_id": "uuid",
    "rating": 3,
    "state_before": "review",
    "state_after": "review",
    "stability_before": 14.2,
    "stability_after": 28.7,
    "difficulty_before": 5.1,
    "difficulty_after": 4.9,
    "scheduled_days": 29,
    "elapsed_days": 14,
    "response_time_ms": 3100
}

EVENT: PronunciationAssessed
stream_type: learner_journey
payload: {
    "reference_text": "Buenos días, ¿cómo estás?",
    "recognised_text": "Buenos días, como estás",
    "accuracy_score": 82.5,
    "fluency_score": 78.0,
    "completeness_score": 100.0,
    "prosody_score": 71.0,
    "overall_score": 79.5,
    "phonemes": [
        {"phoneme": "k", "accuracy": 95.0, "word": "cómo"},
        {"phoneme": "ɾ", "accuracy": 42.0, "word": "estás", "error_type": "substitution",
         "guidance": "Tap the tongue tip against the alveolar ridge briefly"}
    ],
    "provider": "azure"
}

EVENT: ConversationStarted
stream_type: conversation
payload: {
    "target_language": "es",
    "scenario": "Ordering at a restaurant",
    "cefr_level": "A2"
}

EVENT: ConversationTurnSpoken
stream_type: conversation
payload: {
    "turn_number": 3,
    "speaker": "learner",
    "content": "Quisiera una ensalada, por favor",
    "audio_url": "s3://...",
    "grammar_errors": [
        {"error": "quisiera", "note": "Correct use of conditional — well done!"}
    ]
}

EVENT: ConversationEnded
stream_type: conversation
payload: {
    "total_turns": 12,
    "duration_seconds": 480,
    "vocabulary_used": ["quisiera", "ensalada", "cuenta", "propina"]
}

EVENT: ProficiencyEstimated
stream_type: learner_journey
payload: {
    "target_language": "es",
    "skill": "speaking",
    "cefr_level": "A2",
    "confidence": 0.78,
    "model_version": "proficiency-v3"
}

EVENT: StreakUpdated
stream_type: learner_journey
payload: {
    "streak_days": 15,
    "streak_date": "2026-05-19"
}

EVENT: AchievementEarned
stream_type: learner_journey
payload: {
    "achievement_id": "uuid",
    "achievement_name": "First Conversation",
    "criteria_type": "conversation_count",
    "criteria_value": 1
}
*/
```

## Reference Data (Static Tables — Not Event-Sourced)

```sql
-- These are reference/lookup tables that don't change per-user
-- and don't benefit from event sourcing.

CREATE TABLE language (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name_english    VARCHAR(100) NOT NULL,
    name_native     VARCHAR(100) NOT NULL,
    iso_639_1       CHAR(2) NOT NULL UNIQUE,
    iso_639_3       CHAR(3) NOT NULL UNIQUE,
    script          VARCHAR(50) NOT NULL DEFAULT 'Latin',
    writing_direction VARCHAR(3) NOT NULL DEFAULT 'ltr',
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    org_type        VARCHAR(50) NOT NULL,
    max_seats       INTEGER,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'free',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE course (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_language_id UUID NOT NULL REFERENCES language(id),
    source_language_id UUID NOT NULL REFERENCES language(id),
    title           VARCHAR(255) NOT NULL,
    cefr_level_start VARCHAR(10) NOT NULL DEFAULT 'A1',
    cefr_level_end   VARCHAR(10) NOT NULL DEFAULT 'B2',
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE unit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES course(id),
    title           VARCHAR(255) NOT NULL,
    cefr_level      VARCHAR(10) NOT NULL,
    sort_order      INTEGER NOT NULL
);

CREATE TABLE lesson (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id         UUID NOT NULL REFERENCES unit(id),
    title           VARCHAR(255) NOT NULL,
    lesson_type     VARCHAR(50) NOT NULL,
    cefr_level      VARCHAR(10) NOT NULL,
    sort_order      INTEGER NOT NULL
);

CREATE TABLE exercise (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lesson_id       UUID NOT NULL REFERENCES lesson(id),
    exercise_type   VARCHAR(50) NOT NULL,
    prompt_text     TEXT,
    correct_answer  TEXT,
    alternatives    JSONB,
    vocabulary_item_id UUID,
    cefr_level      VARCHAR(10) NOT NULL,
    sort_order      INTEGER NOT NULL
);

CREATE TABLE vocabulary_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_language_id UUID NOT NULL REFERENCES language(id),
    source_language_id UUID NOT NULL REFERENCES language(id),
    target_text     VARCHAR(500) NOT NULL,
    source_text     VARCHAR(500) NOT NULL,
    pronunciation_ipa VARCHAR(255),
    audio_url       TEXT,
    part_of_speech  VARCHAR(50),
    cefr_level      VARCHAR(10) NOT NULL,
    frequency_rank  INTEGER
);

CREATE TABLE sso_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    protocol        VARCHAR(20) NOT NULL,
    entity_id       VARCHAR(500),
    metadata_xml    TEXT,
    oidc_client_id  VARCHAR(255),
    oidc_discovery_url TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE lti_registration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    platform_name   VARCHAR(255) NOT NULL,
    issuer          VARCHAR(500) NOT NULL,
    client_id       VARCHAR(255) NOT NULL,
    auth_endpoint   TEXT NOT NULL,
    token_endpoint  TEXT NOT NULL,
    jwks_url        TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true
);
```

## Materialised Read Models (Projections)

These tables are derived from events and can be rebuilt from the event store at any time. They serve read queries efficiently.

```sql
-- Projection: current user state
CREATE TABLE projection_user (
    user_id         UUID PRIMARY KEY,
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    organisation_id UUID REFERENCES organisation(id),
    native_language_id UUID REFERENCES language(id),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    role            VARCHAR(50) NOT NULL DEFAULT 'learner',
    streak_days     INTEGER NOT NULL DEFAULT 0,
    streak_last_date DATE,
    daily_goal_minutes INTEGER NOT NULL DEFAULT 15,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_event_seq  BIGINT NOT NULL DEFAULT 0,  -- watermark for projection rebuilds
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection: current FSRS card state
CREATE TABLE projection_review_card (
    card_id         UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    vocabulary_item_id UUID NOT NULL,
    state           VARCHAR(20) NOT NULL DEFAULT 'new',
    stability       REAL NOT NULL DEFAULT 0.0,
    difficulty      REAL NOT NULL DEFAULT 0.0,
    elapsed_days    INTEGER NOT NULL DEFAULT 0,
    scheduled_days  INTEGER NOT NULL DEFAULT 0,
    reps            INTEGER NOT NULL DEFAULT 0,
    lapses          INTEGER NOT NULL DEFAULT 0,
    last_review_at  TIMESTAMPTZ,
    next_review_at  TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, vocabulary_item_id)
);

CREATE INDEX idx_proj_review_due ON projection_review_card(user_id, next_review_at)
    WHERE state != 'new';

-- Projection: learner language progress
CREATE TABLE projection_learner_language (
    user_id         UUID NOT NULL,
    target_language_id UUID NOT NULL,
    words_known     INTEGER NOT NULL DEFAULT 0,
    words_learning  INTEGER NOT NULL DEFAULT 0,
    total_reviews   BIGINT NOT NULL DEFAULT 0,
    total_study_time_seconds BIGINT NOT NULL DEFAULT 0,
    conversation_count INTEGER NOT NULL DEFAULT 0,
    avg_pronunciation_score REAL,
    current_cefr_level VARCHAR(10),
    current_cefr_confidence REAL,
    lessons_completed INTEGER NOT NULL DEFAULT 0,
    last_activity_at TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, target_language_id)
);

-- Projection: proficiency timeline (for temporal queries and charts)
CREATE TABLE projection_proficiency_timeline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL,
    target_language_id UUID NOT NULL,
    skill           VARCHAR(20) NOT NULL,
    cefr_level      VARCHAR(10) NOT NULL,
    confidence      REAL NOT NULL,
    estimated_at    TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_prof_timeline ON projection_proficiency_timeline(user_id, target_language_id, skill, estimated_at);

-- Projection: pronunciation weakness tracker
CREATE TABLE projection_pronunciation_weakness (
    user_id         UUID NOT NULL,
    target_language_id UUID NOT NULL,
    phoneme         VARCHAR(20) NOT NULL,
    avg_accuracy    REAL NOT NULL,
    attempt_count   INTEGER NOT NULL DEFAULT 0,
    last_assessed_at TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, target_language_id, phoneme)
);

-- Projection: daily activity (for streak calculation and dashboards)
CREATE TABLE projection_daily_activity (
    user_id         UUID NOT NULL,
    activity_date   DATE NOT NULL,
    target_language_id UUID NOT NULL,
    study_time_seconds INTEGER NOT NULL DEFAULT 0,
    exercises_completed INTEGER NOT NULL DEFAULT 0,
    exercises_correct   INTEGER NOT NULL DEFAULT 0,
    reviews_completed INTEGER NOT NULL DEFAULT 0,
    conversations_count INTEGER NOT NULL DEFAULT 0,
    xp_earned       INTEGER NOT NULL DEFAULT 0,
    last_event_seq  BIGINT NOT NULL DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, activity_date, target_language_id)
);

-- Projection: xAPI statement view (for LRS export)
CREATE TABLE projection_xapi_statement (
    statement_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,           -- back-reference to source event
    actor_user_id   UUID NOT NULL,
    actor_email     VARCHAR(255) NOT NULL,
    verb_id         VARCHAR(255) NOT NULL,
    verb_display    VARCHAR(100) NOT NULL,
    object_type     VARCHAR(100) NOT NULL DEFAULT 'Activity',
    object_id       VARCHAR(500) NOT NULL,
    object_name     VARCHAR(255),
    result_score_raw REAL,
    result_success  BOOLEAN,
    result_completion BOOLEAN,
    result_duration INTERVAL,
    context_json    JSONB,
    timestamp_at    TIMESTAMPTZ NOT NULL,
    stored_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_xapi_proj_actor ON projection_xapi_statement(actor_user_id, timestamp_at);
```

## Event-to-xAPI Mapping

```sql
/*
Event-to-xAPI mapping rules (implemented in projection handler):

ExerciseCompleted → verb: "completed" (http://adlnet.gov/expapi/verbs/completed)
                    object: exercise activity IRI
                    result: {score: {raw: is_correct ? 100 : 0}, success: is_correct}

ReviewRated       → verb: "answered" (http://adlnet.gov/expapi/verbs/answered)
                    object: vocabulary item IRI
                    result: {score: {raw: rating * 25}, success: rating >= 3}

LessonCompleted   → verb: "completed"
                    object: lesson activity IRI
                    result: {score: {raw: exercises_correct/exercises_completed * 100},
                             completion: true, duration: PT{duration_seconds}S}

ProficiencyEstimated → verb: "progressed" (http://adlnet.gov/expapi/verbs/progressed)
                       object: language activity IRI
                       result: {extensions: {"cefr_level": "A2", "skill": "speaking"}}

ConversationEnded → verb: "experienced" (http://adlnet.gov/expapi/verbs/experienced)
                    object: conversation scenario IRI
                    result: {duration: PT{duration_seconds}S}
*/
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | learning_event, stream_snapshot |
| Reference Data | 10 | language, organisation, course, unit, lesson, exercise, vocabulary_item, sso_connection, lti_registration + CEFR mappings |
| Read Model Projections | 7 | projection_user, projection_review_card, projection_learner_language, projection_proficiency_timeline, projection_pronunciation_weakness, projection_daily_activity, projection_xapi_statement |
| **Total** | **19** | Lower table count than normalized model; complexity shifts to projection handlers |

---

## Key Design Decisions

1. **Single `learning_event` table as the source of truth** — all learner interactions (exercises, reviews, conversations, pronunciation assessments, proficiency estimates) are stored as events. This eliminates the need for separate audit tables and provides a complete, immutable record.

2. **Stream-based organisation** — events are grouped into streams (identified by `stream_id` + `stream_type`). A learner's journey in a language is one stream; each review card is a separate stream. This allows efficient replay of a single aggregate without scanning the entire event store.

3. **JSONB payloads with event versioning** — `event_version` enables schema evolution. When the payload structure for `ExerciseCompleted` changes, the version increments and projection handlers can handle both old and new formats.

4. **Snapshots for performance** — `stream_snapshot` stores periodic state snapshots to avoid replaying the full event history for long-lived streams. A learner with 10,000+ review events over years of study would be expensive to replay without snapshots.

5. **xAPI as a natural projection** — because each learning event maps directly to an xAPI Actor-Verb-Object statement, the xAPI LRS is simply a read model projection. No separate xAPI event capture is needed. This means the platform IS an LRS natively.

6. **FSRS optimizer runs on the event stream** — the `ReviewRated` events contain all fields the FSRS optimizer needs (rating, state transitions, stability/difficulty before and after, elapsed/scheduled days). Personalised FSRS parameters can be computed by querying the event store directly.

7. **Reference data is NOT event-sourced** — languages, courses, lessons, exercises, and vocabulary items are static reference data managed by content authors. Event sourcing is applied to learner-generated data, not content authoring. This avoids unnecessary complexity.

8. **Projections include `last_event_seq` watermark** — each projection row tracks which event it was last updated from. This enables incremental projection updates and idempotent rebuild: if a projection is behind, it can catch up from `last_event_seq` without replaying everything.

9. **Temporal queries via event replay** — to answer "what was the learner's CEFR level on date X?", replay `ProficiencyEstimated` events up to that date. To answer "how many words did the learner know on date X?", replay `ExerciseCompleted` and `ReviewRated` events. No separate temporal history tables needed.

10. **Offline-first mobile sync** — events are created on the mobile device with local UUIDs and timestamps, then synced to the server. Conflict resolution is natural: events are append-only, so conflicts are impossible (duplicate events are deduplicated by `event_id`).

---

## Example Queries

### Rebuild a review card's current state from events

```sql
SELECT payload
FROM learning_event
WHERE stream_id = :card_stream_id
  AND stream_type = 'review_card'
ORDER BY sequence_number ASC;

-- Application code replays these events to compute current
-- stability, difficulty, state, next_review_at
```

### Get CEFR proficiency at a specific past date

```sql
SELECT DISTINCT ON (payload->>'skill')
    payload->>'skill' AS skill,
    payload->>'cefr_level' AS cefr_level,
    (payload->>'confidence')::REAL AS confidence,
    occurred_at
FROM learning_event
WHERE user_id = :user_id
  AND event_type = 'ProficiencyEstimated'
  AND payload->>'target_language' = 'es'
  AND occurred_at <= '2026-03-15T23:59:59Z'
ORDER BY payload->>'skill', occurred_at DESC;
```

### Export xAPI statements for an LMS

```sql
SELECT statement_id, actor_email, verb_id, verb_display,
       object_id, object_name, result_score_raw, result_success,
       result_completion, result_duration, context_json, timestamp_at
FROM projection_xapi_statement
WHERE actor_user_id = :user_id
  AND timestamp_at BETWEEN :start_date AND :end_date
ORDER BY timestamp_at ASC;
```

### FSRS optimizer: get all review history for a user

```sql
SELECT
    payload->>'card_id' AS card_id,
    (payload->>'rating')::INTEGER AS rating,
    payload->>'state_before' AS state_before,
    payload->>'state_after' AS state_after,
    (payload->>'stability_before')::REAL AS stability_before,
    (payload->>'stability_after')::REAL AS stability_after,
    (payload->>'difficulty_before')::REAL AS difficulty_before,
    (payload->>'difficulty_after')::REAL AS difficulty_after,
    (payload->>'elapsed_days')::INTEGER AS elapsed_days,
    (payload->>'scheduled_days')::INTEGER AS scheduled_days,
    occurred_at
FROM learning_event
WHERE user_id = :user_id
  AND event_type = 'ReviewRated'
ORDER BY occurred_at ASC;
```
