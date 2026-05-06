# Language Learning Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source language learning platform combining conversational AI practice, adaptive spaced repetition, and continuous CEFR-aligned progress tracking.

The Language Learning Platform delivers structured, CEFR-aligned curriculum with always-available AI conversation practice, phoneme-level pronunciation coaching, and adaptive review scheduling. It is built for consumer learners, educational institutions, and corporate L&D teams who need rigorous, measurable language acquisition without paying premium per-seat fees for incumbent suites.

---

## Why Language Learning Platform?

- Incumbent leaders like Duolingo focus on vocabulary drills and gamification with limited grammar instruction in context, and free-form speaking practice is gated behind the Duolingo Max tier (~$168/year).
- Babbel, Rosetta Stone, and Busuu offer structured lessons but ship limited or fixed AI personalisation; conversational practice typically requires booking human teachers separately.
- Tutoring marketplaces (Preply, italki) charge $15–60/hour and provide no structured curriculum, SRS, or progress tracking between sessions.
- Anki is free and powerful, but its AGPL-3.0 licence prevents proprietary embedding, its UX lags consumer apps, and it offers no speech recognition, instructional content, or AI features.
- Most platforms have weak enterprise features: no LMS LTI integration, no SCORM export from many vendors, and informal CEFR alignment that institutional buyers cannot audit.

---

## Key Features

### Structured Curriculum and Vocabulary

- CEFR-aligned lesson delivery (A1–B2 minimum) with documented level descriptors
- Vocabulary, grammar, and speaking exercises across 10+ languages from launch
- Spaced repetition vocabulary scheduling using SM-2 or FSRS implemented from scratch
- Mobile apps (iOS and Android) with offline lesson access
- Progress dashboard showing words known, lessons completed, study time, and estimated CEFR level

### AI Conversation and Pronunciation

- AI conversation partner for free-form speaking practice with contextual error correction
- Register and vocabulary adaptation tied to the learner's current CEFR level
- Speech recognition with pronunciation accuracy scoring at word and sentence level
- Phoneme-level pronunciation diagnosis identifying specific articulation errors with corrective guidance
- Personalised content generation (reading passages, listening clips, scenarios) grounded in learner-declared interests

### Continuous Assessment

- Continuous CEFR proficiency estimation derived from production activities across all exercises
- Live proficiency signal for learners and institutions without requiring standalone formal tests
- Adaptive SRS rescheduling based on demonstrated comprehension in production rather than self-rated recall

### Enterprise and Education

- Admin dashboard with team enrolment and progress reporting
- SCORM export and LMS LTI integration for higher education and corporate L&D
- SSO (SAML) for enterprise deployments
- Auditable CEFR alignment methodology suitable for institutional procurement

### Optional Community and Tutoring (Backlog)

- Native speaker community feedback on learner-submitted writing and speaking samples
- In-platform booking or marketplace integration for live human tutor sessions
- Comprehensible-input reader with authentic text import and clickable vocabulary lookup
- Official CEFR certificate issuance via accredited testing partner

---

## AI-Native Advantage

Unlike incumbents that bolt LLM features onto fixed lesson trees, this platform treats AI as the core interaction surface: an always-available conversation partner that responds to free-form utterances, corrects errors in context, and adapts register to the learner's CEFR level. Speech recognition goes beyond "try again" to identify the specific phoneme error, contrast it with the learner's native phoneme inventory, and explain the articulation adjustment needed. Spaced repetition is driven by inferred comprehension from real production rather than self-rated recall, and proficiency is estimated continuously from practice data rather than via disruptive standalone tests.

---

## Tech Stack & Deployment

The platform targets self-hosted and cloud deployment modes, with mobile (iOS, Android) and web clients sharing offline-capable lesson content. CEFR is the primary curriculum framework, with ACTFL alignment for US contexts; SCORM and IMS LTI / xAPI integration support institutional LMS deployments (Canvas, Blackboard). Speech features are built on the W3C Web Speech API where available and server-side ASR/TTS otherwise. SRS uses an SM-2 or FSRS implementation written from scratch to avoid AGPL-3.0 obligations from the Anki codebase. Unicode and CLDR underpin script and locale handling across the supported language set.

---

## Market Context

The online language learning market was valued at ~$21 billion in 2025, projected to reach $116.9 billion by 2033 (CAGR ~17.9%); Duolingo alone generated $748M in 2024 revenue, and Babbel reported ~€352M. Consumer pricing spans free ad-supported tiers through $7–20/month premium subscriptions and $15–60/hour live tutoring; enterprise workforce training typically runs $50–200/employee/year. Primary buyers are consumer learners, corporate HR/L&D teams, K-12 and higher education world-language programs, government and military, and immigration services requiring standardised proficiency testing.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
