<!-- SYNC IMPACT REPORT
Version change: (none) → 1.0.0 (initial ratification)
Added sections: Core Principles (I–VI), Technical Constraints, MVP Scope Contract, Governance
Removed sections: N/A (new file)
Templates reviewed:
  ✅ plan-template.md — Constitution Check gate aligns with principles below; no structural changes needed
  ✅ spec-template.md — Mandatory sections (User Scenarios, Requirements, Success Criteria) align; no changes needed
  ✅ tasks-template.md — Task phase structure (Setup → Foundational → User Stories → Polish) aligns; no changes needed
Follow-up TODOs: None. All placeholders resolved.
-->

# Inkwell Constitution

## Core Principles

### I. Formatting Semantics Are the Product

OCR output MUST capture structural and visual metadata alongside raw text. Highlights, strikethroughs, indentation, spatial groupings, and doodle presence are first-class data — not noise to discard. Any pipeline that returns only raw text is incomplete. Formatting metadata MUST be represented as structured fields (not free-form strings) so downstream AI and Notion integrations can act on them reliably.

**Rationale**: Every competitor extracts raw text only. Formatting semantics are Inkwell's defensible differentiator. Degrading them to prose descriptions eliminates the moat.

### II. Handwriting Calibration is a First-Class Feature

The calibration system MUST be presented as a distinct, named user experience — not a hidden background process. Onboarding MUST collect 3–5 writing samples before first capture. Post-capture correction UI MUST feed signals back into the calibration model. Calibration state MUST be persisted and versioned per user.

**Rationale**: Personalized OCR accuracy is what turns a generic vision API into a product users trust. Without this, Inkwell is just a wrapper around a commodity API.

### III. Notion is the Database

All journal entries MUST be stored as Notion pages. Inkwell MUST NOT maintain a proprietary long-term storage layer for entry content. Structured content (text blocks, formatting metadata, tags, entity relations) MUST map to native Notion constructs: content blocks, page properties, and page relations.

**Rationale**: Notion-first means zero new ecosystem lock-in for users. Building a parallel database breaks the value proposition and doubles the maintenance burden.

### IV. Every Entry Feeds the Knowledge Graph

On each successful capture, the AI MUST attempt to link the new entry to existing Notion pages by topic. Auto-tagging (themes and named entities → Notion properties) and entry-to-entry relations (→ Notion relation properties) MUST be produced and applied atomically with page creation — not deferred to a background job.

**Rationale**: The knowledge graph compounds in value with every entry. Deferring linking until "enough corpus exists" means it never ships; it must be live from entry one.

### V. PWA-Only Distribution

Inkwell MUST function as a Progressive Web App. The camera capture flow MUST use the browser's MediaDevices API (`getUserMedia`). There MUST be no React Native, Capacitor, Cordova, or other native wrapper dependency in the MVP. The app MUST be installable on iOS and Android home screens via the browser's "Add to Home Screen" flow.

**Rationale**: PWA eliminates app store review cycles, aligns with the team's web background, and provides a single codebase for all platforms. Native bridges introduce complexity that has no payoff at MVP scale.

### VI. MVP Scope is Non-Negotiable

The MVP MUST ship exactly these six capabilities and nothing else:
1. Photo capture via browser camera API
2. Formatting-aware OCR (text + highlights + strikethroughs + doodles as metadata)
3. Handwriting calibration onboarding flow
4. Notion page creation with structured content blocks
5. Auto-tagging (AI extracts themes/entities → Notion properties)
6. Entry-to-entry relations (link new entries to existing pages by topic)

Features explicitly deferred until a user corpus exists: pattern/insight digests, visual knowledge graph UI, Obsidian integration, doodle semantic interpretation, multi-user features.

**Rationale**: The risk of building without corpus data is real. Insights require data. Ship the data pipeline first.

## Technical Constraints

- **Platform**: Progressive Web App (TypeScript, modern browser APIs)
- **Storage**: Notion API exclusively — no proprietary backend database
- **Vision/OCR**: Claude vision or GPT-4o vision API for OCR and formatting detection
- **AI features**: Claude or GPT-4o for tagging, entity extraction, and entry linking
- **Camera**: Browser `MediaDevices.getUserMedia()` / `ImageCapture` API
- **Authentication**: Required for Notion OAuth; no custom auth system
- **Offline**: Capture queue is acceptable; full offline-first is deferred post-MVP

## MVP Scope Contract

| Capability | In MVP | Deferred |
|------------|--------|----------|
| Photo capture (camera API) | ✅ | — |
| Formatting-aware OCR | ✅ | — |
| Calibration onboarding (3–5 samples) | ✅ | — |
| Inline post-capture correction UI | ✅ | — |
| Notion page creation (structured blocks) | ✅ | — |
| Auto-tagging via AI | ✅ | — |
| Entry-to-entry relations | ✅ | — |
| Pattern/insight digest | — | Post-MVP |
| Visual knowledge graph UI | — | Post-MVP |
| Obsidian integration | — | Post-MVP |
| Doodle semantic interpretation | — | Post-MVP |
| Multi-user / team features | — | Post-MVP |

## Governance

The constitution supersedes all other project guidelines. Any feature, architectural decision, or technical choice that conflicts with Principles I–VI requires an explicit amendment — not a workaround.

**Amendment procedure**: Update this file via `/speckit.constitution`, increment the semantic version (MAJOR for principle removal/redefinition, MINOR for new principle, PATCH for clarifications), and update `LAST_AMENDED_DATE`. All dependent templates must be reviewed after each amendment.

**Compliance**: Every spec, plan, and task breakdown MUST include a Constitution Check section validating alignment with all six principles before implementation proceeds.

**Version**: 1.0.0 | **Ratified**: 2026-04-10 | **Last Amended**: 2026-04-10
