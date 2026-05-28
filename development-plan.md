# Language Learning Platform — Phased Development Plan

> Project: 119-language-learning-platform
> Generated: 2026-05-25
> Status: Research complete; ready for development

---

## Table of Contents

1. [Technology Decisions](#technology-decisions)
2. [Project Structure](#project-structure)
3. [Phase Dependency Graph](#phase-dependency-graph)
4. [Phase 1: Foundation — Identity, Languages, and Curriculum Schema](#phase-1-foundation--identity-languages-and-curriculum-schema)
5. [Phase 2: Vocabulary Engine and Spaced Repetition (FSRS)](#phase-2-vocabulary-engine-and-spaced-repetition-fsrs)
6. [Phase 3: Exercise System and Lesson Delivery](#phase-3-exercise-system-and-lesson-delivery)
7. [Phase 4: Speech Recognition and Pronunciation Assessment](#phase-4-speech-recognition-and-pronunciation-assessment)
8. [Phase 5: AI Conversation Partner](#phase-5-ai-conversation-partner)
9. [Phase 6: Continuous CEFR Proficiency Estimation](#phase-6-continuous-cefr-proficiency-estimation)
10. [Phase 7: Mobile Apps (iOS and Android)](#phase-7-mobile-apps-ios-and-android)
11. [Phase 8: Gamification, Streaks, and Engagement](#phase-8-gamification-streaks-and-engagement)
12. [Phase 9: Personalised Content Generation](#phase-9-personalised-content-generation)
13. [Phase 10: Enterprise Tier — SSO, Admin, SCORM, LTI](#phase-10-enterprise-tier--sso-admin-scorm-lti)
14. [Phase 11: xAPI Learning Record Store and Analytics](#phase-11-xapi-learning-record-store-and-analytics)
15. [Phase 12: Community Features and Tutor Marketplace](#phase-12-community-features-and-tutor-marketplace)
16. [Definition of Done — Global Criteria](#definition-of-done--global-criteria)

---

## Technology Decisions

### Database: PostgreSQL 16+ with Hybrid Relational + JSONB Model

**Rationale:** Data Model Suggestion 3 (Hybrid Relational + JSONB) is the recommended starting point. The fully normalized model (Suggestion 1, 31 tables) imposes excessive migration overhead when adding language-specific fields — and language variation is the defining characteristic of this domain (Japanese kanji/readings, Arabic root morphology, German case declension). The event-sourced model (Suggestion 2) adds architectural complexity (projection handlers, eventual consistency) that is not justified at MVP scale. The hybrid model provides:
- Relational columns for frequently-queried fields (CEFR level, frequency rank, next_review_at)
- JSONB `language_data` on vocabulary items to absorb cross-language variation without schema migrations
- Relational FSRS state on `review_card` (the system's hottest query path)
- Unified `activity_log` with JSONB `detail` for exercise attempts, pronunciation assessments, and conversation turns
- 18 tables vs. 31, halving migration complexity

**Migration path:** If institutional buyers require full audit trails and temporal queries, event sourcing (Suggestion 2) can be introduced incrementally on the activity_log table by making it append-only and adding projection tables — without rewriting the curriculum or FSRS schema.

### Backend: Node.js (TypeScript) with NestJS

**Rationale:** NestJS provides a modular, decorator-based framework with first-class TypeScript support, built-in dependency injection, OpenAPI/Swagger generation, and WebSocket support for real-time conversation features. The TypeScript ecosystem has mature FSRS implementations (ts-fsrs, MIT-licensed) that can be integrated directly. NestJS's module system maps naturally to the domain boundaries (auth, curriculum, SRS, pronunciation, conversation, enterprise).

### ORM / Query Builder: Drizzle ORM

**Rationale:** Drizzle provides type-safe SQL with excellent PostgreSQL JSONB support, explicit migrations, and lower runtime overhead than Prisma. Its schema-as-code approach makes JSONB column types transparent. For complex reporting queries, raw SQL via Drizzle's `sql` template tag avoids ORM abstraction leaks.

### Frontend (Web): Next.js 15 (App Router, React Server Components)

**Rationale:** Next.js provides SSR for SEO (marketing pages, course catalog), React Server Components for data-heavy dashboard views, and client components for interactive exercises. The App Router's layout nesting maps to the course > unit > lesson > exercise navigation hierarchy.

### Mobile: React Native (Expo)

**Rationale:** Shared TypeScript codebase with the web frontend. Expo provides OTA updates (critical for iterating on lesson content without App Store review cycles), offline SQLite via expo-sqlite for offline lesson access, and expo-av for audio recording/playback. The FSRS scheduling logic (ts-fsrs) runs identically on web and mobile.

### Speech Recognition (ASR): Azure AI Speech Service (primary) + self-hosted Whisper (fallback)

**Rationale:** Azure Pronunciation Assessment is the only commercial API providing phoneme-level accuracy scores with per-phoneme error classification — which is the platform's core differentiator ("explain the articulation error, not just 'try again'"). Self-hosted Whisper (MIT license) serves as a fallback for basic transcription in markets where Azure is not available, and for offline mobile pronunciation practice where audio stays on-device.

### Text-to-Speech (TTS): ElevenLabs (primary) + Web Speech API (fallback)

**Rationale:** ElevenLabs Multilingual v3 provides high-naturalness multilingual speech with consistent voice identity across languages — suitable for a consistent AI tutor persona. The W3C Web Speech API's SpeechSynthesis serves as a zero-cost fallback for browsers that support it.

### LLM (Conversation + Content Generation): Claude API (Anthropic)

**Rationale:** The AI conversation partner and personalised content generation features require a large language model. Claude provides strong multilingual capability, long context windows for conversation history, and structured output for grammar error analysis. The API supports streaming for real-time conversation flow.

### Translation: DeepL API

**Rationale:** DeepL provides the highest-quality neural machine translation for the major language pairs, with glossary support for preserving pedagogical vocabulary choices. Used for generating bilingual vocabulary lists and localising curriculum content.

### SRS Algorithm: FSRS (Free Spaced Repetition Scheduler)

**Rationale:** FSRS is MIT-licensed, implemented in TypeScript (ts-fsrs), and empirically superior to SM-2 in retention prediction. It avoids AGPL-3.0 obligations from Anki's codebase. The algorithm's Difficulty/Stability/Retrievability model enables the platform's "adaptive SRS driven by comprehension" differentiator.

### Object Storage: S3-compatible (AWS S3 or MinIO for self-hosted)

**Rationale:** Audio recordings (learner pronunciation attempts, TTS-generated audio, native speaker reference audio) are the platform's largest storage category. S3-compatible storage provides CDN integration, lifecycle policies for old recordings, and presigned URLs for secure client-side upload.

### Authentication: NextAuth.js + custom SAML/OIDC for enterprise

**Rationale:** NextAuth.js handles consumer authentication (email/password, Google, Apple sign-in). Enterprise SSO (SAML 2.0, OIDC) is implemented via dedicated middleware for the enterprise tier.

### CI/CD: GitHub Actions

**Rationale:** Standard choice for open-source projects. Supports matrix testing across Node.js versions and PostgreSQL versions.

### Containerisation: Docker + Docker Compose (dev) / Kubernetes (prod)

**Rationale:** Docker Compose for local development with PostgreSQL, MinIO, and Redis. Kubernetes for production deployment with horizontal scaling of the API and worker services.

---

## Project Structure

```
language-learning-platform/
├── apps/
│   ├── web/                          # Next.js 15 web application
│   │   ├── src/
│   │   │   ├── app/                  # App Router pages and layouts
│   │   │   │   ├── (marketing)/      # Public pages (landing, pricing)
│   │   │   │   ├── (auth)/           # Login, register, SSO callback
│   │   │   │   ├── (learner)/        # Learner dashboard, lessons, review
│   │   │   │   ├── (admin)/          # Organisation admin dashboard
│   │   │   │   └── api/              # API route handlers
│   │   │   ├── components/           # React components
│   │   │   │   ├── exercises/        # Exercise-type-specific components
│   │   │   │   ├── pronunciation/    # Audio recorder, waveform, feedback
│   │   │   │   ├── conversation/     # Chat UI, streaming responses
│   │   │   │   ├── review/           # SRS review card UI
│   │   │   │   └── dashboard/        # Progress charts, stats
│   │   │   └── lib/                  # Client-side utilities
│   │   └── public/
│   ├── mobile/                       # React Native (Expo) app
│   │   ├── src/
│   │   │   ├── screens/
│   │   │   ├── components/
│   │   │   ├── navigation/
│   │   │   └── lib/
│   │   └── app.json
│   └── api/                          # NestJS API server
│       ├── src/
│       │   ├── modules/
│       │   │   ├── auth/             # Authentication (local, SSO)
│       │   │   ├── user/             # User management
│       │   │   ├── organisation/     # Multi-tenancy, enterprise config
│       │   │   ├── curriculum/       # Courses, units, lessons
│       │   │   ├── vocabulary/       # Vocabulary items, language data
│       │   │   ├── exercise/         # Exercise templates and attempts
│       │   │   ├── srs/              # FSRS scheduling engine
│       │   │   ├── pronunciation/    # Azure Speech / Whisper integration
│       │   │   ├── conversation/     # AI conversation partner (LLM)
│       │   │   ├── proficiency/      # CEFR estimation engine
│       │   │   ├── content-gen/      # AI content generation
│       │   │   ├── gamification/     # Streaks, achievements, XP
│       │   │   ├── enterprise/       # SSO, SCORM, LTI
│       │   │   ├── xapi/            # xAPI statement generation and LRS
│       │   │   └── community/        # Community feedback, tutor marketplace
│       │   ├── common/               # Shared guards, pipes, interceptors
│       │   └── main.ts
│       └── test/
├── packages/
│   ├── db/                           # Drizzle schema, migrations, seed
│   │   ├── schema/
│   │   ├── migrations/
│   │   └── seed/
│   ├── fsrs/                         # FSRS algorithm wrapper (ts-fsrs)
│   ├── cefr/                         # CEFR level definitions, mappings
│   ├── types/                        # Shared TypeScript types
│   └── ui/                           # Shared UI component library
├── infra/
│   ├── docker/
│   │   ├── docker-compose.yml
│   │   └── Dockerfile.*
│   └── k8s/
├── docs/
│   ├── api/                          # OpenAPI spec (generated)
│   └── architecture/
├── turbo.json                        # Turborepo config
├── package.json
└── tsconfig.base.json
```

**Monorepo tooling:** Turborepo for build orchestration across `apps/` and `packages/`. pnpm as the package manager.

---

## Phase Dependency Graph

```
Phase 1: Foundation
    │
    ├──> Phase 2: Vocabulary & SRS
    │       │
    │       ├──> Phase 3: Exercises & Lesson Delivery
    │       │       │
    │       │       ├──> Phase 4: Pronunciation
    │       │       │       │
    │       │       │       ├──> Phase 5: AI Conversation
    │       │       │       │       │
    │       │       │       │       └──> Phase 6: CEFR Estimation
    │       │       │       │
    │       │       │       └──> Phase 9: Content Generation
    │       │       │
    │       │       └──> Phase 7: Mobile Apps
    │       │
    │       └──> Phase 8: Gamification
    │
    └──> Phase 10: Enterprise (SSO, SCORM, LTI)
            │
            └──> Phase 11: xAPI & Analytics
                    │
                    └──> Phase 12: Community & Tutors
```

**Legend:**
- Arrow (──>) means "depends on" — the target phase cannot start until the source phase is complete.
- Phases 4 and 8 can run in parallel after Phase 3 is complete.
- Phase 10 depends only on Phase 1 and can run in parallel with Phases 2-9 if an enterprise-focused team is available.
- Phase 12 is the final phase and depends on all prior phases being stable.

---

## Phase 1: Foundation — Identity, Languages, and Curriculum Schema

**Goal:** Establish the database, authentication system, user management, language catalog, and course/unit/lesson content hierarchy. At the end of this phase, an authenticated user can browse a catalog of courses and see an empty lesson shell.

### Task 1.1: Monorepo and Infrastructure Setup

**What:** Initialize the Turborepo monorepo with `apps/web`, `apps/api`, `packages/db`, `packages/types`. Configure Docker Compose with PostgreSQL 16, Redis, and MinIO. Set up CI with GitHub Actions running lint, type-check, and test on every PR.

**Design:**

```typescript
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": [".next/**", "dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "lint": {},
    "typecheck": {},
    "test": { "dependsOn": ["^build"] },
    "db:migrate": { "cache": false },
    "db:seed": { "cache": false, "dependsOn": ["db:migrate"] }
  }
}
```

```yaml
# infra/docker/docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: langlearn
      POSTGRES_USER: langlearn
      POSTGRES_PASSWORD: langlearn_dev
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports: ["9000:9000", "9001:9001"]
    volumes: ["miniodata:/data"]

volumes:
  pgdata:
  miniodata:
```

**Testing:**
- `pnpm install` succeeds in CI with zero warnings
- `docker compose up -d` starts all three services; health checks pass
- `pnpm turbo build` completes for all packages and apps
- GitHub Actions workflow runs lint + typecheck + test on a PR and reports status

### Task 1.2: Database Schema — Core Identity and Curriculum Tables

**What:** Create the Drizzle schema for `organisation`, `app_user`, `language`, `course`, `unit`, `lesson`, `vocabulary_item`, and `lesson_vocabulary` tables. Write the initial migration. Seed with 10 languages (Spanish, French, German, Italian, Portuguese, Japanese, Korean, Mandarin, Arabic, Russian).

**Design:**

```typescript
// packages/db/schema/language.ts
import { pgTable, uuid, varchar, char, boolean, timestamp, jsonb } from 'drizzle-orm/pg-core';

export const language = pgTable('language', {
  id: uuid('id').primaryKey().defaultRandom(),
  nameEnglish: varchar('name_english', { length: 100 }).notNull(),
  nameNative: varchar('name_native', { length: 100 }).notNull(),
  iso6391: char('iso_639_1', { length: 2 }).notNull().unique(),
  iso6393: char('iso_639_3', { length: 3 }).notNull().unique(),
  script: varchar('script', { length: 50 }).notNull().default('Latin'),
  writingDirection: varchar('writing_direction', { length: 3 }).notNull().default('ltr'),
  localeData: jsonb('locale_data').notNull().default({}),
  isActive: boolean('is_active').notNull().default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

// packages/db/schema/user.ts
export const appUser = pgTable('app_user', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').references(() => organisation.id),
  email: varchar('email', { length: 255 }).notNull().unique(),
  displayName: varchar('display_name', { length: 255 }).notNull(),
  passwordHash: varchar('password_hash', { length: 255 }),
  nativeLanguageId: uuid('native_language_id').references(() => language.id),
  role: varchar('role', { length: 50 }).notNull().default('learner'),
  authProvider: varchar('auth_provider', { length: 50 }).notNull().default('local'),
  preferences: jsonb('preferences').notNull().default({}),
  streakDays: integer('streak_days').notNull().default(0),
  streakLastDate: date('streak_last_date'),
  isActive: boolean('is_active').notNull().default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

**Testing:**
- `pnpm db:migrate` applies cleanly on a fresh PostgreSQL instance
- `pnpm db:migrate` is idempotent (running twice produces no errors)
- `pnpm db:seed` inserts 10 languages with correct ISO codes and writing directions
- Query `SELECT * FROM language WHERE writing_direction = 'rtl'` returns Arabic
- Query `SELECT * FROM language WHERE script = 'Han'` returns Mandarin and Japanese (Japanese uses Han + Kana)
- Foreign key constraints are enforced: inserting an `app_user` with a non-existent `organisation_id` fails

### Task 1.3: Authentication — Local, Google, Apple Sign-In

**What:** Implement NextAuth.js with email/password (bcrypt), Google OAuth, and Apple Sign-In providers. Create registration and login pages. Implement JWT access tokens (15-minute expiry) and refresh tokens (30-day expiry) for the NestJS API.

**Design:**

```typescript
// apps/web/src/app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';
import GoogleProvider from 'next-auth/providers/google';
import AppleProvider from 'next-auth/providers/apple';
import { DrizzleAdapter } from '@auth/drizzle-adapter';
import { db } from '@langlearn/db';
import bcrypt from 'bcryptjs';

export const authOptions = {
  adapter: DrizzleAdapter(db),
  providers: [
    CredentialsProvider({
      name: 'credentials',
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        const user = await db.query.appUser.findFirst({
          where: eq(appUser.email, credentials.email),
        });
        if (!user || !user.passwordHash) return null;
        const valid = await bcrypt.compare(credentials.password, user.passwordHash);
        return valid ? { id: user.id, email: user.email, name: user.displayName } : null;
      },
    }),
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
    AppleProvider({
      clientId: process.env.APPLE_CLIENT_ID!,
      clientSecret: process.env.APPLE_CLIENT_SECRET!,
    }),
  ],
  session: { strategy: 'jwt', maxAge: 30 * 24 * 60 * 60 },
  callbacks: {
    async jwt({ token, user }) {
      if (user) { token.userId = user.id; token.role = user.role; }
      return token;
    },
    async session({ session, token }) {
      session.user.id = token.userId;
      session.user.role = token.role;
      return session;
    },
  },
};
```

**Testing:**
- Register a new user with email/password; verify `password_hash` is bcrypt in DB; login succeeds
- Register with an existing email returns 409 Conflict
- Login with wrong password returns 401
- Google OAuth flow redirects, returns to app, and creates a user with `auth_provider = 'google'`
- JWT token contains `userId` and `role` claims
- Expired JWT (>15 min) is rejected by API; refresh token issues a new access token
- SQL injection attempt in email field is rejected by input validation

### Task 1.4: NestJS API — User and Course CRUD

**What:** Implement NestJS modules for `user`, `organisation`, and `curriculum` with REST endpoints. Generate OpenAPI 3.2 documentation. Implement role-based access control (RBAC) guards.

**Design:**

```typescript
// apps/api/src/modules/curriculum/curriculum.controller.ts
@ApiTags('Curriculum')
@Controller('api/v1/courses')
export class CurriculumController {
  constructor(private readonly curriculumService: CurriculumService) {}

  @Get()
  @ApiOperation({ summary: 'List published courses' })
  async listCourses(
    @Query('targetLanguage') targetLanguage?: string,
    @Query('sourceLanguage') sourceLanguage?: string,
  ): Promise<CourseListDto[]> {
    return this.curriculumService.listPublished({ targetLanguage, sourceLanguage });
  }

  @Get(':courseId')
  @ApiOperation({ summary: 'Get course with units and lessons' })
  async getCourse(@Param('courseId', ParseUUIDPipe) courseId: string): Promise<CourseDetailDto> {
    return this.curriculumService.getWithStructure(courseId);
  }

  @Post()
  @UseGuards(AuthGuard, RolesGuard)
  @Roles('admin', 'super_admin')
  @ApiOperation({ summary: 'Create a course (admin only)' })
  async createCourse(@Body() dto: CreateCourseDto): Promise<CourseDetailDto> {
    return this.curriculumService.create(dto);
  }
}
```

**Testing:**
- `GET /api/v1/courses` returns an empty list initially; returns seeded courses after seed
- `GET /api/v1/courses/:id` returns course with nested units and lessons
- `POST /api/v1/courses` without auth returns 401
- `POST /api/v1/courses` with learner role returns 403
- `POST /api/v1/courses` with admin role creates course; verify in DB
- OpenAPI spec at `/api/docs` renders all endpoints with correct schemas
- Invalid UUID in path parameter returns 400 with descriptive error

### Task 1.5: Web Frontend — Course Catalog and Lesson Shell

**What:** Build the Next.js web app with a marketing landing page, course catalog page, and a lesson detail page (content placeholder). Implement the learner dashboard layout with sidebar navigation.

**Design:**

```typescript
// apps/web/src/app/(learner)/courses/page.tsx
export default async function CourseCatalogPage() {
  const courses = await fetch(`${API_URL}/api/v1/courses`).then(r => r.json());

  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
      {courses.map((course: CourseListDto) => (
        <CourseCard
          key={course.id}
          title={course.title}
          targetLanguage={course.targetLanguageName}
          cefrRange={`${course.cefrLevelStart}–${course.cefrLevelEnd}`}
          lessonCount={course.lessonCount}
          href={`/courses/${course.id}`}
        />
      ))}
    </div>
  );
}
```

**Testing:**
- Landing page renders at `/` with marketing content; Lighthouse performance score > 90
- `/courses` displays course cards with language names, CEFR range, and lesson count
- `/courses/:id` displays course structure (units and lessons) in a collapsible tree
- Unauthenticated users can browse catalog; clicking a lesson redirects to login
- Responsive layout works at 320px, 768px, 1024px, and 1440px widths
- Dark mode toggle works without FOUC (flash of unstyled content)

### Phase 1 Definition of Done

- [ ] Monorepo builds and passes CI (lint, typecheck, test)
- [ ] PostgreSQL schema deployed with 10 seeded languages
- [ ] User registration and login work via email, Google, and Apple
- [ ] Course CRUD API documented in OpenAPI 3.2
- [ ] Course catalog renders on web with responsive layout
- [ ] RBAC enforced: learners cannot create courses; admins can
- [ ] Docker Compose runs the full stack locally with one command

---

## Phase 2: Vocabulary Engine and Spaced Repetition (FSRS)

**Goal:** Implement the vocabulary item store with language-specific JSONB data, the FSRS scheduling algorithm, and the review card system. At the end of this phase, a learner can review vocabulary flashcards with adaptive scheduling.

### Task 2.1: Vocabulary Item Schema and API

**What:** Create the `vocabulary_item` table with JSONB `language_data`, the `lesson_vocabulary` junction table, and CRUD endpoints for vocabulary management. Implement vocabulary import from CSV/JSON for batch loading.

**Design:**

```typescript
// packages/db/schema/vocabulary.ts
export const vocabularyItem = pgTable('vocabulary_item', {
  id: uuid('id').primaryKey().defaultRandom(),
  targetLanguageId: uuid('target_language_id').notNull().references(() => language.id),
  sourceLanguageId: uuid('source_language_id').notNull().references(() => language.id),
  targetText: varchar('target_text', { length: 500 }).notNull(),
  sourceText: varchar('source_text', { length: 500 }).notNull(),
  pronunciationIpa: varchar('pronunciation_ipa', { length: 255 }),
  audioUrl: text('audio_url'),
  partOfSpeech: varchar('part_of_speech', { length: 50 }),
  cefrLevel: varchar('cefr_level', { length: 10 }).notNull(),
  frequencyRank: integer('frequency_rank'),
  languageData: jsonb('language_data').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  langLevelIdx: index('idx_vocab_lang_level').on(table.targetLanguageId, table.cefrLevel),
  freqIdx: index('idx_vocab_freq').on(table.targetLanguageId, table.frequencyRank),
  langDataIdx: index('idx_vocab_lang_data').using('gin', table.languageData),
}));
```

```typescript
// apps/api/src/modules/vocabulary/vocabulary.service.ts
async importBatch(items: VocabularyImportDto[]): Promise<{ imported: number; errors: ValidationError[] }> {
  const errors: ValidationError[] = [];
  const valid: typeof vocabularyItem.$inferInsert[] = [];

  for (const item of items) {
    const validation = vocabularyImportSchema.safeParse(item);
    if (!validation.success) {
      errors.push({ item: item.targetText, issues: validation.error.issues });
      continue;
    }
    valid.push(validation.data);
  }

  if (valid.length > 0) {
    await db.insert(vocabularyItem).values(valid).onConflictDoNothing();
  }

  return { imported: valid.length, errors };
}
```

**Testing:**
- Import 100 Spanish vocabulary items from CSV; verify all 100 inserted with correct CEFR levels
- Import Japanese vocabulary with `language_data` containing kanji, hiragana, and pitch accent; verify JSONB stored correctly
- Query `WHERE language_data @> '{"gender": "feminine"}'` returns only feminine Spanish nouns
- Import with duplicate target_text is idempotent (ON CONFLICT DO NOTHING)
- Import with missing required field (target_text) returns validation error for that row; valid rows still import
- `GET /api/v1/vocabulary?language=es&cefrLevel=A1` returns paginated results sorted by frequency_rank

### Task 2.2: FSRS Algorithm Integration

**What:** Integrate the ts-fsrs library into a `packages/fsrs` wrapper that exposes `scheduleReview(card, rating) -> scheduledCard` and `getOptimalParameters(reviewHistory) -> FSRSParameters`. Create the `review_card` and `review_log` tables.

**Design:**

```typescript
// packages/fsrs/src/index.ts
import { createEmptyCard, fsrs, generatorParameters, Rating, Card, State } from 'ts-fsrs';

export class FSRSEngine {
  private scheduler;

  constructor(params?: Partial<FSRSParameters>) {
    const p = generatorParameters(params);
    this.scheduler = fsrs(p);
  }

  scheduleReview(card: Card, rating: Rating, now?: Date): ScheduleResult {
    const result = this.scheduler.repeat(card, now ?? new Date());
    const scheduled = result[rating];
    return {
      card: scheduled.card,
      log: scheduled.log,
      nextReviewAt: scheduled.card.due,
    };
  }

  createNewCard(): Card {
    return createEmptyCard();
  }

  static ratingFromScore(score: number): Rating {
    if (score <= 0.3) return Rating.Again;
    if (score <= 0.6) return Rating.Hard;
    if (score <= 0.85) return Rating.Good;
    return Rating.Easy;
  }
}
```

```typescript
// packages/db/schema/review.ts
export const reviewCard = pgTable('review_card', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => appUser.id),
  vocabularyItemId: uuid('vocabulary_item_id').notNull().references(() => vocabularyItem.id),
  state: varchar('state', { length: 20 }).notNull().default('new'),
  stability: real('stability').notNull().default(0.0),
  difficulty: real('difficulty').notNull().default(0.0),
  elapsedDays: integer('elapsed_days').notNull().default(0),
  scheduledDays: integer('scheduled_days').notNull().default(0),
  reps: integer('reps').notNull().default(0),
  lapses: integer('lapses').notNull().default(0),
  lastReviewAt: timestamp('last_review_at', { withTimezone: true }),
  nextReviewAt: timestamp('next_review_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  userVocabUnique: uniqueIndex('uq_review_card_user_vocab').on(table.userId, table.vocabularyItemId),
  dueIdx: index('idx_review_due').on(table.userId, table.nextReviewAt),
}));
```

**Testing:**
- New card rated "Good" transitions from state `new` to `learning` with stability > 0
- Card rated "Again" in `review` state transitions to `relearning` with decreased stability
- Card rated "Easy" has longer `scheduled_days` than card rated "Good"
- After 10 simulated reviews with "Good", stability exceeds 30 days
- `review_log` contains one entry per review with correct before/after state and stability
- `FSRSEngine.ratingFromScore(0.1)` returns `Rating.Again`; `(0.5)` returns `Rating.Hard`; `(0.75)` returns `Rating.Good`; `(0.95)` returns `Rating.Easy`
- Concurrent review submissions for the same card are serialized (no lost updates)

### Task 2.3: Review Session API and Web UI

**What:** Build the review session flow: fetch due cards, present flashcard UI, submit rating, receive next card. Implement session tracking in `activity_log`.

**Design:**

```typescript
// apps/api/src/modules/srs/srs.controller.ts
@Controller('api/v1/review')
export class SRSController {
  @Get('due')
  @UseGuards(AuthGuard)
  async getDueCards(
    @CurrentUser() user: UserPayload,
    @Query('language') languageId: string,
    @Query('limit', new DefaultValuePipe(20), ParseIntPipe) limit: number,
  ): Promise<DueCardDto[]> {
    return this.srsService.getDueCards(user.id, languageId, limit);
  }

  @Post('rate')
  @UseGuards(AuthGuard)
  async rateCard(
    @CurrentUser() user: UserPayload,
    @Body() dto: RateCardDto,
  ): Promise<RateCardResponseDto> {
    return this.srsService.rateCard(user.id, dto.cardId, dto.rating);
  }
}
```

```tsx
// apps/web/src/components/review/ReviewCard.tsx
export function ReviewCard({ card, onRate }: ReviewCardProps) {
  const [revealed, setRevealed] = useState(false);

  return (
    <div className="max-w-lg mx-auto">
      <div className="card bg-white rounded-xl shadow-lg p-8 text-center min-h-[300px] flex items-center justify-center">
        {!revealed ? (
          <button onClick={() => setRevealed(true)} className="text-2xl font-semibold">
            {card.targetText}
            {card.pronunciationIpa && (
              <span className="block text-sm text-gray-500 mt-2">[{card.pronunciationIpa}]</span>
            )}
          </button>
        ) : (
          <div>
            <p className="text-2xl font-semibold">{card.targetText}</p>
            <p className="text-lg text-gray-600 mt-4">{card.sourceText}</p>
          </div>
        )}
      </div>
      {revealed && (
        <div className="flex gap-3 mt-6 justify-center">
          <RatingButton label="Again" rating={1} color="red" onClick={() => onRate(1)} />
          <RatingButton label="Hard" rating={2} color="orange" onClick={() => onRate(2)} />
          <RatingButton label="Good" rating={3} color="green" onClick={() => onRate(3)} />
          <RatingButton label="Easy" rating={4} color="blue" onClick={() => onRate(4)} />
        </div>
      )}
    </div>
  );
}
```

**Testing:**
- `GET /api/v1/review/due?language=:id` returns only cards where `next_review_at <= now()`
- After rating a card "Good", it no longer appears in due cards (next_review_at is in the future)
- Review UI displays target text; clicking reveals source text and rating buttons
- Rating a card sends POST and loads the next card without full page reload
- Session ends when no more due cards; UI displays "All caught up" message
- Activity log records `activity_type = 'review_completed'` with session duration
- Keyboard shortcuts (1-4) work for rating without mouse

### Phase 2 Definition of Done

- [ ] Vocabulary items stored with language-specific JSONB data; batch import works
- [ ] FSRS algorithm correctly schedules reviews with state transitions
- [ ] Review log maintains complete, immutable history of all ratings
- [ ] Web UI presents flashcard review with four rating buttons
- [ ] Due card query uses index scan (verified via EXPLAIN ANALYZE)
- [ ] 100+ vocabulary items seeded for Spanish A1

---

## Phase 3: Exercise System and Lesson Delivery

**Goal:** Implement the polymorphic exercise system and lesson delivery flow. At the end of this phase, a learner can complete a structured lesson containing multiple exercise types (multiple choice, fill-in-the-blank, translation, word ordering, matching) and have their responses scored.

### Task 3.1: Exercise Schema and Exercise Engine

**What:** Create the `exercise` table with JSONB `exercise_data` for polymorphic exercise types. Build the exercise engine that validates answers, scores responses, and determines exercise progression within a lesson.

**Design:**

```typescript
// apps/api/src/modules/exercise/engines/exercise-engine.factory.ts
export class ExerciseEngineFactory {
  static create(exerciseType: string): ExerciseEngine {
    switch (exerciseType) {
      case 'multiple_choice': return new MultipleChoiceEngine();
      case 'fill_blank': return new FillBlankEngine();
      case 'translate_to_target': return new TranslationEngine('to_target');
      case 'translate_to_source': return new TranslationEngine('to_source');
      case 'word_order': return new WordOrderEngine();
      case 'matching': return new MatchingEngine();
      case 'dictation': return new DictationEngine();
      default: throw new UnsupportedExerciseTypeError(exerciseType);
    }
  }
}

// apps/api/src/modules/exercise/engines/fill-blank.engine.ts
export class FillBlankEngine implements ExerciseEngine {
  evaluate(exercise: ExerciseData, userAnswer: string): ExerciseResult {
    const normalized = userAnswer.trim().toLowerCase();
    const correct = exercise.exercise_data.correct_answer.trim().toLowerCase();
    const alternatives = exercise.exercise_data.accept_alternatives?.map(
      (a: string) => a.trim().toLowerCase()
    ) ?? [];

    const isCorrect = normalized === correct || alternatives.includes(normalized);

    return {
      isCorrect,
      correctAnswer: exercise.exercise_data.correct_answer,
      feedback: isCorrect
        ? null
        : `The correct answer is "${exercise.exercise_data.correct_answer}". ${exercise.exercise_data.hint ?? ''}`,
    };
  }
}
```

**Testing:**
- Multiple choice: selecting correct answer returns `isCorrect: true`; selecting wrong answer returns `isCorrect: false` with correct answer shown
- Fill-in-the-blank: answer matching is case-insensitive and trims whitespace
- Fill-in-the-blank: accepted alternatives (e.g., "don't" vs "do not") are both marked correct
- Translation: DeepL-assisted fuzzy matching scores translations within acceptable Levenshtein distance
- Word order: correctly ordered words score 100%; each misplacement reduces score proportionally
- Matching: all correct pairs required for full score; partial credit for partial matches
- Unsupported exercise type throws `UnsupportedExerciseTypeError`

### Task 3.2: Lesson Delivery Flow

**What:** Build the lesson delivery API that sequences exercises within a lesson, tracks progress, handles retry on incorrect answers, and records completion. Build the lesson UI with exercise progression, progress bar, and completion summary.

**Design:**

```typescript
// apps/api/src/modules/exercise/lesson-session.service.ts
export class LessonSessionService {
  async startLesson(userId: string, lessonId: string): Promise<LessonSessionDto> {
    const exercises = await db.query.exercise.findMany({
      where: eq(exercise.lessonId, lessonId),
      orderBy: asc(exercise.sortOrder),
    });

    const sessionId = randomUUID();
    await db.insert(activityLog).values({
      userId,
      targetLanguageId: exercises[0]?.targetLanguageId,
      activityType: 'lesson_started',
      sessionId,
      lessonId,
      detail: { exerciseCount: exercises.length },
    });

    return {
      sessionId,
      lessonId,
      exercises: exercises.map(this.toExercisePresentation),
      totalExercises: exercises.length,
      currentIndex: 0,
    };
  }

  async submitAnswer(
    userId: string,
    sessionId: string,
    exerciseId: string,
    answer: string,
  ): Promise<AnswerResultDto> {
    const ex = await db.query.exercise.findFirst({ where: eq(exercise.id, exerciseId) });
    const engine = ExerciseEngineFactory.create(ex.exerciseType);
    const result = engine.evaluate(ex, answer);

    await db.insert(activityLog).values({
      userId,
      targetLanguageId: ex.targetLanguageId,
      activityType: 'exercise_attempt',
      sessionId,
      exerciseId,
      isCorrect: result.isCorrect,
      detail: {
        userAnswer: answer,
        correctAnswer: result.correctAnswer,
        responseTimeMs: result.responseTimeMs,
      },
    });

    // If correct and vocab item exists, create or update review card
    if (result.isCorrect && ex.vocabularyItemId) {
      await this.srsService.ensureCardExists(userId, ex.vocabularyItemId);
    }

    return result;
  }
}
```

```tsx
// apps/web/src/app/(learner)/lessons/[lessonId]/page.tsx
'use client';
export default function LessonPage({ params }: { params: { lessonId: string } }) {
  const [session, setSession] = useState<LessonSession | null>(null);
  const [currentIndex, setCurrentIndex] = useState(0);
  const [results, setResults] = useState<AnswerResult[]>([]);

  // ... exercise rendering by type, progress bar, completion summary
  const currentExercise = session?.exercises[currentIndex];

  return (
    <div className="max-w-2xl mx-auto py-8">
      <ProgressBar current={currentIndex + 1} total={session?.totalExercises ?? 0} />
      {currentExercise && (
        <ExerciseRenderer
          exercise={currentExercise}
          onSubmit={handleSubmit}
        />
      )}
      {currentIndex >= (session?.totalExercises ?? 0) && (
        <LessonCompleteSummary results={results} />
      )}
    </div>
  );
}
```

**Testing:**
- Starting a lesson returns all exercises in sort_order sequence
- Submitting a correct answer advances to the next exercise
- Submitting an incorrect answer shows feedback and allows retry (up to 3 attempts)
- Progress bar accurately reflects position within lesson
- Completing all exercises shows summary with score (correct/total), XP earned, and time taken
- `activity_log` contains `lesson_started` and `exercise_attempt` entries for the session
- Vocabulary items from correct exercises have `review_card` entries created in `new` state
- Closing the browser mid-lesson and returning resumes from the last unanswered exercise (session state in Redis)

### Task 3.3: Curriculum Seeding — Spanish A1 Course

**What:** Create a complete Spanish for English Speakers A1 course with 5 units, 25 lessons, and 250 exercises covering vocabulary, grammar, translation, and fill-in-the-blank types. Seed 500 vocabulary items.

**Design:**

```typescript
// packages/db/seed/spanish-a1.ts
const units = [
  { title: 'Greetings & Introductions', cefrLevel: 'A1', lessons: [
    { title: 'Hello & Goodbye', type: 'vocabulary', exercises: [
      { type: 'multiple_choice', data: { prompt: 'How do you say "Hello" in Spanish?', correct_answer: 'Hola', alternatives: ['Adiós', 'Gracias', 'Por favor'] } },
      { type: 'fill_blank', data: { sentence: '_____, me llamo María.', correct_answer: 'Hola', hint: 'A greeting' } },
      // ... 8 more exercises per lesson
    ]},
    // ... 4 more lessons per unit
  ]},
  // ... 4 more units
];
```

**Testing:**
- Seed script completes without errors; database contains 5 units, 25 lessons, 250 exercises, 500 vocabulary items
- Every exercise has a valid `exercise_type` and well-formed `exercise_data`
- Every vocabulary item has a valid `cefr_level` of 'A1'
- Every lesson has at least 8 exercises
- A learner can complete the full "Greetings & Introductions" unit start to finish without errors

### Phase 3 Definition of Done

- [ ] Six exercise types implemented with correct scoring logic
- [ ] Lesson delivery flow tracks progress, handles retries, and records completion
- [ ] Complete Spanish A1 course seeded with 25 lessons and 250 exercises
- [ ] Exercise attempts recorded in activity_log
- [ ] Correct exercises create SRS review cards automatically
- [ ] Lesson completion summary shows score, XP, and time

---

## Phase 4: Speech Recognition and Pronunciation Assessment

**Goal:** Integrate Azure Pronunciation Assessment for phoneme-level pronunciation feedback. At the end of this phase, learners can record speech, receive per-phoneme accuracy scores with corrective guidance, and practice speaking exercises.

### Task 4.1: Audio Recording and Upload

**What:** Implement browser-based audio recording using the MediaRecorder API. Upload recordings to S3-compatible storage via presigned URLs. Build the audio playback component for reviewing recordings.

**Design:**

```typescript
// apps/web/src/lib/audio-recorder.ts
export class AudioRecorder {
  private mediaRecorder: MediaRecorder | null = null;
  private chunks: Blob[] = [];

  async start(): Promise<void> {
    const stream = await navigator.mediaDevices.getUserMedia({
      audio: { channelCount: 1, sampleRate: 16000 },
    });
    this.mediaRecorder = new MediaRecorder(stream, { mimeType: 'audio/webm;codecs=opus' });
    this.chunks = [];
    this.mediaRecorder.ondataavailable = (e) => this.chunks.push(e.data);
    this.mediaRecorder.start();
  }

  async stop(): Promise<Blob> {
    return new Promise((resolve) => {
      this.mediaRecorder!.onstop = () => {
        const blob = new Blob(this.chunks, { type: 'audio/webm' });
        resolve(blob);
      };
      this.mediaRecorder!.stop();
      this.mediaRecorder!.stream.getTracks().forEach(t => t.stop());
    });
  }
}

// apps/api/src/modules/pronunciation/upload.controller.ts
@Post('presign')
async getPresignedUrl(@CurrentUser() user: UserPayload): Promise<{ uploadUrl: string; key: string }> {
  const key = `audio/${user.id}/${randomUUID()}.webm`;
  const uploadUrl = await this.s3Service.getPresignedPutUrl(key, 'audio/webm', 60);
  return { uploadUrl, key };
}
```

**Testing:**
- Microphone permission prompt appears on first recording attempt
- Recording produces a valid WebM/Opus blob; playback works in the audio component
- Presigned URL upload succeeds; file is retrievable from S3
- Recording duration is capped at 30 seconds; UI shows recording indicator and timer
- Recording with no microphone access shows a clear error message (not a crash)
- Audio file size for a 10-second recording is under 200KB

### Task 4.2: Azure Pronunciation Assessment Integration

**What:** Integrate Azure AI Speech Service's Pronunciation Assessment API. Send learner audio with reference text; receive per-word and per-phoneme accuracy scores. Store results in `activity_log` with phoneme-level detail in the JSONB `detail` column.

**Design:**

```typescript
// apps/api/src/modules/pronunciation/azure-pronunciation.service.ts
import * as sdk from 'microsoft-cognitiveservices-speech-sdk';

export class AzurePronunciationService {
  async assess(
    audioBuffer: Buffer,
    referenceText: string,
    language: string, // BCP-47 tag, e.g. 'es-ES'
  ): Promise<PronunciationResult> {
    const speechConfig = sdk.SpeechConfig.fromSubscription(
      this.configService.get('AZURE_SPEECH_KEY'),
      this.configService.get('AZURE_SPEECH_REGION'),
    );
    speechConfig.speechRecognitionLanguage = language;

    const pronunciationConfig = new sdk.PronunciationAssessmentConfig(
      referenceText,
      sdk.PronunciationAssessmentGradingSystem.HundredMark,
      sdk.PronunciationAssessmentGranularity.Phoneme,
      true, // enable miscue
    );
    pronunciationConfig.enableProsodyAssessment();

    const audioConfig = sdk.AudioConfig.fromWavFileInput(audioBuffer);
    const recognizer = new sdk.SpeechRecognizer(speechConfig, audioConfig);
    pronunciationConfig.applyTo(recognizer);

    return new Promise((resolve, reject) => {
      recognizer.recognizeOnceAsync((result) => {
        const assessment = sdk.PronunciationAssessmentResult.fromResult(result);
        const words = assessment.detailResult.Words.map(word => ({
          word: word.Word,
          accuracyScore: word.PronunciationAssessment.AccuracyScore,
          phonemes: word.Phonemes.map(p => ({
            phoneme: p.Phoneme,
            accuracyScore: p.PronunciationAssessment.AccuracyScore,
          })),
        }));

        resolve({
          recognisedText: result.text,
          accuracyScore: assessment.accuracyScore,
          fluencyScore: assessment.fluencyScore,
          completenessScore: assessment.completenessScore,
          prosodyScore: assessment.prosodyScore,
          words,
        });
        recognizer.close();
      }, reject);
    });
  }
}
```

**Testing:**
- Submit a native-quality Spanish recording of "Buenos dias" against reference text; accuracy score > 90
- Submit an English-accented recording; accuracy score < 70; specific phonemes flagged with low scores
- Phoneme `/ɾ/` (Spanish flap r) is correctly identified and scored separately from `/r/`
- Completeness score drops when words are omitted from the reference text
- Prosody score reflects intonation patterns (question vs. statement)
- Azure API timeout (>10 seconds) returns a graceful error, not a crash
- Results are stored in `activity_log` with `activity_type = 'pronunciation_assessment'` and full phoneme detail in `detail` JSONB

### Task 4.3: Phoneme Feedback and Corrective Guidance

**What:** Build the phoneme feedback UI that displays per-phoneme scores color-coded by accuracy, with corrective guidance for low-scoring phonemes. Implement L1-L2 phoneme contrast explanations using the language's `locale_data.common_l1_difficulties`.

**Design:**

```typescript
// apps/api/src/modules/pronunciation/feedback.service.ts
export class PronunciationFeedbackService {
  async generateGuidance(
    phoneme: string,
    targetLanguageId: string,
    nativeLanguageId: string,
    accuracyScore: number,
  ): Promise<PhonemeGuidance | null> {
    if (accuracyScore >= 80) return null; // No guidance needed

    const targetLang = await this.getLanguage(targetLanguageId);
    const nativeLang = await this.getLanguage(nativeLanguageId);

    // Look up L1 interference patterns from locale_data
    const l1Code = nativeLang.iso6391;
    const difficulties = targetLang.localeData?.common_l1_difficulties?.[l1Code] ?? [];

    // Generate articulatory guidance via LLM for specific phoneme errors
    const guidance = await this.llmService.generatePhonemeGuidance({
      phoneme,
      targetLanguage: targetLang.nameEnglish,
      nativeLanguage: nativeLang.nameEnglish,
      knownDifficulties: difficulties,
    });

    return {
      phoneme,
      accuracyScore,
      articulatoryGuidance: guidance.articulatoryDescription,
      commonMistake: guidance.commonL1Substitution,
      practiceWord: guidance.practiceWord,
    };
  }
}
```

```tsx
// apps/web/src/components/pronunciation/PhonemeDisplay.tsx
export function PhonemeDisplay({ phonemes }: { phonemes: PhonemeResult[] }) {
  return (
    <div className="flex flex-wrap gap-1 justify-center my-4">
      {phonemes.map((p, i) => (
        <span
          key={i}
          className={cn(
            'inline-block px-2 py-1 rounded text-lg font-mono',
            p.accuracyScore >= 80 ? 'bg-green-100 text-green-800' :
            p.accuracyScore >= 50 ? 'bg-yellow-100 text-yellow-800' :
            'bg-red-100 text-red-800',
          )}
          title={`${p.phoneme}: ${p.accuracyScore}%`}
        >
          {p.phoneme}
        </span>
      ))}
    </div>
  );
}
```

**Testing:**
- Phonemes with accuracy >= 80 are displayed in green; 50-79 in yellow; <50 in red
- Low-scoring phonemes show corrective guidance tooltip/popover with articulatory description
- Guidance for a native English speaker learning Spanish `/ɾ/` mentions "tap the tongue tip against the alveolar ridge"
- Guidance is specific to the learner's native language (English speaker gets different advice than Chinese speaker)
- Phoneme display renders correctly for non-Latin phonemes (Arabic, Japanese)
- Hovering over a phoneme shows the score percentage

### Task 4.4: Speaking Exercises in Lesson Flow

**What:** Add `speaking_repeat` and `speaking_freeform` exercise types to the lesson engine. Speaking exercises present reference text and audio, record the learner's attempt, send to Azure for assessment, and score based on pronunciation accuracy.

**Design:**

```tsx
// apps/web/src/components/exercises/SpeakingRepeatExercise.tsx
export function SpeakingRepeatExercise({ exercise, onComplete }: ExerciseProps) {
  const [recording, setRecording] = useState(false);
  const [result, setResult] = useState<PronunciationResult | null>(null);
  const recorder = useRef(new AudioRecorder());

  const handleRecord = async () => {
    if (recording) {
      const blob = await recorder.current.stop();
      setRecording(false);
      const result = await submitPronunciation(blob, exercise.exercise_data.reference_text);
      setResult(result);
      onComplete({
        isCorrect: result.overallScore >= exercise.exercise_data.acceptable_accuracy,
        score: result.overallScore,
      });
    } else {
      await recorder.current.start();
      setRecording(true);
    }
  };

  return (
    <div className="text-center">
      <p className="text-xl mb-4">{exercise.exercise_data.reference_text}</p>
      <AudioPlayer src={exercise.exercise_data.reference_audio_url} label="Listen to the native speaker" />
      <RecordButton recording={recording} onClick={handleRecord} />
      {result && (
        <>
          <ScoreDisplay overall={result.overallScore} accuracy={result.accuracyScore} fluency={result.fluencyScore} />
          <PhonemeDisplay phonemes={result.phonemes} />
        </>
      )}
    </div>
  );
}
```

**Testing:**
- Speaking exercise shows reference text and "Listen" button with native audio
- "Record" button starts recording; visual indicator shows recording state
- After stopping, audio is uploaded and assessed within 5 seconds
- Score >= acceptable_accuracy (default 70) marks exercise as correct
- Score < acceptable_accuracy allows retry with "Try again" button
- Phoneme-level feedback displays after assessment
- Speaking exercises contribute to pronunciation weakness tracking in `learner_language_summary.proficiency`

### Phase 4 Definition of Done

- [ ] Audio recording works in Chrome, Firefox, Safari, and Edge
- [ ] Azure Pronunciation Assessment returns per-phoneme accuracy scores
- [ ] Corrective phoneme guidance adapts to learner's native language
- [ ] Speaking exercises integrated into lesson flow with scoring
- [ ] Pronunciation results stored in activity_log with phoneme detail
- [ ] Error handling for microphone permissions and Azure API failures

---

## Phase 5: AI Conversation Partner

**Goal:** Build the always-available AI conversation partner that simulates authentic dialogue in the target language, adapts to the learner's CEFR level, corrects errors in context, and provides grammar feedback.

### Task 5.1: Conversation Session Management

**What:** Create the `conversation` table and API endpoints for starting, continuing, and ending conversation sessions. Implement conversation scenario templates (e.g., "Ordering at a restaurant", "Checking into a hotel").

**Design:**

```typescript
// apps/api/src/modules/conversation/conversation.service.ts
export class ConversationService {
  async startConversation(
    userId: string,
    targetLanguageId: string,
    scenarioId?: string,
  ): Promise<ConversationSessionDto> {
    const user = await this.userService.getWithLanguageStats(userId, targetLanguageId);
    const cefrLevel = user.languageStats?.proficiency?.overall?.cefr ?? 'A1';
    const scenario = scenarioId
      ? await this.getScenario(scenarioId)
      : await this.selectScenarioForLevel(cefrLevel, targetLanguageId);

    const session = await db.insert(conversation).values({
      userId,
      targetLanguageId,
      cefrLevel,
      scenarioTitle: scenario.title,
    }).returning();

    const systemPrompt = this.buildSystemPrompt(user, scenario, cefrLevel);
    const aiOpening = await this.llmService.generateConversationTurn(systemPrompt, []);

    await this.recordTurn(session[0].id, 0, 'ai', aiOpening.content);

    return {
      sessionId: session[0].id,
      scenario: scenario.title,
      cefrLevel,
      aiMessage: aiOpening.content,
    };
  }

  private buildSystemPrompt(
    user: UserWithStats,
    scenario: ConversationScenario,
    cefrLevel: string,
  ): string {
    return `You are a language practice partner helping a learner practice ${user.targetLanguage.nameEnglish}.

SCENARIO: ${scenario.title}
DESCRIPTION: ${scenario.description}
LEARNER'S CEFR LEVEL: ${cefrLevel}
LEARNER'S NATIVE LANGUAGE: ${user.nativeLanguage.nameEnglish}

RULES:
- Speak ONLY in ${user.targetLanguage.nameEnglish} unless the learner is completely stuck.
- Use vocabulary and grammar appropriate for ${cefrLevel} level.
- Keep your responses to 1-3 sentences.
- When the learner makes a grammatical error, gently correct it in your next response.
- After each exchange, provide a brief correction note in JSON format:
  {"corrections": [{"error": "...", "correction": "...", "rule": "...", "explanation": "..."}]}
- Be encouraging but accurate. Do not invent words or grammar rules.`;
  }
}
```

**Testing:**
- Starting a conversation creates a `conversation` record with correct CEFR level
- AI opening message is in the target language and appropriate for the learner's CEFR level
- AI at A1 uses simple present tense and basic vocabulary; AI at B2 uses subjunctive and compound sentences
- Scenario selection avoids repeating the same scenario within 7 days for the same learner
- System prompt includes learner's native language for L1-aware error correction

### Task 5.2: Real-Time Conversation Streaming

**What:** Implement streaming conversation responses via WebSocket or Server-Sent Events (SSE). The learner types or speaks a message; the AI responds in real-time with streaming text. Grammar corrections are extracted and displayed after each turn.

**Design:**

```typescript
// apps/api/src/modules/conversation/conversation.gateway.ts
@WebSocketGateway({ namespace: '/conversation' })
export class ConversationGateway {
  @SubscribeMessage('send_message')
  async handleMessage(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { sessionId: string; content: string },
  ) {
    const session = await this.conversationService.getSession(data.sessionId);
    const history = await this.conversationService.getTurnHistory(data.sessionId);

    // Record learner turn
    await this.conversationService.recordTurn(data.sessionId, history.length, 'learner', data.content);

    // Stream AI response
    const stream = await this.llmService.streamConversationTurn(
      session.systemPrompt,
      [...history, { role: 'user', content: data.content }],
    );

    let fullResponse = '';
    for await (const chunk of stream) {
      fullResponse += chunk;
      client.emit('ai_chunk', { text: chunk });
    }

    // Extract corrections from response
    const { message, corrections } = this.extractCorrections(fullResponse);

    // Record AI turn
    await this.conversationService.recordTurn(data.sessionId, history.length + 1, 'ai', message);

    // Log corrections
    await this.logConversationActivity(session, data.content, corrections);

    client.emit('ai_complete', { corrections });
  }
}
```

**Testing:**
- Learner sends a message; AI response streams token-by-token within 500ms of first token
- Grammar corrections are displayed in a collapsible panel after each AI turn
- Conversation history persists across page refreshes within the same session
- Ending a conversation records `total_turns`, `duration_seconds`, and a summary in the `conversation` table
- WebSocket reconnects automatically on network interruption; conversation resumes
- Conversation turns are recorded in `activity_log` with `activity_type = 'conversation_turn'`

### Task 5.3: Voice Input for Conversations

**What:** Allow learners to speak their conversation turns instead of typing. Record audio, transcribe via Whisper/Azure, send transcription to the LLM, and optionally assess pronunciation of the spoken turn.

**Design:**

```typescript
// apps/web/src/components/conversation/VoiceInput.tsx
export function VoiceInput({ onTranscription, onPronunciationResult }: VoiceInputProps) {
  const recorder = useRef(new AudioRecorder());
  const [recording, setRecording] = useState(false);

  const handleToggle = async () => {
    if (recording) {
      const blob = await recorder.current.stop();
      setRecording(false);

      // Parallel: transcribe + pronunciation assessment
      const [transcription, pronunciation] = await Promise.all([
        transcribeAudio(blob),
        assessPronunciation(blob), // Optional, can be toggled off
      ]);

      onTranscription(transcription.text);
      if (pronunciation) onPronunciationResult(pronunciation);
    } else {
      await recorder.current.start();
      setRecording(true);
    }
  };

  return (
    <button onClick={handleToggle} className={cn('mic-button', recording && 'recording')}>
      {recording ? <MicOffIcon /> : <MicIcon />}
    </button>
  );
}
```

**Testing:**
- Microphone button starts/stops recording with visual feedback
- Transcription appears in the chat input field; learner can edit before sending
- Pronunciation assessment runs in parallel with transcription; results shown inline
- Voice input works in noisy environments (tested with background noise)
- Conversation flows naturally between text and voice input within the same session
- Voice turns include pronunciation scores in the activity log

### Phase 5 Definition of Done

- [ ] AI conversation partner responds in the target language at the learner's CEFR level
- [ ] Responses stream in real-time via WebSocket/SSE
- [ ] Grammar corrections extracted and displayed after each turn
- [ ] Voice input with transcription and optional pronunciation assessment
- [ ] Conversation history persisted and resumable
- [ ] Conversation summary generated on session end

---

## Phase 6: Continuous CEFR Proficiency Estimation

**Goal:** Build the continuous proficiency estimation engine that infers CEFR level from all learner production activities (exercises, reviews, conversations, pronunciation) without requiring standalone formal tests.

### Task 6.1: Proficiency Estimation Model

**What:** Build a proficiency estimation service that aggregates signals from exercise accuracy, vocabulary breadth, conversation complexity, and pronunciation scores to produce per-skill CEFR level estimates with confidence scores.

**Design:**

```typescript
// apps/api/src/modules/proficiency/proficiency-estimator.service.ts
export class ProficiencyEstimatorService {
  async estimateProficiency(
    userId: string,
    targetLanguageId: string,
  ): Promise<ProficiencyEstimate> {
    const [exerciseStats, vocabStats, conversationStats, pronunciationStats] = await Promise.all([
      this.getExerciseStats(userId, targetLanguageId, 30), // last 30 days
      this.getVocabStats(userId, targetLanguageId),
      this.getConversationStats(userId, targetLanguageId, 30),
      this.getPronunciationStats(userId, targetLanguageId, 30),
    ]);

    // Reading: exercise accuracy on reading/translation exercises + vocab breadth
    const reading = this.estimateSkill('reading', {
      exerciseAccuracy: exerciseStats.readingAccuracy,
      vocabKnown: vocabStats.wordsKnown,
      cefrVocabThresholds: CEFR_VOCAB_THRESHOLDS, // A1: 500, A2: 1000, B1: 2000, B2: 4000
    });

    // Speaking: pronunciation scores + conversation turn complexity
    const speaking = this.estimateSkill('speaking', {
      avgPronunciationScore: pronunciationStats.avgScore,
      avgTurnLength: conversationStats.avgTurnLength,
      grammarErrorRate: conversationStats.grammarErrorRate,
      vocabularyDiversity: conversationStats.uniqueWordsUsed,
    });

    // Writing: exercise accuracy on translation-to-target + fill-blank
    const writing = this.estimateSkill('writing', {
      exerciseAccuracy: exerciseStats.writingAccuracy,
      vocabKnown: vocabStats.wordsKnown,
    });

    // Listening: exercise accuracy on listening comprehension + dictation
    const listening = this.estimateSkill('listening', {
      exerciseAccuracy: exerciseStats.listeningAccuracy,
    });

    const overall = this.computeOverallLevel([reading, writing, listening, speaking]);

    // Store estimates
    for (const estimate of [reading, writing, listening, speaking, overall]) {
      await db.insert(activityLog).values({
        userId,
        targetLanguageId,
        activityType: 'proficiency_estimated',
        detail: {
          skill: estimate.skill,
          cefr_level: estimate.cefrLevel,
          confidence: estimate.confidence,
          model_version: 'proficiency-v1',
        },
      });
    }

    // Update summary
    await this.updateLanguageSummary(userId, targetLanguageId, { reading, writing, listening, speaking, overall });

    return { reading, writing, listening, speaking, overall };
  }

  private estimateSkill(skill: string, signals: SkillSignals): SkillEstimate {
    // Weighted signal aggregation with CEFR threshold mapping
    // Each CEFR level has defined thresholds for each signal
    const levelScores = CEFR_LEVELS.map(level => ({
      level,
      score: this.computeLevelFit(level, skill, signals),
    }));

    const best = levelScores.reduce((a, b) => a.score > b.score ? a : b);
    return {
      skill,
      cefrLevel: best.level,
      confidence: best.score,
    };
  }
}
```

**Testing:**
- Learner with 100% accuracy on A1 exercises and 500 known words is estimated at A1 with high confidence
- Learner with 80% accuracy on B1 exercises, 2000 known words, and 75% pronunciation score is estimated at B1
- Speaking estimate requires pronunciation data; without it, confidence is low
- Proficiency estimate runs after every 10th exercise completion (not on every attempt)
- `learner_language_summary.proficiency` JSONB is updated with latest per-skill estimates
- Historical estimates are queryable from `activity_log` for progress charts

### Task 6.2: Proficiency Dashboard and Progress Charts

**What:** Build the proficiency dashboard showing per-skill CEFR levels, confidence indicators, progress over time charts, and vocabulary growth charts.

**Design:**

```tsx
// apps/web/src/components/dashboard/ProficiencyRadar.tsx
// Radar chart showing CEFR level per skill (reading, writing, listening, speaking)
export function ProficiencyRadar({ proficiency }: { proficiency: ProficiencyEstimate }) {
  const data = [
    { skill: 'Reading', level: cefrToNumber(proficiency.reading.cefrLevel), confidence: proficiency.reading.confidence },
    { skill: 'Writing', level: cefrToNumber(proficiency.writing.cefrLevel), confidence: proficiency.writing.confidence },
    { skill: 'Listening', level: cefrToNumber(proficiency.listening.cefrLevel), confidence: proficiency.listening.confidence },
    { skill: 'Speaking', level: cefrToNumber(proficiency.speaking.cefrLevel), confidence: proficiency.speaking.confidence },
  ];

  return <RadarChart data={data} maxValue={6} />; // 6 = C2
}

function cefrToNumber(level: string): number {
  const map: Record<string, number> = { 'A1': 1, 'A2': 2, 'B1': 3, 'B2': 4, 'C1': 5, 'C2': 6 };
  return map[level] ?? 0;
}
```

**Testing:**
- Radar chart displays four axes (reading, writing, listening, speaking) with CEFR levels 1-6
- Confidence is indicated by opacity or saturation of the radar fill
- Progress timeline chart shows CEFR level changes over the past 90 days
- Vocabulary growth chart shows words_known over time
- Dashboard loads within 2 seconds for a learner with 6 months of activity data
- Dashboard is responsive on mobile (charts resize correctly)

### Phase 6 Definition of Done

- [ ] Per-skill CEFR proficiency estimated from production data
- [ ] Estimates update automatically after sufficient new activity
- [ ] Proficiency dashboard with radar chart and progress timeline
- [ ] Historical estimates stored for temporal queries
- [ ] Vocabulary growth tracking and visualization

---

## Phase 7: Mobile Apps (iOS and Android)

**Goal:** Build cross-platform mobile apps using React Native (Expo) with offline lesson access, SRS review, audio recording, and push notifications.

### Task 7.1: React Native App Shell

**What:** Initialize the Expo project with navigation (React Navigation), authentication flow, and API client. Share TypeScript types with the web app via `packages/types`.

**Design:**

```typescript
// apps/mobile/src/navigation/AppNavigator.tsx
export function AppNavigator() {
  const { isAuthenticated } = useAuth();

  return (
    <NavigationContainer>
      {isAuthenticated ? (
        <Tab.Navigator>
          <Tab.Screen name="Learn" component={LearnStack} />
          <Tab.Screen name="Review" component={ReviewStack} />
          <Tab.Screen name="Conversation" component={ConversationStack} />
          <Tab.Screen name="Profile" component={ProfileStack} />
        </Tab.Navigator>
      ) : (
        <AuthStack />
      )}
    </NavigationContainer>
  );
}
```

**Testing:**
- App launches on iOS simulator and Android emulator without crashes
- Login flow works with email/password and Google sign-in
- Bottom tab navigation between Learn, Review, Conversation, and Profile tabs
- API requests include JWT token in Authorization header
- Token refresh works transparently when access token expires

### Task 7.2: Offline Lesson Access

**What:** Implement offline lesson download using expo-sqlite. Lessons, exercises, and vocabulary audio are cached locally. SRS reviews work offline and sync when back online.

**Design:**

```typescript
// apps/mobile/src/lib/offline-sync.ts
export class OfflineSyncService {
  async downloadCourseForOffline(courseId: string): Promise<void> {
    const course = await api.getCourseWithContent(courseId);

    // Store course structure in SQLite
    await localDb.exec(`INSERT OR REPLACE INTO courses VALUES (?, ?, ?)`,
      [course.id, JSON.stringify(course), Date.now()]);

    // Download audio files
    for (const vocab of course.vocabulary) {
      if (vocab.audioUrl) {
        const localPath = await FileSystem.downloadAsync(vocab.audioUrl, localAudioPath(vocab.id));
        await localDb.exec(`INSERT OR REPLACE INTO audio_cache VALUES (?, ?)`,
          [vocab.id, localPath.uri]);
      }
    }
  }

  async syncPendingReviews(): Promise<void> {
    const pending = await localDb.exec(`SELECT * FROM pending_reviews`);
    for (const review of pending) {
      try {
        await api.rateCard(review.cardId, review.rating);
        await localDb.exec(`DELETE FROM pending_reviews WHERE id = ?`, [review.id]);
      } catch (e) {
        if (!isNetworkError(e)) throw e;
        break; // Stop syncing, will retry later
      }
    }
  }
}
```

**Testing:**
- Download a course for offline access; progress indicator shows download status
- Put device in airplane mode; open the app; lessons and review cards are accessible
- Complete an SRS review offline; rating is queued in `pending_reviews`
- Re-enable network; pending reviews sync automatically within 30 seconds
- Audio playback works offline for downloaded vocabulary items
- Storage usage displayed in settings; user can delete offline courses

### Task 7.3: Mobile Audio Recording and Pronunciation

**What:** Implement audio recording on mobile using expo-av. Connect to the same pronunciation assessment flow as the web app.

**Testing:**
- Audio recording works on both iOS and Android with in-ear headphones and speaker
- Recording quality is sufficient for Azure Pronunciation Assessment (16kHz, mono)
- Pronunciation feedback displays phoneme-level results in the mobile UI
- Recording permissions are requested gracefully; denial shows an explanatory message
- Background audio (music, notifications) does not interfere with recording

### Task 7.4: Push Notifications for Daily Reminders

**What:** Implement push notifications via Expo Notifications for daily study reminders and streak maintenance alerts.

**Testing:**
- User can set daily reminder time in preferences
- Push notification fires at the configured time with message "Time to practice [language]!"
- Tapping the notification opens the app to the review screen
- Streak-at-risk notification fires at 8 PM if the user has not studied that day
- Notifications can be disabled in app settings; disabling stops all notifications

### Phase 7 Definition of Done

- [ ] Mobile app runs on iOS and Android via Expo
- [ ] Offline lesson access with SQLite caching
- [ ] Offline SRS reviews with sync-on-reconnect
- [ ] Audio recording and pronunciation assessment on mobile
- [ ] Push notifications for daily reminders and streak alerts
- [ ] App Store and Google Play submission-ready (icons, splash screen, metadata)

---

## Phase 8: Gamification, Streaks, and Engagement

**Goal:** Implement the engagement layer: daily streaks, XP system, achievements/badges, and leaderboards.

### Task 8.1: Streak System

**What:** Implement daily streak tracking. A streak increments when the user completes at least one study activity in a calendar day (user's timezone). Streak freezes (max 2 per month) protect against missed days.

**Design:**

```typescript
// apps/api/src/modules/gamification/streak.service.ts
export class StreakService {
  async recordActivity(userId: string): Promise<StreakUpdate> {
    const user = await db.query.appUser.findFirst({ where: eq(appUser.id, userId) });
    const userTimezone = user.preferences?.timezone ?? 'UTC';
    const today = DateTime.now().setZone(userTimezone).toISODate();

    if (user.streakLastDate === today) {
      return { streakDays: user.streakDays, isNewDay: false };
    }

    const yesterday = DateTime.now().setZone(userTimezone).minus({ days: 1 }).toISODate();
    let newStreak: number;

    if (user.streakLastDate === yesterday) {
      newStreak = user.streakDays + 1;
    } else if (user.streakLastDate === null) {
      newStreak = 1;
    } else {
      // Check for streak freeze
      const frozeUsed = await this.checkStreakFreeze(userId, user.streakLastDate, today);
      newStreak = frozeUsed ? user.streakDays + 1 : 1;
    }

    await db.update(appUser)
      .set({ streakDays: newStreak, streakLastDate: today })
      .where(eq(appUser.id, userId));

    return { streakDays: newStreak, isNewDay: true };
  }
}
```

**Testing:**
- First activity of the day increments streak by 1
- Second activity of the same day does not change streak
- Missing a day without a freeze resets streak to 1
- Using a streak freeze preserves the streak across a missed day
- Streak is calculated in the user's timezone, not UTC
- Maximum 2 streak freezes per calendar month

### Task 8.2: XP System and Achievements

**What:** Implement XP earned from exercises, reviews, and conversations. Define achievements with criteria (streak milestones, words learned milestones, first conversation, pronunciation score milestones). Award badges when criteria are met.

**Design:**

```typescript
// packages/db/seed/achievements.ts
const achievements = [
  { name: 'First Steps', description: 'Complete your first lesson', criteria: { type: 'lessons_completed', threshold: 1 } },
  { name: 'Vocabulary Builder', description: 'Learn 100 words', criteria: { type: 'words_known', threshold: 100 } },
  { name: 'Vocabulary Master', description: 'Learn 1000 words', criteria: { type: 'words_known', threshold: 1000 } },
  { name: 'Streak Starter', description: 'Maintain a 7-day streak', criteria: { type: 'streak', threshold: 7 } },
  { name: 'Streak Champion', description: 'Maintain a 30-day streak', criteria: { type: 'streak', threshold: 30 } },
  { name: 'First Conversation', description: 'Complete your first AI conversation', criteria: { type: 'conversation_count', threshold: 1 } },
  { name: 'Perfect Pronunciation', description: 'Score 95% or above on pronunciation', criteria: { type: 'pronunciation_score', threshold: 95 } },
  { name: 'Polyglot', description: 'Study 3 languages', criteria: { type: 'languages_studied', threshold: 3 } },
];
```

**Testing:**
- Completing first lesson awards "First Steps" badge; notification displays
- Reaching 100 known words awards "Vocabulary Builder" badge
- Achievement cannot be earned twice (unique constraint on `user_id, achievement_id`)
- Achievement earned event recorded in activity_log
- Achievements page shows earned badges with dates and locked badges with progress indicators
- XP calculation: 10 XP per correct exercise, 5 XP per review, 25 XP per conversation turn, 50 XP bonus for lesson completion

### Task 8.3: Weekly Leaderboard

**What:** Implement a weekly leaderboard showing XP earned in the current week among learners studying the same language. Leaderboard resets every Monday.

**Testing:**
- Leaderboard shows top 30 learners for the current week sorted by XP
- Current user's rank is highlighted even if not in top 30
- Leaderboard resets at midnight Monday UTC
- Users can opt out of leaderboards in privacy settings
- Leaderboard query performs within 200ms (indexed on XP + language + week)

### Phase 8 Definition of Done

- [ ] Daily streaks track correctly across timezones
- [ ] Streak freezes work (max 2/month)
- [ ] XP awarded for all learning activities
- [ ] 10+ achievements defined and awarded automatically
- [ ] Weekly leaderboard displays and resets correctly
- [ ] Engagement metrics do not degrade API response times

---

## Phase 9: Personalised Content Generation

**Goal:** Generate personalised reading passages, dialogue scenarios, and listening exercises aligned to the learner's interests and current CEFR level using the LLM.

### Task 9.1: Learner Interest Profiles

**What:** Allow learners to select interest tags (cooking, sports, travel, technology, music, film, etc.) during onboarding and in settings. Store in `app_user.preferences.interests`.

**Testing:**
- Onboarding flow presents 15+ interest tags; learner selects 3-5
- Interests are stored in `preferences.interests` JSONB array
- Interests can be updated in settings at any time
- Content generation API receives interest tags as input

### Task 9.2: AI Content Generation Pipeline

**What:** Build the content generation pipeline that produces reading passages, dialogue scenarios, and comprehension questions at the learner's CEFR level using their interest tags. Content goes through an editorial approval workflow before being served to other learners.

**Design:**

```typescript
// apps/api/src/modules/content-gen/content-generator.service.ts
export class ContentGeneratorService {
  async generateReadingPassage(
    targetLanguageId: string,
    cefrLevel: string,
    interestTags: string[],
  ): Promise<GeneratedContentDto> {
    const language = await this.getLanguage(targetLanguageId);

    const prompt = `Generate a reading passage in ${language.nameEnglish} at CEFR ${cefrLevel} level.

TOPIC: ${interestTags.join(', ')}
LENGTH: 150-250 words
REQUIREMENTS:
- Use only vocabulary and grammar appropriate for ${cefrLevel}
- Include 5-8 vocabulary words that a learner at this level should learn
- Write 3 comprehension questions with answers

OUTPUT FORMAT (JSON):
{
  "title": "...",
  "text": "...",
  "translation": "...",
  "vocabulary_highlights": ["word1", "word2", ...],
  "comprehension_questions": [
    {"question": "...", "answer": "...", "type": "factual|inferential"}
  ]
}`;

    const result = await this.llmService.generateStructured(prompt);

    return db.insert(generatedContent).values({
      targetLanguageId,
      contentType: 'reading_passage',
      cefrLevel,
      title: result.title,
      content: result,
      isApproved: false, // Requires editorial review
    }).returning();
  }
}
```

**Testing:**
- Generated passage for "cooking" + "A1" + Spanish uses basic vocabulary (cocinar, comer, delicioso) and present tense only
- Generated passage for "technology" + "B2" + French uses subjunctive, passive voice, and domain vocabulary
- Content passes validation: has title, text, translation, vocabulary highlights, and 3 questions
- Generated content is stored with `is_approved = false`; not served until reviewed
- Admin can approve or reject generated content from the admin dashboard
- Approved content appears in the learner's personalized content feed

### Task 9.3: Personalised Content Feed

**What:** Build the personalised content feed that surfaces approved content matching the learner's interests, CEFR level, and target language. Content the learner has already read is excluded.

**Testing:**
- Feed shows content matching the learner's interests and CEFR level
- Content already read by the learner is not shown again
- Feed refreshes with new content as more is generated and approved
- Learner can read a passage, listen to TTS audio, and answer comprehension questions
- Correct comprehension answers award XP and feed vocabulary into SRS

### Phase 9 Definition of Done

- [ ] Learners can set interest tags during onboarding and in settings
- [ ] AI generates reading passages, dialogues, and scenarios at appropriate CEFR levels
- [ ] Editorial approval workflow prevents unreviewed content from reaching learners
- [ ] Personalised content feed surfaces relevant, unseen content
- [ ] Generated content vocabulary feeds into the SRS system

---

## Phase 10: Enterprise Tier — SSO, Admin, SCORM, LTI

**Goal:** Build the enterprise tier: organisation management, SSO (SAML 2.0, OIDC), admin dashboard with team progress reporting, SCORM 2004 content export, and IMS LTI 1.3 integration.

### Task 10.1: Organisation Management and SSO

**What:** Implement organisation CRUD, seat management, and SSO configuration (SAML 2.0 and OIDC). Enterprise users authenticate via their organisation's IdP.

**Design:**

```typescript
// apps/api/src/modules/enterprise/sso.service.ts
export class SSOService {
  async configureSAML(orgId: string, config: SAMLConfigDto): Promise<void> {
    const org = await db.query.organisation.findFirst({ where: eq(organisation.id, orgId) });

    await db.update(organisation).set({
      config: {
        ...org.config,
        sso: {
          protocol: 'saml2',
          entity_id: config.entityId,
          metadata_xml: config.metadataXml,
          sso_url: config.ssoUrl,
          certificate: config.certificate,
        },
      },
    }).where(eq(organisation.id, orgId));
  }

  async handleSAMLCallback(samlResponse: string): Promise<AuthResult> {
    const parsed = await this.samlParser.parseResponse(samlResponse);
    const org = await this.findOrgByEntityId(parsed.issuer);
    const user = await this.findOrCreateSSOUser(org.id, parsed.email, parsed.displayName);
    return this.authService.generateTokens(user);
  }
}
```

**Testing:**
- Organisation admin can configure SAML 2.0 with IdP metadata XML
- SAML login flow: user clicks "SSO Login" -> redirected to IdP -> authenticates -> redirected back with SAML assertion -> user session created
- OIDC login flow works with Azure AD, Okta, and Google Workspace
- New SSO users are auto-provisioned with the learner role in the organisation
- Seat limit enforcement: adding users beyond `max_seats` returns 403
- Organisation admin cannot access other organisations' data

### Task 10.2: Admin Dashboard and Progress Reporting

**What:** Build the organisation admin dashboard showing team enrolment, individual and aggregate progress, CEFR level distribution, and exportable CSV reports.

**Testing:**
- Admin sees list of all organisation members with their current CEFR levels per language
- Aggregate view shows CEFR level distribution chart (how many learners at each level)
- Admin can filter by team, department, or date range
- CSV export includes learner name, email, language, current CEFR level, study hours, and streak
- Admin can enrol/remove learners and assign courses to teams
- Non-admin users cannot access the admin dashboard (403)

### Task 10.3: SCORM 2004 Export

**What:** Export courses as SCORM 2004 (4th Edition) packages that can be imported into enterprise LMS platforms (Cornerstone, SAP SuccessFactors, Moodle).

**Design:**

```typescript
// apps/api/src/modules/enterprise/scorm-exporter.service.ts
export class SCORMExporterService {
  async exportCourse(courseId: string): Promise<Buffer> {
    const course = await this.getCourseWithFullContent(courseId);
    const zip = new JSZip();

    // imsmanifest.xml
    zip.file('imsmanifest.xml', this.generateManifest(course));

    // SCOs (Shareable Content Objects) — one per lesson
    for (const unit of course.units) {
      for (const lesson of unit.lessons) {
        const scoPath = `content/${unit.id}/${lesson.id}/`;
        zip.file(`${scoPath}index.html`, this.generateLessonHTML(lesson));
        zip.file(`${scoPath}scorm_api.js`, SCORM_API_WRAPPER);
      }
    }

    return zip.generateAsync({ type: 'nodebuffer' });
  }
}
```

**Testing:**
- Exported SCORM ZIP contains valid `imsmanifest.xml` conforming to SCORM 2004 4th Edition schema
- SCORM package imports successfully into Moodle 4.x
- SCORM package imports successfully into SCORM Cloud (cloud.scorm.com) test environment
- Lesson completion in the LMS is tracked via SCORM API `cmi.completion_status`
- Score is reported via `cmi.score.raw`

### Task 10.4: IMS LTI 1.3 Integration

**What:** Implement LTI 1.3 Tool Provider. The language learning platform can be launched from within Canvas, Moodle, or Blackboard as an embedded tool. Support Deep Linking for content selection and Assignment and Grade Services for grade passback.

**Design:**

```typescript
// apps/api/src/modules/enterprise/lti.controller.ts
@Controller('api/v1/lti')
export class LTIController {
  @Post('login')
  async ltiLogin(@Body() body: LTILoginRequest, @Res() res: Response) {
    // OIDC login initiation (LTI 1.3 uses OIDC for authentication)
    const state = randomUUID();
    const nonce = randomUUID();
    await this.ltiService.storeState(state, nonce);
    const authUrl = this.ltiService.buildAuthRedirect(body, state, nonce);
    res.redirect(authUrl);
  }

  @Post('callback')
  async ltiCallback(@Body() body: LTICallbackRequest) {
    const claims = await this.ltiService.validateAndDecodeJWT(body.id_token);
    const user = await this.ltiService.findOrCreateLTIUser(claims);
    const session = await this.authService.createSession(user);

    // If Deep Linking request, show content picker
    if (claims['https://purl.imsglobal.org/spec/lti-dl/claim/deep_linking_settings']) {
      return this.handleDeepLinking(claims, session);
    }

    // Standard launch: redirect to the course/lesson
    return this.handleResourceLaunch(claims, session);
  }

  @Post('grade')
  async postGrade(@Body() body: PostGradeDto) {
    // Assignment and Grade Services (AGS) — post learner's score back to LMS
    await this.ltiService.postScore(body.lineItemUrl, body.userId, body.score);
  }
}
```

**Testing:**
- LTI launch from Canvas creates a user session and redirects to the correct lesson
- LTI launch from Moodle creates a user session and redirects correctly
- Deep Linking flow allows instructor to select a specific lesson to embed
- Grade passback sends the lesson score to the LMS gradebook
- Invalid LTI JWT is rejected with 401
- LTI user is linked to the correct organisation based on the platform issuer

### Phase 10 Definition of Done

- [ ] SSO works with SAML 2.0 (tested with Okta) and OIDC (tested with Azure AD)
- [ ] Admin dashboard shows team progress, CEFR distribution, and CSV export
- [ ] SCORM 2004 packages import into Moodle and SCORM Cloud
- [ ] LTI 1.3 launch works from Canvas and Moodle with grade passback
- [ ] Seat limit enforcement prevents over-provisioning

---

## Phase 11: xAPI Learning Record Store and Analytics

**Goal:** Implement xAPI statement generation from all learning activities, an internal Learning Record Store (LRS), and an analytics dashboard for institutional buyers.

### Task 11.1: xAPI Statement Generation

**What:** Map each activity type in `activity_log` to an xAPI Actor-Verb-Object statement. Store statements in a queryable format. Support export to external LRS systems.

**Design:**

```typescript
// apps/api/src/modules/xapi/xapi-mapper.service.ts
export class XAPIMapperService {
  mapActivityToStatement(activity: ActivityLog, user: AppUser): XAPIStatement {
    const actor: XAPIActor = {
      mbox: `mailto:${user.email}`,
      name: user.displayName,
      objectType: 'Agent',
    };

    switch (activity.activityType) {
      case 'exercise_attempt':
        return {
          actor,
          verb: { id: 'http://adlnet.gov/expapi/verbs/answered', display: { 'en-US': 'answered' } },
          object: {
            id: `${BASE_IRI}/exercises/${activity.exerciseId}`,
            objectType: 'Activity',
            definition: { type: 'http://adlnet.gov/expapi/activities/assessment' },
          },
          result: {
            score: { raw: activity.isCorrect ? 100 : 0, min: 0, max: 100 },
            success: activity.isCorrect,
          },
          timestamp: activity.createdAt.toISOString(),
        };

      case 'lesson_completed':
        return {
          actor,
          verb: { id: 'http://adlnet.gov/expapi/verbs/completed', display: { 'en-US': 'completed' } },
          object: {
            id: `${BASE_IRI}/lessons/${activity.lessonId}`,
            objectType: 'Activity',
          },
          result: {
            completion: true,
            duration: `PT${activity.detail.duration_seconds}S`,
            score: {
              raw: Math.round((activity.detail.exercises_correct / activity.detail.exercises_completed) * 100),
              min: 0, max: 100,
            },
          },
          timestamp: activity.createdAt.toISOString(),
        };

      // ... other activity types
    }
  }
}
```

**Testing:**
- Exercise attempt generates a valid xAPI statement with verb "answered" and success/score result
- Lesson completion generates a statement with verb "completed", duration, and score
- Conversation end generates a statement with verb "experienced" and duration
- All statements conform to xAPI 2.0 JSON schema (validated against official schema)
- Statements export as valid JSON array for import into external LRS (Learning Locker, Watershed)
- xAPI statement query API supports filtering by actor, verb, activity, and date range

### Task 11.2: Analytics Dashboard for Institutions

**What:** Build an analytics dashboard for institutional buyers (enterprise and education) showing aggregate learning analytics: active learners, study time distribution, CEFR progression rates, most/least effective content, and retention rates.

**Testing:**
- Dashboard shows active learners (daily/weekly/monthly active) with trend line
- Study time distribution chart shows average daily study time by cohort
- CEFR progression chart shows average time to reach each level by language
- Content effectiveness: lessons with highest/lowest completion rates and average scores
- Retention curve: percentage of learners still active after 7, 30, 60, 90 days
- All charts filter by organisation, language, date range, and cohort

### Phase 11 Definition of Done

- [ ] All activity types mapped to valid xAPI statements
- [ ] xAPI statements exportable to external LRS in standard JSON format
- [ ] Internal query API for xAPI statements with filtering
- [ ] Analytics dashboard with 6+ institutional KPI charts
- [ ] Dashboard loads within 3 seconds for organisations with 1000+ learners

---

## Phase 12: Community Features and Tutor Marketplace

**Goal:** Add community features (native speaker feedback on writing/speaking) and optional live tutor marketplace integration.

### Task 12.1: Community Writing Feedback

**What:** Allow learners to submit writing samples for correction by native speakers in the community. Native speakers can correct, annotate, and provide feedback on submitted texts.

**Design:**

```typescript
// apps/api/src/modules/community/writing-submission.service.ts
export class WritingSubmissionService {
  async submitForReview(
    userId: string,
    targetLanguageId: string,
    text: string,
    cefrLevel: string,
  ): Promise<WritingSubmission> {
    return db.insert(writingSubmission).values({
      userId,
      targetLanguageId,
      originalText: text,
      cefrLevel,
      status: 'pending',
    }).returning();
  }

  async submitCorrection(
    reviewerId: string,
    submissionId: string,
    corrections: InlineCorrection[],
    feedback: string,
  ): Promise<void> {
    // Verify reviewer is a native speaker of the target language
    const reviewer = await this.getUser(reviewerId);
    const submission = await this.getSubmission(submissionId);
    if (reviewer.nativeLanguageId !== submission.targetLanguageId) {
      throw new ForbiddenException('Only native speakers can review');
    }

    await db.insert(writingCorrection).values({
      submissionId,
      reviewerId,
      corrections: corrections,
      feedback,
    });

    await db.update(writingSubmission)
      .set({ status: 'corrected' })
      .where(eq(writingSubmission.id, submissionId));
  }
}
```

**Testing:**
- Learner submits a writing sample; it appears in the community review queue
- Native speakers of the target language see submissions to review
- Non-native speakers cannot submit corrections (enforced by language match)
- Inline corrections highlight specific errors with explanations
- Learner receives a notification when their submission is corrected
- Corrected writing is viewable with diff-style highlighting

### Task 12.2: Tutor Marketplace Integration

**What:** Build an in-platform booking system for live human tutor sessions. Tutors set availability and hourly rates. Learners browse tutors by language, price, and rating, and book sessions.

**Testing:**
- Tutors can register with a profile (languages, rates, availability, intro video)
- Learners can search tutors by language, price range, availability, and rating
- Booking a session creates a calendar event and video call link
- Payment processing via Stripe (hold on booking, release on session completion)
- Post-session rating system (1-5 stars + written review)
- Tutors cannot rate themselves; learners cannot rate before session completion

### Phase 12 Definition of Done

- [ ] Community writing feedback with native speaker corrections
- [ ] Tutor marketplace with booking, payment, and ratings
- [ ] Moderation tools for community content
- [ ] Terms of service for community contributions (IP ownership clear)
- [ ] Abuse reporting and blocking mechanisms

---

## Definition of Done — Global Criteria

Every phase must satisfy these criteria before being considered complete:

### Code Quality
- [ ] All code passes ESLint with zero warnings and zero errors
- [ ] TypeScript strict mode enabled with no `any` types in production code
- [ ] Test coverage >= 80% for backend services; >= 70% for frontend components
- [ ] All API endpoints have OpenAPI documentation with request/response examples
- [ ] No secrets in code; all sensitive values in environment variables

### Testing
- [ ] Unit tests for all business logic (exercise engines, FSRS scheduling, proficiency estimation)
- [ ] Integration tests for all API endpoints using a test database
- [ ] End-to-end tests for critical user flows (registration, lesson completion, review session)
- [ ] Load testing: API handles 1000 concurrent users with p95 response time < 500ms
- [ ] Accessibility: WCAG 2.1 AA compliance for all web pages

### Security
- [ ] OWASP Top 10 mitigations verified (SQL injection, XSS, CSRF, etc.)
- [ ] Rate limiting on authentication endpoints (10 attempts per minute)
- [ ] Input validation on all API endpoints using Zod schemas
- [ ] Audio recordings stored with encryption at rest
- [ ] GDPR compliance: data export and deletion endpoints for user data
- [ ] Password policy: minimum 8 characters, breach database check

### Performance
- [ ] Database queries use indexed columns (verified via EXPLAIN ANALYZE for critical paths)
- [ ] SRS due card query executes in < 50ms for a user with 10,000 review cards
- [ ] Lesson loading time < 2 seconds on 3G connection
- [ ] AI conversation first-token latency < 1 second

### Documentation
- [ ] Architecture decision records (ADRs) for major technology choices
- [ ] API documentation auto-generated from OpenAPI spec
- [ ] Deployment runbook for production environment
- [ ] Database schema diagram auto-generated from Drizzle schema
