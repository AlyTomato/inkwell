# Feature Specification: Inkwell MVP

**Feature Branch**: `001-inkwell-mvp`
**Created**: 2026-04-10
**Status**: Draft
**Input**: User description — Inkwell PWA: formatting-aware OCR journaling with Notion knowledge graph

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Capture & Formatting-Aware OCR (Priority: P1)

A user opens the Inkwell app on their phone, points the camera at a handwritten journal page, and triggers a capture. The app processes the photo and returns a structured result containing: the full text of the entry, plus a layer of formatting annotations — which passages are highlighted, which are struck through, where paragraph breaks and indentation occur, and where non-text marks (doodles) appear on the page. The user can review the OCR result and make inline corrections before confirming.

**Why this priority**: This is the core input loop. Without accurate, formatting-rich capture, nothing downstream (Notion publishing, knowledge graph) has meaningful data to work with. Every other story depends on this pipeline running.

**Independent Test**: Can be tested standalone by providing a handwritten journal photo and verifying the structured output includes (a) transcribed text, (b) highlight annotations on the correct passages, (c) strikethrough annotations on the correct passages, (d) doodle presence markers, and (e) structural indicators like paragraph breaks — without any Notion integration required.

**Acceptance Scenarios**:

1. **Given** a photo of a handwritten page with a highlighted sentence, **When** the user triggers capture, **Then** the OCR output identifies the highlighted passage separately from unhighlighted text.
2. **Given** a photo containing struck-through words, **When** OCR completes, **Then** the struck-through words are flagged as deleted/annotated, not silently discarded.
3. **Given** a photo with a small doodle in the margin, **When** OCR completes, **Then** the output includes a doodle indicator noting its approximate location on the page.
4. **Given** an OCR result with an incorrect word, **When** the user taps to correct it, **Then** the correction is applied to the structured result and the correction is recorded as a calibration signal.
5. **Given** the user submits an empty or completely illegible photo, **When** OCR attempts to process it, **Then** the app surfaces a clear error and prompts the user to retry.

---

### User Story 2 — Handwriting Calibration Onboarding (Priority: P2)

A new user completing their first setup sees a calibration flow before making their first capture. They are prompted to photograph 3–5 sample writing prompts provided by the app. After completing the samples, the app acknowledges that it has built a baseline model for their handwriting. Going forward, every inline correction the user makes in the post-capture review screen feeds back into the model, improving accuracy over time.

**Why this priority**: Calibration is Inkwell's defensible moat. Without it, OCR accuracy depends entirely on the generic vision model's out-of-the-box performance on the user's specific handwriting style. The onboarding experience sets expectations and gates first real use.

**Independent Test**: Can be tested standalone by completing the onboarding flow with 3 samples and then running OCR on a sample from the same user. Accuracy on calibrated input MUST exceed accuracy on uncalibrated input for a recognizably non-standard handwriting style.

**Acceptance Scenarios**:

1. **Given** a first-time user opens the app, **When** they complete authentication, **Then** the calibration onboarding flow is presented before the main capture screen.
2. **Given** the calibration flow is active, **When** the user photographs all required sample prompts, **Then** the app confirms calibration is complete and transitions to the main capture screen.
3. **Given** the calibration flow is active, **When** the user has photographed at least 3 samples but fewer than 5, **Then** the app offers the option to skip remaining samples and proceed.
4. **Given** a calibrated user makes a post-capture correction (e.g., fixing a misread word), **When** the correction is submitted, **Then** the correction is recorded and associated with the user's calibration profile.
5. **Given** a returning user who has completed calibration, **When** they open the app, **Then** the calibration onboarding flow is NOT shown again.

---

### User Story 3 — Notion Publishing with Auto-Tagging & Entry Relations (Priority: P3)

After reviewing and confirming an OCR result, the user publishes the entry. The app creates a structured Notion page containing the entry content in readable blocks — with formatting semantics preserved (highlights called out, strikethroughs noted, doodles acknowledged). The page is auto-tagged with AI-extracted themes and named entities as Notion properties. The app then identifies existing Notion pages topically related to the new entry and creates Notion relation links between them, seeding the knowledge graph.

**Why this priority**: This is the value-delivery endpoint — the reason the capture exists. Without Notion publishing, Inkwell is just a camera app. The tags and relations are what differentiate it from a simple Notion clipper.

**Independent Test**: Can be tested standalone by providing a structured OCR result (from Story 1) and verifying that (a) a Notion page is created with correct content blocks, (b) formatting metadata is represented in the page, (c) at least one tag/entity property is set, and (d) if related entries exist in the database, at least one relation link is created.

**Acceptance Scenarios**:

1. **Given** a confirmed OCR result, **When** the user taps "Publish to Notion," **Then** a Notion page is created containing the entry text formatted as readable paragraph blocks.
2. **Given** the entry contains highlighted passages, **When** the Notion page is created, **Then** highlighted passages are represented as a distinct block type (e.g., callout), not silently merged with regular text.
3. **Given** the AI identifies themes and named entities in the entry (e.g., "anxiety," "therapy session," "goal setting"), **When** the page is created, **Then** those are applied as Notion page properties (tags or multi-select).
4. **Given** existing Notion pages in the user's database are topically related to the new entry, **When** the page is created, **Then** Notion relation links are added connecting the new entry to the related ones.
5. **Given** no existing entries match the topic of the new entry, **When** the page is created, **Then** the page is still created successfully with tags applied, and the relations property is left empty (not an error).
6. **Given** the Notion API is unreachable at publish time, **When** the user taps "Publish," **Then** the app surfaces a clear error and preserves the confirmed OCR result locally so the user can retry without re-capturing.

---

### Edge Cases

- What happens when the camera captures a partially out-of-focus or motion-blurred photo?
- How does the app handle pages with mixed printing and cursive handwriting in the same entry?
- What if a page is photographed at a sharp angle (keystoning / perspective distortion)?
- What happens if the user's Notion workspace lacks the required database schema (missing tag/relation properties)?
- How does the app behave when OCR returns zero text (blank page, non-text image)?
- What if the Notion API returns a partial success (page created but properties not applied)?
- What happens to the calibration model if the user makes conflicting corrections over time?
- How is a user's calibration data handled if they clear app storage or reinstall?

---

## Requirements *(mandatory)*

### Functional Requirements

**Capture & OCR**

- **FR-001**: The app MUST allow users to capture a photo of a handwritten page using the device camera without leaving the browser.
- **FR-002**: OCR output MUST include the full transcribed text of the entry.
- **FR-003**: OCR output MUST identify highlighted passages as a distinct annotation type, preserving their position relative to the surrounding text.
- **FR-004**: OCR output MUST identify struck-through text as a distinct annotation type (not discarded or merged with normal text).
- **FR-005**: OCR output MUST indicate the presence and approximate page location of non-text marks (doodles/drawings).
- **FR-006**: OCR output MUST preserve structural information: paragraph breaks, indentation levels, and section spacing.
- **FR-007**: Users MUST be able to review and make inline corrections to OCR output before confirming publication.
- **FR-008**: The app MUST record every user correction as a calibration feedback signal tied to the user's profile.
- **FR-028**: OCR output MUST include a `symbols` array as a reserved top-level field alongside `text`, `formatting`, and `doodles`. For MVP this field always returns an empty array. The field MUST be present in the output contract from day one to avoid a breaking schema change when Editorial Marks Intelligence ships in a future phase.

**Calibration**

- **FR-009**: First-time users MUST complete a calibration onboarding flow before accessing the main capture screen.
- **FR-010**: The calibration onboarding flow MUST collect between 3 and 5 handwriting samples via camera capture.
- **FR-011**: Users MUST be able to skip remaining calibration samples after completing the minimum of 3.
- **FR-012**: Calibration completion state MUST be persisted across sessions so returning users are not re-prompted.
- **FR-013**: The calibration model MUST update progressively as the user submits post-capture corrections.
- **FR-014**: The calibration model MUST be stored on-device using browser IndexedDB. No server-side sync is required for MVP. Cross-device calibration portability is explicitly deferred to v2.

**Notion Integration**

- **FR-015**: Confirmed OCR results MUST be published to Notion as a new page in a designated database.
- **FR-016**: The Notion page MUST represent the entry's text using native Notion paragraph blocks.
- **FR-017**: Highlighted passages MUST be represented in the Notion page as a visually distinct block type (e.g., callout block) rather than merged with surrounding text.
- **FR-018**: Struck-through text MUST be represented in the Notion page in a way that preserves its "annotated/deleted" meaning.
- **FR-019**: Doodle regions MUST be cropped from the captured photo and embedded as Notion image blocks at the position they appear in the entry flow. The OCR output MUST include the bounding coordinates of each detected doodle region to enable cropping. Text callouts or property flags are not sufficient — the actual drawing MUST be preserved in Notion.
- **FR-020**: The app MUST use AI to extract themes and named entities from each confirmed entry.
- **FR-021**: Extracted themes and entities MUST be applied to the Notion page as a multi-select tag property.
- **FR-022**: On each publish, the app MUST query existing Notion pages and identify topically related entries.
- **FR-023**: The app MUST create Notion relation links from the new entry to related existing entries.
- **FR-024**: Notion page creation, tag application, and relation linking MUST either all succeed or surface a clear error — partially applied state MUST NOT be silently accepted.
- **FR-025**: If the Notion API is unreachable, the confirmed OCR result MUST be preserved locally so the user can retry publication without re-capturing.

**Authentication**

- **FR-026**: Users MUST authenticate with Notion via OAuth before any read/write operations against their workspace.
- **FR-027**: The authentication session MUST persist across app restarts so users are not prompted to re-authenticate on every visit.

### Key Entities

- **Journal Entry**: A single captured page — contains raw OCR text, formatting annotations (highlights, strikethroughs, structural markers, doodle indicators), a reserved `symbols` array (empty in MVP, populated post-MVP by Editorial Marks Intelligence), capture timestamp, calibration feedback applied, and publication status.
- **Formatting Annotation**: A structured mark on an entry — type (highlight / strikethrough / doodle / structural), position reference (character range or page region), and any associated content.
- **Calibration Profile**: A per-user record of writing samples and correction history used to improve OCR accuracy; versioned across sessions.
- **Notion Page**: The published representation of a Journal Entry in the user's Notion workspace — content blocks, tag properties, relation links.
- **Tag**: An AI-extracted theme or named entity applied to a Journal Entry as a Notion page property.
- **Entry Relation**: A link between two Notion pages created based on topical similarity between their entries.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can complete the full capture-to-publish workflow (photo → OCR review → publish to Notion) in under 90 seconds on a mid-range mobile device.
- **SC-002**: OCR correctly transcribes at least 90% of words on a legible handwritten page for a user who has completed calibration onboarding.
- **SC-003**: Formatting annotations (highlights and strikethroughs) are correctly identified in at least 85% of cases where they are present on a legible page.
- **SC-004**: Auto-tagging produces at least 1 relevant theme or entity for 95% of entries containing 3 or more sentences.
- **SC-005**: Entry-to-entry relation linking correctly identifies at least 1 topically related existing entry in 80% of cases where a related entry exists (corpus of ≥ 5 entries).
- **SC-006**: At least 80% of users who begin the calibration onboarding flow complete it (completion rate).
- **SC-007**: Users who complete calibration and submit at least 10 post-capture corrections see measurable OCR accuracy improvement over their uncalibrated baseline.
- **SC-008**: Zero data loss — if the Notion publish step fails, 100% of confirmed OCR results are recoverable by the user without re-capturing.

---

## Assumptions

- Users have a Notion account and are willing to grant Inkwell write access to a designated journal database.
- Users are operating on a modern smartphone (iOS 16+ / Android 12+) with a functional rear camera.
- Lighting conditions at capture time are reasonably adequate — extreme low-light enhancement is deferred post-MVP.
- The Inkwell app manages Notion database schema setup on first publish (creates required properties if absent).
- Entry relation detection is based on semantic/topical similarity between the new entry's content and existing page content — the specific matching mechanism is a planning-phase decision.
- Cross-device calibration sync is desirable but not required for MVP; single device/browser scope is acceptable.
- The MVP is single-user only; multi-user and shared-notebook features are deferred.
- Users have sufficient internet connectivity for OCR, AI tagging, and Notion API calls; full offline-first capture queuing is deferred.
- Calibration data (handwriting samples and correction history) is stored on-device only — it never leaves the user's device in MVP. Cross-device sync is a v2 concern.
- Doodle image regions require cropping from the original capture photo — the app retains the full-resolution capture photo in temporary storage until the entry is published and all image blocks are uploaded to Notion.

## Deferred Features

### Editorial Marks Intelligence *(post-MVP)*

Inkwell should eventually recognize common handwritten editorial symbols and translate them into semantic actions in the Notion output. This is distinct from doodle detection (FR-019) — doodles are decorative non-text marks; editorial symbols are functional marks with specific meaning that should alter the structure or presentation of the published entry.

**Symbol vocabulary to support (priority order)**:

1. **Swap/transpose arrows** — words or phrases marked for reordering; output applies the transposition to the text in the Notion page
2. **Insertion carets** (`^` with text above/below) — insertion point markers; output splices the inserted text into the correct position
3. **Encircled text** — tag the enclosed passage as a callout or highlighted block in Notion
4. **Double underline** — strong emphasis, represented distinctly from single underline
5. **Margin asterisks/stars** — flag the adjacent passage as an action item or key insight (Notion callout or to-do block)
6. **Cross-out with replacement** — strikethrough plus arrow to correction; output applies the substitution rather than displaying both original and replacement

**Data model reservation (MVP deliverable)**:

The OCR output contract includes a `symbols` array (FR-028) reserved now to avoid a breaking change when this feature ships. The expected shape when populated:

```json
{
  "symbols": [
    {
      "type": "swap_arrow",
      "from": "word or phrase A",
      "to": "word or phrase B",
      "confidence": 0.91,
      "bounding_box": { "x": 120, "y": 340, "w": 80, "h": 30 }
    }
  ]
}
```

Supported `type` values (post-MVP): `swap_arrow`, `insertion_caret`, `encircle`, `double_underline`, `margin_star`, `crossout_replacement`.

**Calibration extension note**:

The on-device calibration system (FR-014, IndexedDB) should be extended in a future phase to learn user-specific symbol conventions alongside letterform corrections. A user who draws transposition arrows a particular way should be able to train Inkwell to recognize their personal editorial shorthand. This extends the existing calibration model scope — it does not require a separate system.

**Prerequisite for activation**: Editorial Marks Intelligence requires a meaningful corpus of entries (sufficient to evaluate symbol detection accuracy) and should not be enabled until the MVP capture pipeline is stable and in regular use.

---

## Clarifications

### Session 2026-04-10

- Q: Where is the calibration model persisted — on-device, server-side, or Notion? → A: On-device using browser IndexedDB. Cross-device sync deferred to v2.
- Q: How should doodles be represented in the Notion page? → A: Cropped image regions embedded as Notion image blocks. OCR output must include bounding coordinates for each doodle region to enable cropping before upload.
- Q: (Feature flag) Add Editorial Marks Intelligence as deferred post-MVP feature. → A: Added to Deferred Features section. `symbols` array reserved in OCR output contract as FR-028 (MVP deliverable, always empty array for MVP). Calibration extension to cover symbol conventions noted as future phase work.
