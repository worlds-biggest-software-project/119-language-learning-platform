# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Language Learning Platform · Created: 2026-05-19

## Philosophy

This model follows a fully normalized relational approach where every domain concept has its own dedicated table with explicit foreign key relationships. The schema is designed for maximum data integrity, clear audit semantics, and straightforward SQL querying. Each entity — learners, languages, courses, lessons, vocabulary items, review cards, pronunciation assessments, conversations — has a first-class table with well-defined columns and constraints.

The normalized approach mirrors how established LMS platforms (Moodle, Canvas) and enterprise SaaS applications structure their data. It prioritises referential integrity, makes schema evolution explicit via migrations, and supports complex cross-entity reporting queries (e.g., "show me all learners in organisation X who have reached B1 in Spanish with pronunciation accuracy above 80%") without requiring JSON path traversals or event replay.

This is the most conventional choice and the easiest to reason about for a team familiar with relational databases. It trades some flexibility (adding a new language-specific field requires a migration) for clarity and query performance on well-indexed columns.

**Best for:** Teams that prioritise data integrity, complex reporting, and straightforward SQL queries over schema flexibility.

**Trade-offs:**
- (+) Maximum referential integrity; the database enforces all relationships
- (+) Standard SQL queries for all reporting; no special query syntax needed
- (+) Well-understood by most backend developers; easy to hire for
- (+) Excellent tooling support (ORMs, migration frameworks, query builders)
- (-) Adding language-specific or jurisdiction-specific fields requires schema migrations
- (-) High table count increases migration complexity as the system grows
- (-) Temporal queries ("what was the learner's level on date X?") require explicit history tables
- (-) Less flexible for rapid prototyping; every new concept needs a migration

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CEFR (A1-C2) | `cefr_level` enum used throughout; `cefr_descriptor` table maps levels to official descriptors; proficiency estimates stored as CEFR levels |
| ACTFL Proficiency Guidelines | `actfl_level` enum on proficiency records for US education contexts; mapping table links CEFR to ACTFL |
| FSRS Algorithm (MIT) | `review_card` table stores FSRS state fields: `stability`, `difficulty`, `retrievability`, `state`, `last_review_at`, `next_review_at` |
| xAPI / IEEE 9274.1.1 | `xapi_statement` table stores Actor-Verb-Object statements for LRS compatibility; exportable to external LRS |
| SCORM 2004 | `scorm_package` and `scorm_tracking` tables support SCORM content import and progress tracking |
| IMS LTI 1.3 | `lti_registration` and `lti_deployment` tables store LTI platform credentials and launch context |
| ISO 639-1/639-3 | `language.iso_639_1` and `language.iso_639_3` columns for standard language identification |
| Unicode / CLDR | `language.script` and `language.writing_direction` columns informed by CLDR data |
| W3C Web Speech API | Pronunciation assessment results stored in `pronunciation_assessment` with phoneme-level detail |
| OpenID Connect / SAML 2.0 | `sso_connection` table for enterprise IdP configurations |

---

## Core Identity & Multi-Tenancy

```sql
-- Organisations (enterprise/education tenants)
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    org_type        VARCHAR(50) NOT NULL CHECK (org_type IN ('enterprise', 'education', 'government', 'individual')),
    max_seats       INTEGER,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'free',
    saml_metadata_url TEXT,            -- SAML 2.0 IdP metadata URL
    oidc_issuer_url   TEXT,            -- OpenID Connect issuer
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organisation_slug ON organisation(slug);

-- Users / Learners
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID REFERENCES organisation(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),       -- NULL for SSO-only users
    native_language_id UUID,            -- FK to language table
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local' CHECK (auth_provider IN ('local', 'saml', 'oidc', 'google', 'apple')),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    daily_goal_minutes INTEGER NOT NULL DEFAULT 15,
    streak_days     INTEGER NOT NULL DEFAULT 0,
    streak_last_date DATE,
    role            VARCHAR(50) NOT NULL DEFAULT 'learner' CHECK (role IN ('learner', 'teacher', 'admin', 'org_admin', 'super_admin')),
    cefr_self_assessed VARCHAR(10),     -- learner's self-reported starting level
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_app_user_organisation ON app_user(organisation_id);
CREATE INDEX idx_app_user_email ON app_user(email);

-- SSO Connections for enterprise
CREATE TABLE sso_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    protocol        VARCHAR(20) NOT NULL CHECK (protocol IN ('saml2', 'oidc')),
    entity_id       VARCHAR(500),       -- SAML entity ID
    metadata_xml    TEXT,               -- SAML metadata document
    oidc_client_id  VARCHAR(255),
    oidc_client_secret VARCHAR(255),
    oidc_discovery_url TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Language & Curriculum

```sql
-- Supported languages
CREATE TABLE language (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name_english    VARCHAR(100) NOT NULL,  -- e.g. "Spanish"
    name_native     VARCHAR(100) NOT NULL,  -- e.g. "Español"
    iso_639_1       CHAR(2) NOT NULL UNIQUE, -- ISO 639-1 code (e.g. "es")
    iso_639_3       CHAR(3) NOT NULL UNIQUE, -- ISO 639-3 code (e.g. "spa")
    script          VARCHAR(50) NOT NULL DEFAULT 'Latin', -- Unicode script name
    writing_direction VARCHAR(3) NOT NULL DEFAULT 'ltr' CHECK (writing_direction IN ('ltr', 'rtl')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- CEFR level descriptors
CREATE TABLE cefr_descriptor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    level           VARCHAR(10) NOT NULL CHECK (level IN ('A1', 'A2', 'B1', 'B2', 'C1', 'C2')),
    skill           VARCHAR(20) NOT NULL CHECK (skill IN ('reading', 'writing', 'listening', 'speaking', 'interaction')),
    descriptor_text TEXT NOT NULL,       -- official CEFR descriptor from Companion Volume 2020
    language_id     UUID REFERENCES language(id), -- NULL = language-agnostic descriptor
    UNIQUE(level, skill, language_id)
);

-- CEFR to ACTFL mapping
CREATE TABLE cefr_actfl_mapping (
    cefr_level      VARCHAR(10) NOT NULL,
    actfl_level     VARCHAR(30) NOT NULL, -- e.g. "Intermediate Mid"
    PRIMARY KEY (cefr_level, actfl_level)
);

-- Courses (e.g. "Spanish for English Speakers")
CREATE TABLE course (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_language_id UUID NOT NULL REFERENCES language(id),
    source_language_id UUID NOT NULL REFERENCES language(id),
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    cefr_level_start VARCHAR(10) NOT NULL DEFAULT 'A1',
    cefr_level_end   VARCHAR(10) NOT NULL DEFAULT 'B2',
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_course_languages ON course(target_language_id, source_language_id);

-- Units within a course
CREATE TABLE unit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES course(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    cefr_level      VARCHAR(10) NOT NULL,
    sort_order      INTEGER NOT NULL,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_unit_course ON unit(course_id, sort_order);

-- Lessons within a unit
CREATE TABLE lesson (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id         UUID NOT NULL REFERENCES unit(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    lesson_type     VARCHAR(50) NOT NULL CHECK (lesson_type IN ('vocabulary', 'grammar', 'listening', 'speaking', 'reading', 'conversation', 'review', 'assessment')),
    cefr_level      VARCHAR(10) NOT NULL,
    estimated_minutes INTEGER NOT NULL DEFAULT 10,
    sort_order      INTEGER NOT NULL,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lesson_unit ON lesson(unit_id, sort_order);
```

## Vocabulary & Grammar

```sql
-- Vocabulary items (language-pair specific)
CREATE TABLE vocabulary_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_language_id UUID NOT NULL REFERENCES language(id),
    source_language_id UUID NOT NULL REFERENCES language(id),
    target_text     VARCHAR(500) NOT NULL,  -- word or phrase in target language
    source_text     VARCHAR(500) NOT NULL,  -- translation in source language
    pronunciation_ipa VARCHAR(255),         -- IPA transcription
    audio_url       TEXT,                   -- URL to native speaker audio
    part_of_speech  VARCHAR(50),            -- noun, verb, adjective, etc.
    cefr_level      VARCHAR(10) NOT NULL,
    frequency_rank  INTEGER,                -- corpus frequency rank
    example_sentence_target TEXT,
    example_sentence_source TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vocab_target_lang ON vocabulary_item(target_language_id, cefr_level);
CREATE INDEX idx_vocab_frequency ON vocabulary_item(target_language_id, frequency_rank);

-- Vocabulary-to-lesson junction
CREATE TABLE lesson_vocabulary (
    lesson_id       UUID NOT NULL REFERENCES lesson(id) ON DELETE CASCADE,
    vocabulary_item_id UUID NOT NULL REFERENCES vocabulary_item(id),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (lesson_id, vocabulary_item_id)
);

-- Grammar rules
CREATE TABLE grammar_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_language_id UUID NOT NULL REFERENCES language(id),
    source_language_id UUID NOT NULL REFERENCES language(id),
    title           VARCHAR(255) NOT NULL,
    explanation     TEXT NOT NULL,
    cefr_level      VARCHAR(10) NOT NULL,
    examples_json   JSONB NOT NULL DEFAULT '[]',
    -- examples_json example:
    -- [
    --   {"target": "Yo como manzanas", "source": "I eat apples", "highlight": "como"},
    --   {"target": "Ella come pan", "source": "She eats bread", "highlight": "come"}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_grammar_rule_lang ON grammar_rule(target_language_id, cefr_level);
```

## Exercises & Assessments

```sql
-- Exercise templates within lessons
CREATE TABLE exercise (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lesson_id       UUID NOT NULL REFERENCES lesson(id) ON DELETE CASCADE,
    exercise_type   VARCHAR(50) NOT NULL CHECK (exercise_type IN (
        'multiple_choice', 'fill_blank', 'translate_to_target', 'translate_to_source',
        'listening_comprehension', 'speaking_repeat', 'speaking_freeform',
        'word_order', 'matching', 'dictation', 'conversation_turn'
    )),
    prompt_text     TEXT,                   -- question or instruction
    prompt_audio_url TEXT,                  -- audio for listening exercises
    correct_answer  TEXT,                   -- expected answer (NULL for freeform)
    alternatives    JSONB,                  -- wrong answers for multiple choice
    -- alternatives example: ["como", "comes", "comen", "comemos"]
    vocabulary_item_id UUID REFERENCES vocabulary_item(id),
    grammar_rule_id UUID REFERENCES grammar_rule(id),
    cefr_level      VARCHAR(10) NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_exercise_lesson ON exercise(lesson_id, sort_order);

-- Learner exercise attempts
CREATE TABLE exercise_attempt (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    exercise_id     UUID NOT NULL REFERENCES exercise(id),
    session_id      UUID NOT NULL,          -- groups attempts within one study session
    user_answer     TEXT,
    is_correct      BOOLEAN NOT NULL,
    response_time_ms INTEGER,               -- time taken to answer
    attempt_number  INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_exercise_attempt_user ON exercise_attempt(user_id, created_at);
CREATE INDEX idx_exercise_attempt_session ON exercise_attempt(session_id);
```

## Spaced Repetition (FSRS)

```sql
-- Review cards linked to vocabulary items (one per user per vocab item per language pair)
CREATE TABLE review_card (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    vocabulary_item_id UUID NOT NULL REFERENCES vocabulary_item(id),
    -- FSRS algorithm state
    state           VARCHAR(20) NOT NULL DEFAULT 'new' CHECK (state IN ('new', 'learning', 'review', 'relearning')),
    stability       REAL NOT NULL DEFAULT 0.0,       -- FSRS stability (storage strength)
    difficulty      REAL NOT NULL DEFAULT 0.0,       -- FSRS difficulty (0-10 scale)
    elapsed_days    INTEGER NOT NULL DEFAULT 0,
    scheduled_days  INTEGER NOT NULL DEFAULT 0,
    reps            INTEGER NOT NULL DEFAULT 0,       -- total review count
    lapses          INTEGER NOT NULL DEFAULT 0,       -- times forgotten
    last_review_at  TIMESTAMPTZ,
    next_review_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, vocabulary_item_id)
);

CREATE INDEX idx_review_card_next ON review_card(user_id, next_review_at) WHERE state != 'new';
CREATE INDEX idx_review_card_state ON review_card(user_id, state);

-- Review log (immutable history of all reviews)
CREATE TABLE review_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_card_id  UUID NOT NULL REFERENCES review_card(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    rating          SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 4),
    -- 1=Again, 2=Hard, 3=Good, 4=Easy (FSRS rating scale)
    state_before    VARCHAR(20) NOT NULL,
    state_after     VARCHAR(20) NOT NULL,
    stability_before REAL NOT NULL,
    stability_after  REAL NOT NULL,
    difficulty_before REAL NOT NULL,
    difficulty_after  REAL NOT NULL,
    scheduled_days  INTEGER NOT NULL,
    elapsed_days    INTEGER NOT NULL,
    response_time_ms INTEGER,
    reviewed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_review_log_card ON review_log(review_card_id, reviewed_at);
CREATE INDEX idx_review_log_user ON review_log(user_id, reviewed_at);
```

## Pronunciation Assessment

```sql
-- Pronunciation assessment sessions
CREATE TABLE pronunciation_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    exercise_id     UUID REFERENCES exercise(id),
    target_language_id UUID NOT NULL REFERENCES language(id),
    reference_text  TEXT NOT NULL,           -- expected utterance
    recognised_text TEXT,                    -- what ASR transcribed
    audio_recording_url TEXT,               -- stored learner audio
    -- Aggregate scores (0.0 - 100.0)
    accuracy_score  REAL,
    fluency_score   REAL,
    completeness_score REAL,
    prosody_score   REAL,
    overall_score   REAL,
    assessment_provider VARCHAR(50) NOT NULL DEFAULT 'azure',  -- azure, whisper, google
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pronunciation_user ON pronunciation_assessment(user_id, created_at);

-- Phoneme-level detail for pronunciation assessments
CREATE TABLE pronunciation_phoneme (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assessment_id   UUID NOT NULL REFERENCES pronunciation_assessment(id) ON DELETE CASCADE,
    phoneme         VARCHAR(20) NOT NULL,    -- IPA symbol
    accuracy_score  REAL NOT NULL,           -- 0.0-100.0
    word_context    VARCHAR(255),            -- the word containing this phoneme
    position_in_word INTEGER,
    error_type      VARCHAR(50),             -- e.g. 'substitution', 'omission', 'insertion'
    native_language_contrast TEXT,           -- explanation of L1 interference
    corrective_guidance TEXT                 -- articulatory instruction
);

CREATE INDEX idx_pronunciation_phoneme_assessment ON pronunciation_phoneme(assessment_id);
```

## AI Conversation

```sql
-- AI conversation sessions
CREATE TABLE conversation_session (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    target_language_id UUID NOT NULL REFERENCES language(id),
    scenario_title  VARCHAR(255),            -- e.g. "Ordering at a restaurant"
    cefr_level      VARCHAR(10) NOT NULL,
    total_turns     INTEGER NOT NULL DEFAULT 0,
    duration_seconds INTEGER,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at        TIMESTAMPTZ
);

CREATE INDEX idx_conversation_user ON conversation_session(user_id, started_at);

-- Individual turns in a conversation
CREATE TABLE conversation_turn (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES conversation_session(id) ON DELETE CASCADE,
    turn_number     INTEGER NOT NULL,
    speaker         VARCHAR(10) NOT NULL CHECK (speaker IN ('learner', 'ai')),
    content_text    TEXT NOT NULL,
    audio_url       TEXT,                    -- learner's spoken audio or AI TTS audio
    -- AI feedback on learner turns
    grammar_errors  JSONB,
    -- grammar_errors example:
    -- [{"error": "yo soy come", "correction": "yo como", "rule": "verb conjugation", "explanation": "..."}]
    vocabulary_used JSONB,                   -- vocabulary items used in this turn
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_conversation_turn_session ON conversation_turn(session_id, turn_number);
```

## Proficiency Tracking

```sql
-- Continuous CEFR proficiency estimates
CREATE TABLE proficiency_estimate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    target_language_id UUID NOT NULL REFERENCES language(id),
    skill           VARCHAR(20) NOT NULL CHECK (skill IN ('reading', 'writing', 'listening', 'speaking', 'overall')),
    cefr_level      VARCHAR(10) NOT NULL,
    confidence      REAL NOT NULL CHECK (confidence BETWEEN 0 AND 1), -- model confidence
    estimated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proficiency_user_lang ON proficiency_estimate(user_id, target_language_id, skill, estimated_at);

-- Learner progress per course
CREATE TABLE learner_course_progress (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    course_id       UUID NOT NULL REFERENCES course(id),
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_activity_at TIMESTAMPTZ,
    lessons_completed INTEGER NOT NULL DEFAULT 0,
    total_study_time_seconds BIGINT NOT NULL DEFAULT 0,
    current_unit_id UUID REFERENCES unit(id),
    current_lesson_id UUID REFERENCES lesson(id),
    UNIQUE(user_id, course_id)
);

CREATE INDEX idx_learner_progress_user ON learner_course_progress(user_id);

-- Learner per-language statistics (aggregated)
CREATE TABLE learner_language_stats (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    target_language_id UUID NOT NULL REFERENCES language(id),
    words_known     INTEGER NOT NULL DEFAULT 0,
    words_learning  INTEGER NOT NULL DEFAULT 0,
    total_reviews   BIGINT NOT NULL DEFAULT 0,
    total_study_time_seconds BIGINT NOT NULL DEFAULT 0,
    conversation_count INTEGER NOT NULL DEFAULT 0,
    avg_pronunciation_score REAL,
    current_cefr_level VARCHAR(10),
    last_activity_at TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, target_language_id)
);
```

## Study Sessions & Gamification

```sql
-- Study sessions (one per app open / study block)
CREATE TABLE study_session (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    target_language_id UUID NOT NULL REFERENCES language(id),
    session_type    VARCHAR(50) NOT NULL CHECK (session_type IN ('lesson', 'review', 'conversation', 'pronunciation', 'mixed')),
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at        TIMESTAMPTZ,
    duration_seconds INTEGER,
    exercises_completed INTEGER NOT NULL DEFAULT 0,
    exercises_correct   INTEGER NOT NULL DEFAULT 0,
    xp_earned       INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX idx_study_session_user ON study_session(user_id, started_at);

-- Achievements / badges
CREATE TABLE achievement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    description     TEXT NOT NULL,
    icon_url        TEXT,
    criteria_type   VARCHAR(50) NOT NULL, -- e.g. 'streak', 'words_known', 'lessons_completed'
    criteria_value  INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE learner_achievement (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    achievement_id  UUID NOT NULL REFERENCES achievement(id),
    earned_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, achievement_id)
);
```

## Enterprise & LMS Integration

```sql
-- LTI 1.3 registrations
CREATE TABLE lti_registration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    platform_name   VARCHAR(255) NOT NULL,  -- e.g. "Canvas", "Moodle"
    issuer          VARCHAR(500) NOT NULL,   -- LTI platform issuer URL
    client_id       VARCHAR(255) NOT NULL,
    auth_endpoint   TEXT NOT NULL,
    token_endpoint  TEXT NOT NULL,
    jwks_url        TEXT NOT NULL,
    deployment_id   VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- SCORM content packages
CREATE TABLE scorm_package (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES course(id),
    scorm_version   VARCHAR(20) NOT NULL DEFAULT '2004_4th' CHECK (scorm_version IN ('2004_4th', '1.2')),
    package_url     TEXT NOT NULL,           -- URL to exported SCORM ZIP
    manifest_xml    TEXT,
    exported_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- xAPI statement storage (for LRS compatibility)
CREATE TABLE xapi_statement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id    UUID NOT NULL UNIQUE,    -- xAPI statement UUID
    actor_user_id   UUID NOT NULL REFERENCES app_user(id),
    verb_id         VARCHAR(255) NOT NULL,   -- e.g. "http://adlnet.gov/expapi/verbs/completed"
    verb_display    VARCHAR(100) NOT NULL,   -- e.g. "completed"
    object_type     VARCHAR(100) NOT NULL,   -- e.g. "Activity"
    object_id       VARCHAR(500) NOT NULL,   -- activity IRI
    object_name     VARCHAR(255),
    result_score_raw REAL,
    result_score_min REAL,
    result_score_max REAL,
    result_success  BOOLEAN,
    result_completion BOOLEAN,
    result_duration INTERVAL,
    context_json    JSONB,                   -- xAPI context object
    stored_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    timestamp_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_xapi_actor ON xapi_statement(actor_user_id, timestamp_at);
CREATE INDEX idx_xapi_verb ON xapi_statement(verb_id, timestamp_at);
CREATE INDEX idx_xapi_object ON xapi_statement(object_id, timestamp_at);
```

## Content Generation & Personalisation

```sql
-- Learner interest profiles (for personalised content generation)
CREATE TABLE learner_interest (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    interest_tag    VARCHAR(100) NOT NULL,   -- e.g. "cooking", "sports", "film", "travel"
    weight          REAL NOT NULL DEFAULT 1.0,
    PRIMARY KEY (user_id, interest_tag)
);

-- AI-generated content cache
CREATE TABLE generated_content (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_language_id UUID NOT NULL REFERENCES language(id),
    content_type    VARCHAR(50) NOT NULL CHECK (content_type IN ('reading_passage', 'dialogue', 'listening_script', 'scenario')),
    cefr_level      VARCHAR(10) NOT NULL,
    topic_tags      TEXT[] NOT NULL,         -- array of interest tags
    title           VARCHAR(255) NOT NULL,
    content_text    TEXT NOT NULL,
    audio_url       TEXT,
    model_used      VARCHAR(100),            -- LLM model that generated it
    reviewed_by     UUID REFERENCES app_user(id), -- editorial review
    is_approved     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_generated_content_lang ON generated_content(target_language_id, cefr_level);
CREATE INDEX idx_generated_content_tags ON generated_content USING GIN(topic_tags);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organisation, app_user, sso_connection |
| Language & Curriculum | 6 | language, cefr_descriptor, cefr_actfl_mapping, course, unit, lesson |
| Vocabulary & Grammar | 3 | vocabulary_item, lesson_vocabulary, grammar_rule |
| Exercises & Assessments | 2 | exercise, exercise_attempt |
| Spaced Repetition (FSRS) | 2 | review_card, review_log |
| Pronunciation | 2 | pronunciation_assessment, pronunciation_phoneme |
| AI Conversation | 2 | conversation_session, conversation_turn |
| Proficiency Tracking | 3 | proficiency_estimate, learner_course_progress, learner_language_stats |
| Study Sessions & Gamification | 3 | study_session, achievement, learner_achievement |
| Enterprise & LMS | 3 | lti_registration, scorm_package, xapi_statement |
| Content & Personalisation | 2 | learner_interest, generated_content |
| **Total** | **31** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables distributed ID generation, safe for mobile offline sync, and avoids sequential ID enumeration attacks.

2. **FSRS state stored directly on `review_card`** — the `stability`, `difficulty`, and `state` fields match the FSRS algorithm's Card struct (ts-fsrs / py-fsrs). Retrievability is computed at query time from `stability` and elapsed time since `last_review_at`, not stored.

3. **Separate `review_log` table** — immutable append-only log of every review event. This enables FSRS parameter optimisation (the FSRS optimizer requires full review history) and provides a complete audit trail of learning activity.

4. **Phoneme-level pronunciation detail in its own table** — rather than storing phoneme arrays as JSONB, each phoneme assessment gets a row. This allows SQL queries like "find the phonemes learner X consistently struggles with" without JSON path queries.

5. **xAPI statements stored relationally** — while xAPI defines a JSON format, storing the core fields (actor, verb, object, result) as indexed columns enables efficient reporting without JSON traversal. The full xAPI context is preserved in a `context_json` JSONB column for export to external LRS systems.

6. **`learner_language_stats` as a materialised aggregate** — denormalised summary table updated on each study session to avoid expensive aggregation queries on the dashboard. The source-of-truth remains the underlying detail tables.

7. **Course-Unit-Lesson hierarchy** — three-level content hierarchy mirrors CEFR's structure (level → domain → activity) and matches the organisational patterns used by Duolingo (course → section → unit → lesson), Babbel, and Rosetta Stone.

8. **LTI 1.3 and SCORM as first-class tables** — enterprise sales require LMS integration. Storing LTI registrations and SCORM packages as dedicated tables (not configuration files) enables multi-platform support per organisation.

9. **Interest tags for personalisation** — simple tag-weight model allows the AI content generator to select topics aligned with learner interests without a complex recommendation engine schema.

10. **No soft deletes by default** — rows are hard-deleted with ON DELETE CASCADE where appropriate. Audit requirements are met by the `review_log` and `xapi_statement` tables rather than soft-delete flags on every table.

---

## Example Queries

### Get a learner's due review cards

```sql
SELECT rc.id, vi.target_text, vi.source_text, rc.state, rc.stability, rc.difficulty
FROM review_card rc
JOIN vocabulary_item vi ON vi.id = rc.vocabulary_item_id
WHERE rc.user_id = :user_id
  AND rc.next_review_at <= now()
  AND rc.state IN ('learning', 'review', 'relearning')
ORDER BY rc.next_review_at ASC
LIMIT 20;
```

### Continuous CEFR level across skills

```sql
SELECT DISTINCT ON (skill)
    skill, cefr_level, confidence, estimated_at
FROM proficiency_estimate
WHERE user_id = :user_id
  AND target_language_id = :lang_id
ORDER BY skill, estimated_at DESC;
```

### Weakest phonemes for a learner

```sql
SELECT pp.phoneme,
       AVG(pp.accuracy_score) AS avg_accuracy,
       COUNT(*) AS occurrences
FROM pronunciation_phoneme pp
JOIN pronunciation_assessment pa ON pa.id = pp.assessment_id
WHERE pa.user_id = :user_id
  AND pa.target_language_id = :lang_id
GROUP BY pp.phoneme
HAVING AVG(pp.accuracy_score) < 70
ORDER BY avg_accuracy ASC;
```
