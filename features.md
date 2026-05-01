# Language Learning Platform — Feature & Functionality Survey

> Candidate #119 · Researched: 2026-05-01

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Duolingo | Gamified language app (40+ languages) | Proprietary Freemium | duolingo.com |
| Babbel | Structured conversational lessons with speech recognition | Proprietary Subscription | babbel.com |
| Rosetta Stone (IXL Learning) | Immersive method with speech recognition | Proprietary Freemium/B2B | rosettastone.com |
| Busuu | Structured courses + native speaker community feedback | Proprietary Freemium | busuu.com |
| Pimsleur (Simon & Schuster) | Audio-first spaced repetition program | Proprietary Subscription | pimsleur.com |
| italki | Live tutoring marketplace (40,000+ tutors) | Marketplace | italki.com |
| Preply | Live 1-on-1 tutoring marketplace | Marketplace | preply.com |
| Anki | Open-source SRS flashcard tool | OSS — AGPL-3.0 | apps.ankiweb.net |
| LingQ | Comprehensible-input method with content library | Proprietary Freemium | lingq.com |
| Speechling | AI pronunciation feedback + human coach review | Proprietary Freemium | speechling.com |

## Feature Analysis by Solution

### Duolingo

**Core features**
- Gamified lesson delivery across 40+ languages with streak, heart, and XP mechanics
- Spaced repetition scheduling for vocabulary review
- AI pronunciation scoring with phoneme-level accuracy feedback
- Duolingo Max (AI tier): "Explain My Answer" and "Roleplay" features powered by GPT-4
- Listening, speaking, reading, and writing exercise types within lessons
- Official language proficiency test (Duolingo English Test) accepted by 5,000+ institutions
- Family and classroom plans for group management

**Differentiating features**
- Largest language learning user base: 50.5 million DAU (Q3 2025)
- Duolingo Max Roleplay: open-ended AI conversation practice with contextual feedback
- DET (Duolingo English Test): end-to-end test delivery via webcam with AI proctoring; $65 vs $250+ for IELTS/TOEFL

**UX patterns**
- Mobile-first; average session designed for 5–10 minutes (commuter format)
- Streak counter and daily XP goal create daily habit loop
- Leaderboard comparing XP against 30 weekly peers drives competitive engagement
- Hearts system limits mistakes per session (free tier); Super removes hearts

**Integration points**
- Duolingo for Schools: teacher dashboard with class roster and progress tracking
- No LMS LTI integration; standalone mobile/web app
- Duolingo English Test results delivered via secure PDF to institutions

**Known gaps**
- Vocabulary and short-sentence focus; limited instruction on grammar rules in context
- Free-form speaking practice is limited outside the Max subscription tier
- Not suitable for structured institutional course delivery (no curriculum authoring, LTI, or grade passback)
- CEFR alignment is informal; learning objectives are not mapped to CEFR descriptors in the core product

**Licence / IP notes**
- Proprietary; NASDAQ-listed (DUOL); standard consumer privacy policy; Duolingo English Test data subject to separate biometric processing terms

---

### Babbel

**Core features**
- Structured lessons with explicit grammar instruction and vocabulary building
- Speech recognition for pronunciation practice (proprietary engine)
- Review manager: spaced repetition for vocabulary previously encountered in lessons
- Live online group classes (Babbel Live) with human teachers in small groups
- 14 languages available; lessons designed by in-house linguistics team
- Offline lesson download for mobile

**Differentiating features**
- Only major app combining self-study with optional live group classes under one subscription
- Lesson content authored by professional linguists and native speaker reviewers
- Paid-only model (no free tier beyond trial); positioning as premium quality over gamification

**UX patterns**
- Lesson browser organised by topic and difficulty (beginner, intermediate)
- Audio-visual dialogue presentation followed by interactive exercises and speech recording
- Review section distinct from lessons for dedicated vocabulary practice

**Integration points**
- Babbel for Business: admin dashboard, team enrolment, and basic reporting
- SSO available for enterprise deployments
- No LMS LTI integration

**Known gaps**
- Limited AI personalisation; lesson sequence is largely fixed
- No AI conversation partner; Babbel Live requires booking with human teachers
- 14 languages only; narrower than Duolingo or Rosetta Stone

**Licence / IP notes**
- Proprietary; German company; GDPR compliant; rejected Duolingo acquisition offer ($3B, 2021); remains independent

---

### Rosetta Stone (IXL Learning)

**Core features**
- Immersive method: no translation; target language used from lesson 1
- Speech recognition (TruAccent) for pronunciation correction without phoneme-level explanation
- Structured curriculum across 25 languages from beginner to intermediate
- Live tutoring sessions (Rosetta Stone Live) available as add-on
- Offline mode on mobile
- Enterprise and education tier with administrator reporting

**Differentiating features**
- Immersive methodology is pedagogically distinctive; removes native language as a crutch
- Longest-established brand in digital language learning (founded 1992); high name recognition in enterprise procurement
- Education tier includes CEFR-mapped curriculum and administrator dashboards for institutional buyers

**UX patterns**
- Core lessons: image + audio matching; no text translation in the immersive tier
- Pronunciation recording with visual waveform comparison against native speaker model
- Progress tracker showing units and milestones completed

**Integration points**
- Education platform: LMS integration available (Canvas, Blackboard) via SCORM or LTI
- Enterprise: SAML SSO and admin reporting dashboard
- API access for enterprise seat management

**Known gaps**
- Immersive method works poorly for learners who need grammatical explanation (common adult learner preference)
- AI features are limited; no LLM conversation partner or adaptive personalisation
- Content has not been significantly refreshed in recent years; some learners report it feeling dated

**Licence / IP notes**
- Proprietary; acquired by Cambium Learning / IXL Learning (2021); standard enterprise DPA

---

### Busuu

**Core features**
- Structured CEFR-aligned courses (A1–B2) across 12 languages
- Native speaker community feedback: submit writing or speaking exercises for correction by native speakers
- AI-powered grammar coaching: Grammar Patrol identifies and explains grammatical errors
- Study plan personalisation based on goals and available time per week
- Offline course download on mobile
- Business tier with team management, progress reporting, and SCORM content export

**Differentiating features**
- Native speaker community feedback is unique among structured-course apps; learners get corrections from humans, not just algorithms
- Grammar Patrol AI provides explicit grammatical explanations rather than just marking answers wrong
- CEFR certification (McGraw-Hill partnership) available within the app

**UX patterns**
- Lesson feed with recommended next activity based on study plan
- Correction interface: learner submits a recording or paragraph; native speaker community responds with inline corrections
- Streak and weekly goal system similar to Duolingo but without hearts mechanism

**Integration points**
- Busuu for Business: admin portal, SSO, and basic LMS integration
- SCORM content export for enterprise LMS deployment
- No LTI integration for higher education

**Known gaps**
- Community feedback response times are unpredictable (volunteer-driven)
- Limited above B2; no C1–C2 content
- AI conversation partner is not available; limited speaking practice beyond pronunciation exercises

**Licence / IP notes**
- Proprietary; acquired by Chegg for ~$436M (2022); GDPR compliant; privacy policy governs community content submitted by learners

---

### Pimsleur (Simon & Schuster)

**Core features**
- Audio-first 30-minute lesson units using spaced interval recall
- Conversational dialogue focus; heavy emphasis on pronunciation and oral production
- 50+ languages available including less commonly taught languages
- Driving mode: hands-free audio lessons for commuters
- Reading lessons as a supplement to audio core
- Speak Easy AI: in-app AI conversation practice (recent addition)

**Differentiating features**
- Only major platform with audio as the primary modality rather than a supplement
- Spaced interval recall methodology (Graduated Interval Recall) is rigorously implemented in audio format
- Most extensive less-commonly-taught language selection (Tagalog, Swahili, Ojibwe, etc.)

**UX patterns**
- Listen and respond; audio plays a prompt, learner pauses and speaks, audio confirms or corrects
- Minimal visual interface; designed for eyes-off consumption
- Progress bar by lesson number; no gamification mechanics

**Integration points**
- Pimsleur for Business: bulk seat management
- No LMS integration; standalone app
- Audible integration for some Pimsleur content libraries

**Known gaps**
- No written literacy development; reading and writing are secondary
- No human tutoring; community feedback; or social features
- Limited analytics for institutional deployment

**Licence / IP notes**
- Proprietary; Simon & Schuster (Penguin Random House Group); standard consumer privacy policy

---

### italki

**Core features**
- Marketplace of 15,000+ professional teachers and community tutors across 150+ languages
- Session booking, video call integration, and payment processing via the platform
- Teacher profile system with ratings, hourly rates, intro video, and availability calendar
- Notebook: learner can post a written text in the target language for free community correction
- Community Q&A for grammar and vocabulary questions

**Differentiating features**
- Broadest language selection of any tutoring marketplace (150+ languages)
- Community tutor tier provides affordable informal conversation practice ($5–15/hour)
- Free notebook correction by native speakers provides written practice without paying for a session

**UX patterns**
- Teacher search with language, price, availability, and specialty filters
- In-platform video calling (or external Skype/Zoom by arrangement)
- Session notes and homework exchange between teacher and student via platform messaging

**Integration points**
- No LMS integration; standalone marketplace
- Paypal and major credit cards for payment processing
- No institutional or enterprise tier

**Known gaps**
- No structured curriculum; entirely dependent on individual teacher quality and consistency
- No progress tracking, vocabulary review, or SRS between sessions
- Teacher quality is variable; limited quality assurance beyond ratings

**Licence / IP notes**
- Proprietary; Chinese-owned company (acquired by Takoda and rebranded as italki Ltd.); GDPR compliance for EU users; payment data via Stripe/PayPal

---

### Anki

**Core features**
- Spaced repetition flashcard system with configurable SM-2 algorithm
- Unlimited custom decks and card types (text, image, audio, LaTeX)
- Community shared decks: AnkiWeb hosts 10,000+ shared decks across 100+ languages
- Cross-platform sync: desktop (Windows/macOS/Linux) and AnkiWeb (browser) are free; AnkiMobile (iOS) is $25
- Add-on ecosystem: 3,500+ community plugins extending functionality
- Mature algorithm supported by decades of academic SRS research

**Differentiating features**
- Most customisable SRS implementation available; power users can tune algorithm parameters
- Community deck ecosystem is uniquely rich; language learners share vocabulary decks for virtually every language pair
- Completely free on desktop and web; no subscription or per-user fee

**UX patterns**
- Minimalist card review interface: show question → recall → flip to reveal answer → rate difficulty (Again/Hard/Good/Easy)
- Deck browser and statistics panel showing forecast review burden and retention rate
- Card editor with rich media support

**Integration points**
- AnkiConnect plugin: REST API enabling third-party apps to add cards to Anki programmatically
- Export to text file, APKG package, or CSV for backup and sharing
- No LMS integration; no institutional tier

**Known gaps**
- No structured curriculum, instructional content, or language-specific guidance
- UI is functional but unattractive; significantly below consumer app UX standards
- No speech recognition or pronunciation feedback
- No AI content generation or adaptive difficulty beyond standard SRS scheduling

**Licence / IP notes**
- AGPL-3.0: Anki desktop and AnkiWeb are AGPL-3.0. The iOS app (AnkiMobile) is proprietary and sold on the App Store to fund development. The Android app (AnkiDroid) is GPL-3.0 (separate team). AGPL-3.0 is the strongest copyleft licence: any service that runs modified Anki code over a network (e.g., a hosted web app using Anki's scheduler) must publish the modified source code under AGPL-3.0. Building a commercial language learning product that embeds or modifies Anki's scheduler and serves it over the web requires either releasing all modifications under AGPL-3.0 or obtaining a commercial licence (which Anki's developer Damien Elmes does not generally offer). Platforms wishing to use SRS in a proprietary product should implement their own SM-2 or FSRS algorithm from the published specification rather than using Anki code directly.

---

### LingQ

**Core features**
- Comprehensible-input method: learners read and listen to authentic content in the target language
- LingQ: learners highlight unknown words; system tracks them and schedules review
- Vocabulary tracking: 9-level word status system from new to known
- Content library: 200,000+ imported and native lessons across 20 languages
- Import tool: learners can import any text, YouTube video, or podcast for study
- Known words counter as the primary progress metric

**Differentiating features**
- Only major platform built around comprehensible input theory (Krashen) rather than drill and quiz
- Content import feature allows learners to use any authentic material as a learning resource
- Most flexible vocabulary tracking system; known-words metric aligns with research on reading fluency thresholds

**UX patterns**
- Reader interface with word highlighting; click a word to see translation and add a LingQ
- Audio player synced to sentence text for reading-while-listening
- Vocabulary review mode with SRS scheduling for flagged LingQs

**Integration points**
- Browser extension for importing web content
- YouTube import for video lesson creation
- No LMS integration; standalone platform

**Known gaps**
- Exclusively reading and listening; no speaking or writing practice features
- Grammar instruction is absent by design; learners must supplement elsewhere
- UX is functional but not polished; limited visual design investment

**Licence / IP notes**
- Proprietary; Canadian company; standard consumer privacy policy; imported content copyright remains with original creators

---

### Speechling

**Core features**
- Learner records audio responses to sentence prompts in the target language
- AI pronunciation analysis provides initial automated feedback
- Human native-speaker coaches listen to recordings and provide written and audio correction (paid tier)
- 11 languages available
- Sentence library organised by difficulty and topic
- Progress tracking showing sessions completed and coach feedback received

**Differentiating features**
- Combines AI speed with human nuance: AI flags issues; human coach explains cultural and prosodic context the AI misses
- Only platform offering unlimited human coach recordings for a flat monthly fee ($30/month)
- Pronunciation-centric; does not attempt to be a full language course

**UX patterns**
- Record → listen to AI score → optionally wait for human coach response
- Coach inbox showing all awaiting and reviewed recordings
- Minimal gamification; focused on deliberate speaking practice

**Integration points**
- No LMS integration; standalone web/mobile app
- No API or third-party integrations

**Known gaps**
- Narrow scope: pronunciation and speaking only; no vocabulary, grammar, or reading instruction
- 11 languages limit applicability
- Human coach response times are not guaranteed; can range from hours to days

**Licence / IP notes**
- Proprietary; US company; standard privacy policy; audio recordings stored on platform servers; GDPR compliance for EU users

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Vocabulary and phrase delivery with translation and audio pronunciation
- Spaced repetition scheduling for vocabulary review
- Speech recognition for pronunciation feedback
- Progress tracking (lessons completed, vocabulary count, streak or study time)
- Mobile app with offline content access
- At least 5+ languages supported from day one

### Differentiating Features
- AI conversational roleplay partner with contextual error feedback (Duolingo Max)
- Native speaker community correction for written and spoken output (Busuu, italki, LingQ)
- Audio-first immersive learning without visual text scaffolding (Pimsleur)
- Comprehensible-input library with content import (LingQ)
- Human pronunciation coaching combined with AI pre-screening (Speechling)
- Live tutoring marketplace (italki, Preply) with 150+ languages
- Fully configurable community SRS (Anki)
- Official CEFR-aligned proficiency certification (Busuu/McGraw-Hill, Duolingo English Test)

### Underserved Areas / Opportunities
- Phoneme-level pronunciation coaching explaining mouth position and articulation mechanics rather than just "try again"
- CEFR-continuous proficiency estimation from all production activities without requiring a separate formal test
- Adaptive SRS driven by demonstrated comprehension in production (not self-rated recall)
- Personalised content generation aligned to individual interests (sports, cooking, film) in the target language
- Corporate language training with LMS integration, SCORM export, and compliance reporting (most platforms provide weak enterprise features)
- Less-commonly-taught language support beyond the top 15 (Pimsleur is the only app with serious breadth)

### AI-Augmentation Candidates
- Always-available AI conversation partner simulating authentic dialogue at the learner's CEFR level with register adaptation
- Phoneme-level pronunciation diagnosis identifying the specific articulation error and providing native-language-contrasted guidance
- CEFR-continuous proficiency estimation from all exercises as a live signal for learners and institutions
- Personalised content generation (reading passages, listening clips, conversation scenarios) grounded in learner-declared interests
- Adaptive SRS rescheduling based on demonstrated comprehension in production rather than self-rated recall difficulty

## Legal & IP Summary

- **Anki is AGPL-3.0**: any commercial platform that runs modified Anki code as a networked service (web or API) must publish modifications under AGPL-3.0. Building a proprietary SRS on Anki's codebase without licensing it commercially from the author is not viable for a closed-source product. The FSRS (Free Spaced Repetition Scheduler) algorithm is separately published under a permissive licence and can be implemented from scratch in a proprietary product.
- **Duolingo English Test**: uses biometric data (facial recognition for identity verification, gaze detection); GDPR Art. 9 special-category data; processing requires explicit consent and a lawful basis; DET is not administered inside the EU without local compliance checks.
- **Community content (italki notebooks, Busuu corrections, Anki shared decks)**: user-generated content raises IP questions; platforms must clearly licence incoming content from users to display and redistribute it; learner corrections by native speakers may constitute creative works.
- **AI-generated language learning content**: LLM-generated texts in target languages may contain errors, particularly in morphologically complex or low-resource languages; editorial review workflows are required for published curriculum.
- **CEFR and ACTFL alignment claims**: vendors making CEFR-level claims without formal alignment studies risk misleading learners and institutional buyers; alignment validation methodology should be documented and auditable.

## Recommended Feature Scope

**Must-have (MVP)**:
- Structured lesson delivery with vocabulary, grammar, and speaking exercises across at least 10 languages
- Spaced repetition vocabulary scheduling (SM-2 or FSRS algorithm implemented from scratch; not Anki code)
- Speech recognition with pronunciation accuracy scoring at word and sentence level
- Mobile app (iOS and Android) with offline lesson access
- CEFR-aligned curriculum (A1–B2 minimum) with clearly documented level descriptors
- Progress dashboard: words known, lessons completed, study time, and estimated CEFR level

**Should-have (v1.1)**:
- AI conversation partner for free-form speaking practice with contextual error correction and CEFR level adaptation
- Phoneme-level pronunciation feedback identifying specific articulation errors with corrective guidance
- Continuous CEFR proficiency estimation from production activities (not just formal tests)
- Corporate/enterprise tier with admin dashboard, team enrolment, SCORM export, and LMS LTI integration
- Personalised content generation based on learner-declared interests and current proficiency level

**Nice-to-have (backlog)**:
- Native speaker community feedback on learner-submitted writing and speaking samples
- Live tutoring marketplace integration or in-platform booking for human teacher sessions
- Comprehensible-input reader with authentic text import and clickable vocabulary lookup
- Official CEFR certificate issuance via partnership with an accredited testing provider
- Less-commonly-taught language expansion beyond the initial language set using community-contributed curriculum
