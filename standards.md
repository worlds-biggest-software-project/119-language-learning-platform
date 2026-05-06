# Standards & API Reference

> Project: Language Learning Platform · Generated: 2026-05-06

## Industry Standards & Specifications

### Proficiency Frameworks

**CEFR — Common European Framework of Reference for Languages (2001; Companion Volume 2020)**
- Official URL: https://www.coe.int/en/web/common-european-framework-reference-languages
- Published by the Council of Europe. The 2020 Companion Volume extended the original six-level (A1–C2) framework with descriptors for mediation, online interaction, plurilingual/pluricultural competence, sign language, and gender-neutral formulations. Any platform making level-progress claims or issuing certificates should map its curriculum to these descriptors and document the alignment methodology.

**ACTFL Proficiency Guidelines & OPI (2012; OPI Familiarization Guide 2020)**
- Official URL: https://www.actfl.org/assessments/postsecondary-assessments/opi
- The American Council on the Teaching of Foreign Languages describes five major levels (Novice, Intermediate, Advanced, Superior, Distinguished) each subdivided into Low/Mid/High sub-levels. The Oral Proficiency Interview (OPI) is the standardised spoken assessment derived from these guidelines. Relevant for platforms targeting US education and government procurement, which commonly require ACTFL-aligned proficiency ratings for hiring, placement, or certification.

### E-Learning Content & Interoperability Standards

**SCORM 2004 (4th Edition) — Sharable Content Object Reference Model**
- Official URL: https://www.adlnet.gov/scorm/
- Maintained by the ADL Initiative (US Department of Defense). SCORM 2004 defines content packaging (ZIP/imsmanifest.xml), run-time communication between content and LMS (JavaScript API), and sequencing/navigation rules. Despite being superseded in sophistication by xAPI, SCORM 2004 remains the most widely required format for corporate LMS procurement and government contracts. Language learning course packages must produce valid SCORM 2004 output to win enterprise deals where SCORM compliance is mandated.

**xAPI / Experience API — IEEE 9274.1.1-2023**
- Official URL: https://xapi.com/overview/
- The successor to SCORM for activity tracking, standardised as IEEE 9274.1.1-2023 (xAPI 2.0, released October 2023). Uses Actor–Verb–Object statements (e.g., "learner completed pronunciation exercise") stored in a Learning Record Store (LRS). Supports offline mobile, simulation, game-based, and real-world activity tracking — all relevant modalities for language learning. xAPI is the preferred standard for platforms seeking richer analytics than SCORM allows.

**cmi5 — AICC/xAPI Hybrid Profile**
- Official URL: https://aicc.github.io/CMI-5_Spec_Current/
- cmi5 combines xAPI's actor-verb-object statement model with SCORM-style LMS launch and packaging. It is increasingly required by enterprise LMS platforms (Cornerstone, SAP SuccessFactors) that need trackable, packageable content. A language learning platform targeting corporate workforce training should support cmi5 alongside SCORM 2004.

**IMS LTI 1.3 — Learning Tools Interoperability**
- Official URL: https://www.imsglobal.org/spec/lti/v1p3
- 1EdTech (formerly IMS Global) standard enabling a Tool Provider (e.g., a language learning platform) to launch securely from within a Platform (Canvas, Moodle, Blackboard, D2L Brightspace) using OpenID Connect and signed JWTs. LTI Advantage adds three services: Names and Role Provisioning (roster sync), Deep Linking (content selection), and Assignment and Grade Services (grade passback). LTI 1.3 compliance is mandatory for academic institutional sales in higher education.

**IMS QTI v3.0 — Question and Test Interoperability**
- Official URL: https://www.imsglobal.org/spec/qti/v3p0/oview
- 1EdTech standard for the representation and exchange of assessment items and tests in XML. QTI v3 adds HTML5/web component support, native Computer Adaptive Testing support, and improved accessibility. Language assessment items (reading comprehension, grammar questions, listening exercises) authored in QTI v3 can be imported into any compatible LMS or assessment platform without reformatting.

### Web & Browser Standards

**Web Speech API — WICG (W3C Incubator)**
- Official URL: https://wicg.github.io/speech-api/ · MDN: https://developer.mozilla.org/en-US/docs/Web/API/Web_Speech_API
- Defines browser-native SpeechRecognition (speech-to-text) and SpeechSynthesis (text-to-speech) JavaScript interfaces. Not a formal W3C Standard but widely implemented in Chrome/Edge; Firefox support is partial; Safari support limited. Sufficient for basic in-browser pronunciation recording and TTS playback. For production-quality speech assessment, platforms supplement the Web Speech API with server-side ASR (Azure, Google, Whisper) that provides phoneme-level confidence scores the browser API does not expose.

**Unicode Standard & CLDR — Common Locale Data Repository**
- Unicode: https://unicode.org/standard/standard.html · CLDR: https://cldr.unicode.org/
- Unicode defines character encoding for all human scripts. CLDR (maintained by Unicode Consortium with contributions from IBM, Apple, Google, Microsoft) provides locale data: date/time formats, number formats, language names, calendar systems, and collation rules for 100+ languages. A platform supporting non-Latin scripts (Arabic, Chinese, Japanese, Korean, Devanagari, etc.) must implement Unicode correctly and use CLDR data for locale-specific formatting.

**OpenAPI Specification 3.2 (OAS 3.2)**
- Official URL: https://www.openapis.org/ · Swagger: https://swagger.io/specification/
- The de-facto standard for describing RESTful APIs in YAML or JSON. Any public or partner-facing API that the language learning platform exposes (progress data, user management, content delivery) should be documented using OAS 3.2 to enable automated client generation, testing, and third-party integration.

### Security & Authentication Standards

**OAuth 2.0 — IETF RFC 6749 & RFC 6750**
- Official URL: https://datatracker.ietf.org/doc/html/rfc6749
- The industry-standard protocol for delegated authorisation. Required for any third-party integration (LMS, HR system, enterprise SSO) and for the platform's own API access control. RFC 6750 defines the Bearer Token usage pattern.

**OpenID Connect 1.0 (OIDC)**
- Official URL: https://openid.net/developers/how-connect-works/
- Authentication layer built on OAuth 2.0, used for Single Sign-On (SSO). Enterprise customers require OIDC/SAML 2.0 SSO for identity federation with their IdP (Okta, Azure AD, Google Workspace). LTI 1.3 also uses OIDC as its security framework. Required for institutional and corporate tiers.

**SAML 2.0 — Security Assertion Markup Language**
- Official URL: https://www.oasis-open.org/standard/saml/
- XML-based SSO standard still dominant in enterprise HR and LMS environments. Rosetta Stone, Babbel for Business, and Busuu for Business all support SAML 2.0. A language learning platform targeting enterprise must support SAML 2.0 in addition to OIDC.

**ISO 17100:2015 — Translation Services: Requirements**
- Official URL: https://www.iso.org/standard/59149.html
- Governs quality processes for translation services: translator qualifications, translation/revision/verification workflow, and client feedback mechanisms. Relevant when the platform localises curriculum content into target languages using external translation vendors; requiring ISO 17100 certification from vendors provides an auditable quality baseline.

### Data & Algorithm Specifications

**FSRS — Free Spaced Repetition Scheduler**
- Official URL: https://github.com/open-spaced-repetition/free-spaced-repetition-scheduler
- Open-source SRS algorithm under the MIT licence, derived from the DSR model (Wozniak) and DHP model (MaiMemo). Models memory via three variables: Difficulty, Stability (storage strength), and Retrievability (retrieval probability). Standardised implementations available in Python (py-fsrs), TypeScript (ts-fsrs), Rust (fsrs-rs), Go, and Ruby. FSRS supersedes SM-2 in accuracy and is the recommended algorithm for a new platform building its own SRS without using Anki's AGPL-3.0 code.

**SM-2 Algorithm Specification**
- Official URL: https://www.supermemo.com/en/blog/application-of-a-computer-to-improve-the-results-obtained-in-working-with-the-supermemo-method (Wozniak, 1987/1990)
- The foundational spaced repetition algorithm that underpins Anki and most SRS systems. Publicly documented and in the public domain; implementable without licence restrictions. FSRS is generally preferred for new implementations as it is more accurate, but SM-2 remains acceptable for a lightweight initial implementation.

---

## Similar Products — Developer Documentation & APIs

### Microsoft Azure AI Speech Service

- **Description:** Azure's cloud speech platform covering speech-to-text (STT), text-to-speech (TTS), speaker recognition, and — uniquely relevant for language learning — Pronunciation Assessment. The Pronunciation Assessment feature scores spoken input against reference text at word, phoneme, and prosody levels, returning Accuracy, Fluency, Completeness, and Prosody scores.
- **API Documentation:** https://learn.microsoft.com/en-us/azure/ai-services/speech-service/how-to-pronunciation-assessment
- **SDKs/Libraries:** C#, Python, JavaScript, Java, Go, Swift, Objective-C via the Azure Cognitive Services Speech SDK
- **Developer Guide:** https://learn.microsoft.com/en-us/azure/ai-services/speech-service/
- **Standards:** REST (short audio); SDK-based WebSocket for streaming; OpenAPI-documented management plane
- **Authentication:** Azure API Key or Azure Active Directory (OIDC/OAuth 2.0)
- **Notes:** Pronunciation Assessment is the most feature-complete commercial API for per-phoneme pronunciation diagnosis. It returns a `PronunciationAssessmentWordResult` with phoneme-level accuracy scores, enabling the kind of articulation-level feedback absent from all current consumer language learning apps.

### Google Cloud Speech-to-Text v2

- **Description:** Google's ASR service supporting transcription in 85+ languages with specialised models for telephony, video, and medical audio. V2 introduces named Recognizers (persistent configurations) and Language Detection for multilingual audio. Does not include pronunciation scoring comparable to Azure; phoneme-level confidence is available in word-level alternatives.
- **API Documentation:** https://docs.cloud.google.com/speech-to-text/docs/reference/rest
- **SDKs/Libraries:** Python, Java, Node.js, Go, C#, Ruby, PHP via Google Cloud client libraries
- **Developer Guide:** https://cloud.google.com/speech-to-text
- **Standards:** REST and gRPC; OpenAPI 3.0 for REST surface; gRPC proto definitions published on GitHub
- **Authentication:** Google Cloud Service Account (OAuth 2.0 / OIDC)

### OpenAI Whisper / OpenAI Audio API

- **Description:** Whisper is an open-source (MIT licence) ASR model trained on 680,000 hours of multilingual audio, supporting transcription in 99 languages and translation to English. The OpenAI Audio API exposes Whisper via a REST endpoint accepting audio files up to 25 MB.
- **API Documentation:** https://developers.openai.com/api/docs/guides/speech-to-text
- **SDKs/Libraries:** openai Python library, openai Node.js library; Whisper model weights available on Hugging Face for self-hosted deployment
- **Developer Guide:** https://github.com/openai/whisper (open-source); https://developers.openai.com for hosted API
- **Standards:** REST/JSON; MIT licence for open-source model weights
- **Authentication:** OpenAI API Key
- **Notes:** Whisper's open-source weights under MIT licence are ideal for self-hosted deployments where audio privacy is a concern (learner recordings should not transit third-party servers without explicit consent). Self-hosting requires GPU infrastructure but eliminates per-call API costs and data retention risks.

### ElevenLabs Text-to-Speech API

- **Description:** High-quality neural TTS with voice cloning and multilingual support across 70+ languages (Eleven v3 model). Provides emotionally-aware speech synthesis suitable for generating native-speaker audio models for pronunciation training and generating lifelike dialogue audio for immersive listening exercises.
- **API Documentation:** https://elevenlabs.io/docs/api-reference/text-to-speech/convert
- **SDKs/Libraries:** Python SDK, JavaScript/TypeScript SDK
- **Developer Guide:** https://elevenlabs.io/docs/overview/intro
- **Standards:** REST/JSON; ISO 639-1 language codes for language enforcement; streaming via Server-Sent Events
- **Authentication:** ElevenLabs API Key
- **Notes:** ElevenLabs' Multilingual v2 and v3 models maintain consistent voice identity across languages, making them suitable for a consistent AI tutor persona that speaks the learner's target language with high naturalness.

### DeepL Translation API

- **Description:** Neural machine translation API covering 30+ language pairs with high accuracy, formal/informal register control, and glossary support. Relevant for generating translated example sentences, bilingual vocabulary lists, and localising curriculum content across language pairs.
- **API Documentation:** https://developers.deepl.com/docs
- **SDKs/Libraries:** Python (`deepl-python`), Node.js, .NET, PHP, Ruby, Java — all officially maintained by DeepL
- **Developer Guide:** https://developers.deepl.com/api-reference/translate
- **Standards:** REST/JSON; OpenAPI-documented
- **Authentication:** DeepL API Key (Free tier available; Pro tier for production)
- **Notes:** DeepL's glossary feature (custom term mappings) is useful for language-learning content where specific pedagogical vocabulary choices must be preserved in translation.

### Rosetta Stone Enterprise API (Catalyst & Fluency Builder)

- **Description:** Rosetta Stone exposes two GraphQL APIs for enterprise customers: the Catalyst API (user management, progress reporting, licence assignment/unassignment) and the Fluency Builder API (course assignment and deletion). Both APIs require enterprise subscription credentials.
- **API Documentation:** https://resources.rosettastone.com/support/SF/Resources/RosettaStoneAPIServices20220609.pdf (enterprise login required)
- **SDKs/Libraries:** None publicly documented; GraphQL queries/mutations via standard HTTP clients
- **Developer Guide:** https://support.rosettastone.com/s/article/Rosetta-Stone-Integration-Options
- **Standards:** GraphQL; SAML 2.0 for SSO
- **Authentication:** OAuth 2.0 access token (enterprise credentials)
- **Notes:** The GraphQL API design (single endpoint, flexible query) is notable for an edtech platform of this vintage. Not useful for building a competing product but informative as a reference architecture for enterprise user/licence management APIs.

### AnkiConnect (Anki REST API Plugin)

- **Description:** AnkiConnect is an Anki plugin (not an official Anki product) that exposes a local JSON-RPC API on port 8765, enabling third-party tools to add cards, query decks, and manage notes programmatically. Used by tools such as Yomichan/Yomitan (Japanese dictionary browser extensions) to push vocabulary cards into Anki.
- **API Documentation:** https://github.com/FooSoft/anki-connect
- **SDKs/Libraries:** No official SDK; standard HTTP POST with JSON body; community wrappers exist in Python, JavaScript, Rust
- **Developer Guide:** https://foosoft.net/projects/anki-connect/
- **Standards:** JSON-RPC over HTTP (localhost only by default)
- **Authentication:** Optional API key in config
- **Notes:** AnkiConnect is architecturally informative: it demonstrates the pattern of exposing a local SRS engine via a lightweight REST-like interface so external content tools (browser extensions, reading apps) can push vocabulary into a learner's review queue. A language learning platform that supports an open vocabulary export/import API following this pattern would interoperate with the existing Anki ecosystem without itself being subject to AGPL-3.0.

### Busuu for Business API

- **Description:** Busuu's enterprise platform provides an undocumented (non-public) REST API for user management, team enrolment, progress reporting, and SCORM content export. SSO integration via SAML 2.0 and Microsoft Teams integration are supported.
- **API Documentation:** Not publicly available; enterprise contract required
- **SDKs/Libraries:** None publicly documented
- **Developer Guide:** https://business.busuu.com/product
- **Standards:** SCORM 2004 for content export; SAML 2.0 for SSO; REST/JSON (internal)
- **Authentication:** SAML 2.0 / enterprise IdP
- **Notes:** Busuu's API is closed-source and gated behind enterprise agreements, which is representative of the language learning industry as a whole. There is no open interoperability standard for language learning progress data; an AI-native platform that publishes an open, OpenAPI-documented progress and user management API would be genuinely differentiated.

---

## Notes

### Gaps and Emerging Standards

- **No universal language learning data standard exists.** Unlike FHIR for health data or Open Banking APIs for financial data, there is no open, interoperable standard for representing language proficiency progress, vocabulary knowledge, or learning activity history in a portable format. Each platform maintains a proprietary data silo. A platform that publishes learner progress in an open, machine-readable format (e.g., CEFR-mapped xAPI statements with a published vocabulary) would enable a genuine ecosystem of complementary tools.
- **AI-generated audio and pronunciation models are an evolving area.** There is no published standard for representing phoneme-level pronunciation error data or corrective feedback in a machine-readable format. The closest approximation is the Azure Pronunciation Assessment `PronunciationAssessmentWordResult` schema, but this is proprietary. An open schema for pronunciation error reporting (e.g., as a JSON vocabulary over xAPI) would benefit the field.
- **CEFR continuous estimation is not yet standardised.** The CEFR framework is designed for snapshot proficiency assessment, not continuous inference from production data. Research into automated CEFR estimation (e.g., via LLM-scored writing and speech) is active but no standard methodology or API schema exists yet.
- **Whisper + FSRS stack is MIT/open-source.** A platform built on self-hosted Whisper (MIT) for ASR and FSRS (MIT) for scheduling can avoid the AGPL-3.0 restrictions of Anki and the per-call costs of Azure/Google Speech, giving full control over learner audio data. This is the recommended open-source foundation for the SRS and pronunciation feedback subsystems.
