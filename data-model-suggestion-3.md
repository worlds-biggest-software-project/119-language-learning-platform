# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Language Learning Platform · Created: 2026-05-19

## Philosophy

This model uses PostgreSQL's relational strength for core identity, curriculum structure, and SRS state while leveraging JSONB columns for domain areas that vary significantly across languages, exercise types, and tenant configurations. The principle is: if a field is queried frequently, filtered on, or participates in joins, it is a relational column; if it varies by language, exercise type, or tenant — or is consumed as a blob by the frontend — it lives in JSONB.

This approach is particularly well-suited to a multilingual learning platform because language-specific data varies enormously. Japanese requires kanji/hiragana/katakana readings, pitch accent markers, and radical decomposition. Arabic requires root-pattern morphology, diacritical marks, and right-to-left rendering hints. German requires grammatical gender, case declension tables, and separable prefix markers. Rather than creating language-specific columns or tables for every language (which would explode the schema), a JSONB `language_data` column on vocabulary items and exercises absorbs this variation while the core relational fields (target_text, source_text, cefr_level, frequency_rank) remain queryable columns.

The same pattern applies to exercise payloads (different exercise types need different fields), tenant-specific configuration (enterprise customers want custom branding, custom CEFR mappings, custom achievement definitions), and AI-generated content metadata. This model minimises migration overhead while maintaining query performance on the fields that matter most.

**Best for:** Teams building a multi-language MVP rapidly, expecting high variability across languages and exercise types, and wanting to avoid frequent schema migrations.

**Trade-offs:**
- (+) Fewer tables and migrations than a fully normalized model; faster to develop
- (+) Language-specific and exercise-specific fields don't require schema changes
- (+) Tenant-specific customisation stored inline without separate config tables
- (+) PostgreSQL JSONB is indexed (GIN), queryable (@>, ->>, jsonb_path_query), and battle-tested
- (+) Easy to add new languages or exercise types without any DDL changes
- (-) JSONB fields lack database-level type enforcement; validation must happen in application code
- (-) Complex JSONB queries can be slower than relational joins on indexed columns
- (-) Schema documentation becomes critical: without explicit columns, developers must know the JSON structure
- (-) Harder to enforce referential integrity within JSONB (no foreign keys into JSON)
- (-) ORMs handle JSONB less consistently than relational columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CEFR (A1-C2) | Relational `cefr_level` columns on curriculum tables; JSONB `proficiency_data` for per-skill breakdown |
| ACTFL Proficiency Guidelines | Stored in `proficiency_data` JSONB alongside CEFR for US contexts |
| FSRS Algorithm (MIT) | `review_card` table with relational FSRS state columns (not JSONB — these are queried constantly) |
| xAPI / IEEE 9274.1.1 | Activity statements stored in `activity_log` with relational actor/verb/object and JSONB `result_data` + `context_data` |
| SCORM 2004 | SCORM manifest and tracking stored as JSONB in `integration_config` |
| IMS LTI 1.3 | LTI credentials stored in `integration_config` JSONB per organisation |
| ISO 639-1/639-3 | Relational columns on `language` table |
| Unicode / CLDR | `language.locale_data` JSONB stores CLDR-derived formatting rules per language |
| W3C Web Speech API | Pronunciation results stored in `pronunciation_assessment` with JSONB `phoneme_detail` |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    org_type        VARCHAR(50) NOT NULL DEFAULT 'individual',
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'free',
    max_seats       INTEGER,
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "branding": {"logo_url": "...", "primary_color": "#1a73e8"},
    --   "sso": {
    --     "protocol": "saml2",
    --     "entity_id": "https://idp.example.com",
    --     "metadata_xml": "..."
    --   },
    --   "lti": {
    --     "issuer": "https://canvas.example.edu",
    --     "client_id": "abc123",
    --     "auth_endpoint": "...",
    --     "jwks_url": "..."
    --   },
    --   "scorm_enabled": true,
    --   "custom_cefr_labels": {"A1": "Starter", "A2": "Elementary"},
    --   "default_daily_goal_minutes": 20
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_slug ON organisation(slug);
CREATE INDEX idx_org_config ON organisation USING GIN(config);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID REFERENCES organisation(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    native_language_id UUID,
    role            VARCHAR(50) NOT NULL DEFAULT 'learner',
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example:
    -- {
    --   "timezone": "Europe/Berlin",
    --   "daily_goal_minutes": 15,
    --   "interests": ["cooking", "sports", "film"],
    --   "notification_settings": {"daily_reminder": true, "reminder_time": "09:00"},
    --   "accessibility": {"font_size": "large", "high_contrast": false}
    -- }
    streak_days     INTEGER NOT NULL DEFAULT 0,
    streak_last_date DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_org ON app_user(organisation_id);
CREATE INDEX idx_user_email ON app_user(email);
CREATE INDEX idx_user_preferences ON app_user USING GIN(preferences);
```

## Language & Curriculum

```sql
CREATE TABLE language (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name_english    VARCHAR(100) NOT NULL,
    name_native     VARCHAR(100) NOT NULL,
    iso_639_1       CHAR(2) NOT NULL UNIQUE,
    iso_639_3       CHAR(3) NOT NULL UNIQUE,
    script          VARCHAR(50) NOT NULL DEFAULT 'Latin',
    writing_direction VARCHAR(3) NOT NULL DEFAULT 'ltr',
    locale_data     JSONB NOT NULL DEFAULT '{}',
    -- locale_data example (derived from CLDR):
    -- {
    --   "number_format": {"decimal": ",", "thousands": "."},
    --   "date_format": "dd/MM/yyyy",
    --   "phoneme_inventory": ["a", "e", "i", "o", "u", "b", "d", "f", ...],
    --   "tone_language": false,
    --   "has_grammatical_gender": true,
    --   "gender_count": 2,
    --   "case_system": false,
    --   "common_l1_difficulties": {
    --     "en": ["rolled r", "subjunctive mood", "ser vs estar"],
    --     "zh": ["gendered nouns", "verb conjugation", "articles"]
    --   }
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE course (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_language_id UUID NOT NULL REFERENCES language(id),
    source_language_id UUID NOT NULL REFERENCES language(id),
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    cefr_level_start VARCHAR(10) NOT NULL DEFAULT 'A1',
    cefr_level_end   VARCHAR(10) NOT NULL DEFAULT 'B2',
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {
    --   "cefr_descriptors": {
    --     "A1": {"reading": "Can understand familiar names...", "speaking": "..."},
    --     "A2": {...}
    --   },
    --   "actfl_mapping": {"A1": "Novice Mid", "A2": "Novice High"},
    --   "estimated_hours": 120,
    --   "author": "Linguistics Team",
    --   "version": "2.1"
    -- }
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_course_langs ON course(target_language_id, source_language_id);

CREATE TABLE unit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES course(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    cefr_level      VARCHAR(10) NOT NULL,
    sort_order      INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_unit_course ON unit(course_id, sort_order);

CREATE TABLE lesson (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id         UUID NOT NULL REFERENCES unit(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    lesson_type     VARCHAR(50) NOT NULL,
    cefr_level      VARCHAR(10) NOT NULL,
    estimated_minutes INTEGER NOT NULL DEFAULT 10,
    sort_order      INTEGER NOT NULL,
    content_data    JSONB NOT NULL DEFAULT '{}',
    -- content_data example:
    -- {
    --   "intro_text": "In this lesson you will learn to order food...",
    --   "grammar_notes": [
    --     {"title": "The conditional tense", "explanation": "...", "examples": [...]}
    --   ],
    --   "cultural_notes": "In Spain, lunch is the main meal...",
    --   "tips": ["Remember: 'quisiera' is more polite than 'quiero'"]
    -- }
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lesson_unit ON lesson(unit_id, sort_order);
```

## Vocabulary (Relational Core + JSONB Language-Specific Data)

```sql
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
    frequency_rank  INTEGER,
    language_data   JSONB NOT NULL DEFAULT '{}',
    -- language_data varies by target language:
    --
    -- Spanish example:
    -- {
    --   "gender": "feminine",
    --   "plural": "manzanas",
    --   "conjugation": null,
    --   "example_sentence": {"target": "Me gustan las manzanas", "source": "I like apples"},
    --   "related_words": ["manzano", "sidra"],
    --   "register": "neutral"
    -- }
    --
    -- Japanese example:
    -- {
    --   "kanji": "食べる",
    --   "hiragana": "たべる",
    --   "katakana": null,
    --   "pitch_accent": [0, 1, 0],
    --   "jlpt_level": "N5",
    --   "radicals": ["食"],
    --   "verb_group": "ichidan",
    --   "conjugation": {
    --     "te_form": "食べて",
    --     "ta_form": "食べた",
    --     "nai_form": "食べない",
    --     "masu_form": "食べます"
    --   },
    --   "example_sentence": {"target": "りんごを食べる", "source": "To eat an apple"}
    -- }
    --
    -- Arabic example:
    -- {
    --   "root": "ك-ت-ب",
    --   "pattern": "فَعَلَ",
    --   "with_diacritics": "كَتَبَ",
    --   "plural_type": "sound_masculine",
    --   "plural": "كُتّاب",
    --   "form": "Form I",
    --   "example_sentence": {"target": "كتب الطالب الدرس", "source": "The student wrote the lesson"}
    -- }
    --
    -- German example:
    -- {
    --   "gender": "masculine",
    --   "article": "der",
    --   "plural": "Tische",
    --   "declension": {
    --     "nominative": "der Tisch",
    --     "accusative": "den Tisch",
    --     "dative": "dem Tisch",
    --     "genitive": "des Tisches"
    --   },
    --   "example_sentence": {"target": "Der Tisch ist groß", "source": "The table is big"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vocab_lang_level ON vocabulary_item(target_language_id, cefr_level);
CREATE INDEX idx_vocab_freq ON vocabulary_item(target_language_id, frequency_rank);
CREATE INDEX idx_vocab_lang_data ON vocabulary_item USING GIN(language_data);

CREATE TABLE lesson_vocabulary (
    lesson_id       UUID NOT NULL REFERENCES lesson(id) ON DELETE CASCADE,
    vocabulary_item_id UUID NOT NULL REFERENCES vocabulary_item(id),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (lesson_id, vocabulary_item_id)
);
```

## Exercises (Polymorphic via JSONB)

```sql
CREATE TABLE exercise (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lesson_id       UUID NOT NULL REFERENCES lesson(id) ON DELETE CASCADE,
    exercise_type   VARCHAR(50) NOT NULL,
    -- exercise_type: 'multiple_choice', 'fill_blank', 'translate', 'listening',
    --   'speaking_repeat', 'speaking_freeform', 'word_order', 'matching',
    --   'dictation', 'conversation_turn', 'kanji_writing', 'tone_recognition'
    cefr_level      VARCHAR(10) NOT NULL,
    vocabulary_item_id UUID REFERENCES vocabulary_item(id),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    exercise_data   JSONB NOT NULL,
    -- exercise_data varies by exercise_type:
    --
    -- multiple_choice:
    -- {
    --   "prompt": "How do you say 'apple' in Spanish?",
    --   "correct_answer": "manzana",
    --   "alternatives": ["naranja", "plátano", "uva"],
    --   "prompt_audio_url": null
    -- }
    --
    -- fill_blank:
    -- {
    --   "sentence": "Yo ___ manzanas todos los días",
    --   "correct_answer": "como",
    --   "hint": "First person singular of 'comer'",
    --   "accept_alternatives": ["como"]
    -- }
    --
    -- speaking_repeat:
    -- {
    --   "reference_text": "Buenos días, ¿cómo estás?",
    --   "reference_audio_url": "s3://...",
    --   "phonemes_to_assess": ["ð", "ɾ"],
    --   "acceptable_accuracy": 70
    -- }
    --
    -- kanji_writing (Japanese-specific):
    -- {
    --   "kanji": "食",
    --   "stroke_order": [[...], [...], ...],
    --   "readings": {"on": ["ショク"], "kun": ["た.べる"]},
    --   "meaning": "eat",
    --   "radicals": ["食"]
    -- }
    --
    -- tone_recognition (tonal languages):
    -- {
    --   "word": "mā",
    --   "audio_url": "s3://...",
    --   "correct_tone": 1,
    --   "alternatives": [2, 3, 4],
    --   "tone_diagram_url": "..."
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_exercise_lesson ON exercise(lesson_id, sort_order);
CREATE INDEX idx_exercise_type ON exercise(exercise_type);
CREATE INDEX idx_exercise_data ON exercise USING GIN(exercise_data);
```

## Spaced Repetition (FSRS) — Relational, Not JSONB

```sql
-- FSRS state is queried too frequently and critically to be JSONB.
-- These are the hottest tables in the system.

CREATE TABLE review_card (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    vocabulary_item_id UUID NOT NULL REFERENCES vocabulary_item(id),
    state           VARCHAR(20) NOT NULL DEFAULT 'new',
    stability       REAL NOT NULL DEFAULT 0.0,
    difficulty      REAL NOT NULL DEFAULT 0.0,
    elapsed_days    INTEGER NOT NULL DEFAULT 0,
    scheduled_days  INTEGER NOT NULL DEFAULT 0,
    reps            INTEGER NOT NULL DEFAULT 0,
    lapses          INTEGER NOT NULL DEFAULT 0,
    last_review_at  TIMESTAMPTZ,
    next_review_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, vocabulary_item_id)
);

CREATE INDEX idx_review_due ON review_card(user_id, next_review_at) WHERE state != 'new';
CREATE INDEX idx_review_state ON review_card(user_id, state);

CREATE TABLE review_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_card_id  UUID NOT NULL REFERENCES review_card(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    rating          SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 4),
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

## Activity Log (Unified, with JSONB Detail)

```sql
-- Unified activity log combining exercise attempts, pronunciation,
-- conversations, and progress events. Replaces multiple detail tables.
CREATE TABLE activity_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    target_language_id UUID NOT NULL REFERENCES language(id),
    activity_type   VARCHAR(50) NOT NULL,
    -- activity_type: 'exercise_attempt', 'pronunciation_assessment',
    --   'conversation_turn', 'lesson_completed', 'review_completed',
    --   'proficiency_estimated', 'achievement_earned'
    session_id      UUID,                    -- groups activities within a study session
    lesson_id       UUID REFERENCES lesson(id),
    exercise_id     UUID REFERENCES exercise(id),
    is_correct      BOOLEAN,                 -- for exercise attempts
    xp_earned       INTEGER DEFAULT 0,
    detail          JSONB NOT NULL DEFAULT '{}',
    -- detail varies by activity_type:
    --
    -- exercise_attempt:
    -- {
    --   "user_answer": "Yo como manzanas",
    --   "correct_answer": "Yo como manzanas",
    --   "response_time_ms": 4200,
    --   "attempt_number": 1
    -- }
    --
    -- pronunciation_assessment:
    -- {
    --   "reference_text": "Buenos días",
    --   "recognised_text": "Buenos días",
    --   "scores": {"accuracy": 85.0, "fluency": 78.0, "completeness": 100.0, "prosody": 72.0, "overall": 81.0},
    --   "audio_url": "s3://...",
    --   "provider": "azure",
    --   "phonemes": [
    --     {"phoneme": "ð", "accuracy": 45.0, "word": "días", "error_type": "substitution",
    --      "guidance": "Place tongue between teeth and voice the sound"},
    --     {"phoneme": "a", "accuracy": 92.0, "word": "días"}
    --   ]
    -- }
    --
    -- conversation_turn:
    -- {
    --   "conversation_id": "uuid",
    --   "turn_number": 3,
    --   "speaker": "learner",
    --   "content": "Quisiera una ensalada",
    --   "audio_url": "s3://...",
    --   "grammar_feedback": [
    --     {"note": "Correct conditional usage", "type": "positive"}
    --   ]
    -- }
    --
    -- lesson_completed:
    -- {
    --   "exercises_completed": 12,
    --   "exercises_correct": 10,
    --   "duration_seconds": 420
    -- }
    --
    -- proficiency_estimated:
    -- {
    --   "skill": "speaking",
    --   "cefr_level": "A2",
    --   "confidence": 0.78,
    --   "actfl_level": "Novice High",
    --   "model_version": "proficiency-v3"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_activity_user_time ON activity_log(user_id, created_at);
CREATE INDEX idx_activity_type ON activity_log(user_id, activity_type, created_at);
CREATE INDEX idx_activity_session ON activity_log(session_id) WHERE session_id IS NOT NULL;
CREATE INDEX idx_activity_lang ON activity_log(user_id, target_language_id, created_at);
CREATE INDEX idx_activity_detail ON activity_log USING GIN(detail);
```

## Progress & Proficiency

```sql
-- Learner progress per course (relational — queried frequently for dashboards)
CREATE TABLE learner_progress (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    course_id       UUID NOT NULL REFERENCES course(id),
    current_unit_id UUID REFERENCES unit(id),
    current_lesson_id UUID REFERENCES lesson(id),
    lessons_completed INTEGER NOT NULL DEFAULT 0,
    total_study_time_seconds BIGINT NOT NULL DEFAULT 0,
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_activity_at TIMESTAMPTZ,
    progress_data   JSONB NOT NULL DEFAULT '{}',
    -- progress_data example:
    -- {
    --   "completed_lesson_ids": ["uuid1", "uuid2", ...],
    --   "unit_scores": {"uuid-unit-1": 92, "uuid-unit-2": 85},
    --   "words_encountered": 342,
    --   "words_mastered": 180
    -- }
    UNIQUE(user_id, course_id)
);

CREATE INDEX idx_progress_user ON learner_progress(user_id);

-- Learner per-language summary (denormalised for dashboard)
CREATE TABLE learner_language_summary (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    target_language_id UUID NOT NULL REFERENCES language(id),
    words_known     INTEGER NOT NULL DEFAULT 0,
    words_learning  INTEGER NOT NULL DEFAULT 0,
    total_reviews   BIGINT NOT NULL DEFAULT 0,
    total_study_time_seconds BIGINT NOT NULL DEFAULT 0,
    conversation_count INTEGER NOT NULL DEFAULT 0,
    proficiency     JSONB NOT NULL DEFAULT '{}',
    -- proficiency example:
    -- {
    --   "overall": {"cefr": "A2", "confidence": 0.82, "actfl": "Novice High"},
    --   "reading": {"cefr": "B1", "confidence": 0.71},
    --   "writing": {"cefr": "A2", "confidence": 0.65},
    --   "listening": {"cefr": "A2", "confidence": 0.78},
    --   "speaking": {"cefr": "A1", "confidence": 0.88},
    --   "pronunciation": {
    --     "avg_score": 76.5,
    --     "weak_phonemes": [
    --       {"phoneme": "ɾ", "avg_accuracy": 42.0, "attempts": 23},
    --       {"phoneme": "ð", "avg_accuracy": 55.0, "attempts": 18}
    --     ]
    --   },
    --   "last_estimated_at": "2026-05-19T10:30:00Z"
    -- }
    last_activity_at TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, target_language_id)
);
```

## Conversations

```sql
CREATE TABLE conversation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    target_language_id UUID NOT NULL REFERENCES language(id),
    cefr_level      VARCHAR(10) NOT NULL,
    scenario_title  VARCHAR(255),
    total_turns     INTEGER NOT NULL DEFAULT 0,
    duration_seconds INTEGER,
    summary         JSONB NOT NULL DEFAULT '{}',
    -- summary example (populated when conversation ends):
    -- {
    --   "vocabulary_used": ["quisiera", "ensalada", "cuenta"],
    --   "new_vocabulary_encountered": ["propina"],
    --   "grammar_errors_count": 2,
    --   "grammar_errors": [
    --     {"text": "yo soy querer", "correction": "yo quisiera", "rule": "conditional"}
    --   ],
    --   "topics_covered": ["ordering food", "asking for the bill"],
    --   "ai_assessment": "Good use of polite forms. Work on conditional tense."
    -- }
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at        TIMESTAMPTZ
);

CREATE INDEX idx_conversation_user ON conversation(user_id, started_at);
```

## Gamification

```sql
CREATE TABLE achievement_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    description     TEXT NOT NULL,
    icon_url        TEXT,
    criteria        JSONB NOT NULL,
    -- criteria example:
    -- {
    --   "type": "streak",
    --   "threshold": 30,
    --   "description": "Maintain a 30-day streak"
    -- }
    -- or:
    -- {
    --   "type": "compound",
    --   "conditions": [
    --     {"metric": "words_known", "threshold": 1000, "language": "any"},
    --     {"metric": "pronunciation_avg", "threshold": 80, "language": "any"}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE learner_achievement (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    achievement_id  UUID NOT NULL REFERENCES achievement_definition(id),
    earned_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    context         JSONB NOT NULL DEFAULT '{}',
    -- context example: {"language": "es", "streak_days": 30}
    PRIMARY KEY (user_id, achievement_id)
);
```

## AI Content Generation

```sql
CREATE TABLE generated_content (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_language_id UUID NOT NULL REFERENCES language(id),
    content_type    VARCHAR(50) NOT NULL,
    cefr_level      VARCHAR(10) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    content         JSONB NOT NULL,
    -- content example (reading_passage):
    -- {
    --   "text": "María fue al mercado a comprar frutas...",
    --   "translation": "Maria went to the market to buy fruit...",
    --   "audio_url": "s3://...",
    --   "vocabulary_highlights": ["mercado", "comprar", "frutas"],
    --   "comprehension_questions": [
    --     {"question": "¿Adónde fue María?", "answer": "Al mercado", "type": "factual"}
    --   ],
    --   "topic_tags": ["shopping", "food"],
    --   "model_used": "claude-4-sonnet",
    --   "generated_at": "2026-05-19T10:00:00Z"
    -- }
    is_approved     BOOLEAN NOT NULL DEFAULT false,
    reviewed_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_lang_level ON generated_content(target_language_id, cefr_level);
CREATE INDEX idx_content_type ON generated_content(content_type);
CREATE INDEX idx_content_data ON generated_content USING GIN(content);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | organisation (with SSO/LTI config in JSONB), app_user (with preferences in JSONB) |
| Language & Curriculum | 4 | language, course, unit, lesson |
| Vocabulary | 2 | vocabulary_item (with language_data JSONB), lesson_vocabulary |
| Exercises | 1 | exercise (polymorphic via exercise_data JSONB) |
| Spaced Repetition | 2 | review_card, review_log |
| Activity Log | 1 | activity_log (unified, with detail JSONB) |
| Progress & Proficiency | 2 | learner_progress, learner_language_summary |
| Conversations | 1 | conversation (with summary JSONB) |
| Gamification | 2 | achievement_definition, learner_achievement |
| AI Content | 1 | generated_content (with content JSONB) |
| **Total** | **18** | Significantly fewer tables than normalized model |

---

## Key Design Decisions

1. **JSONB for language-specific vocabulary data** — the `language_data` column on `vocabulary_item` absorbs the enormous variation between languages (Japanese readings, Arabic roots, German cases, Spanish conjugations) without language-specific tables or columns. GIN index enables containment queries (e.g., `WHERE language_data @> '{"gender": "feminine"}'`).

2. **JSONB for exercise polymorphism** — instead of exercise-type-specific tables (one for multiple choice, one for fill-blank, one for speaking), a single `exercise_data` JSONB column holds type-specific fields. This allows adding new exercise types (e.g., `kanji_writing`, `tone_recognition`) without DDL changes.

3. **Relational FSRS state** — despite the hybrid philosophy, the SRS card table uses relational columns exclusively. These fields are queried on every review session, sorted by `next_review_at`, and must be indexed efficiently. JSONB would add unnecessary overhead for the system's hottest query path.

4. **Unified `activity_log` table** — instead of separate tables for exercise attempts, pronunciation assessments, and conversation turns, a single table with an `activity_type` discriminator and `detail` JSONB captures all learner interactions. This simplifies timeline queries ("show me everything this learner did today") and reduces join complexity.

5. **Organisation `config` JSONB absorbs SSO, LTI, SCORM, and branding** — rather than separate `sso_connection`, `lti_registration`, and `scorm_package` tables, enterprise configuration lives in the organisation's `config` JSONB. This is appropriate because each organisation typically has at most one SSO provider and one LTI platform, and the configuration is read as a blob, not queried across organisations.

6. **Proficiency as a JSONB summary** — the `learner_language_summary.proficiency` JSONB stores per-skill CEFR estimates, ACTFL mappings, and pronunciation weakness data in a single document. The dashboard reads this as one blob rather than joining multiple proficiency tables. Historical proficiency data is in the `activity_log` (activity_type = 'proficiency_estimated').

7. **Conversation detail in `activity_log`, summary on `conversation`** — individual conversation turns are logged in `activity_log` for completeness, while the `conversation` table stores the high-level session with an AI-generated summary JSONB. This avoids a separate conversation_turn table.

8. **Achievement criteria as JSONB** — achievement definitions use a `criteria` JSONB that can express simple thresholds or compound conditions. This allows product managers to define new achievement types without schema changes.

9. **GIN indexes on all JSONB columns** — enables efficient containment queries (`@>`), existence checks (`?`), and path queries (`@?`) on JSONB data without full-table scans.

10. **18 tables total vs. 31 in the normalized model** — the JSONB hybrid approach nearly halves the table count, reducing migration complexity and join depth at the cost of in-application schema validation.

---

## Example Queries

### Get vocabulary items with Japanese-specific readings

```sql
SELECT target_text, source_text, pronunciation_ipa,
       language_data->>'kanji' AS kanji,
       language_data->>'hiragana' AS hiragana,
       language_data->'pitch_accent' AS pitch_accent,
       language_data->'conjugation' AS conjugation
FROM vocabulary_item
WHERE target_language_id = :japanese_id
  AND cefr_level = 'A1'
ORDER BY frequency_rank ASC
LIMIT 20;
```

### Get all feminine nouns in Spanish

```sql
SELECT target_text, source_text
FROM vocabulary_item
WHERE target_language_id = :spanish_id
  AND language_data @> '{"gender": "feminine"}'
  AND part_of_speech = 'noun'
ORDER BY frequency_rank ASC;
```

### Unified activity timeline for a learner

```sql
SELECT activity_type, is_correct, xp_earned, detail, created_at
FROM activity_log
WHERE user_id = :user_id
  AND target_language_id = :lang_id
  AND created_at >= now() - INTERVAL '7 days'
ORDER BY created_at DESC
LIMIT 50;
```

### Get learner's pronunciation weaknesses from summary

```sql
SELECT
    proficiency->'pronunciation'->'weak_phonemes' AS weak_phonemes,
    proficiency->'speaking'->>'cefr' AS speaking_level
FROM learner_language_summary
WHERE user_id = :user_id
  AND target_language_id = :lang_id;
```

### Find organisations with LTI configured

```sql
SELECT id, name, config->'lti'->>'issuer' AS lti_issuer
FROM organisation
WHERE config ? 'lti'
  AND config->'lti' ? 'issuer';
```
