# Quickstart & Integration Scenarios: Inkwell MVP

**Feature**: 001-inkwell-mvp
**Date**: 2026-04-10

Integration scenarios for the three core end-to-end flows. Each scenario describes the full sequence of service calls and state transitions. Use these as the basis for integration testing.

---

## Scenario 1: First-Time User — Calibration Onboarding → First Capture → Publish

### Preconditions
- App installed (or opened in browser for first time)
- No `NotionAuthToken` in IndexedDB
- No `CalibrationProfile` in IndexedDB

### Flow

```
1. App loads → auth guard → redirect to /onboarding/auth
2. User taps "Connect to Notion" → initiateNotionOAuth()
3. Notion OAuth consent → /api/auth/callback → token stored in IndexedDB
4. App loads → auth present, calibration incomplete → redirect to /onboarding/calibration
5. User photographs 3 writing samples (minimum)
   → For each sample: image + promptText stored in CalibrationProfile.samples
   → CalibrationProfile.sampleCount incremented to 3
   → (No OCR call during sample capture — images are stored raw for analysis)
6. User taps "Start journaling" (skip remaining 2 samples)
   → UI transitions to "Building your handwriting profile..." blocking state
   → POST /api/analyze-handwriting with all 3 sample images + their promptTexts
   → [success path]
      → Response: handwritingProfile + seedCorrections (e.g., 12 auto-generated diffs)
      → CalibrationProfile.handwritingProfile = response.handwritingProfile
      → response.seedCorrections appended to CalibrationProfile.corrections (source: 'onboarding_diff')
      → CalibrationProfile.completedOnboarding = true; written to IndexedDB
      → UI transitions to /capture
   → [failure path — auto-retry up to 3 times]
      → If all retries fail: show "Try again" + "Skip for now"
      → "Skip for now": CalibrationProfile.handwritingProfile remains undefined
        → CalibrationProfile.completedOnboarding = true (no profile)
        → UI transitions to /capture; OCR uses corrections-only hints until profile is generated
7. App redirects to /capture
8. User photographs journal page
   → PendingEntry created with status: 'ocr_pending' (before OCR call)
   → POST /api/ocr with image + calibrationHints (handwritingProfile + 0 user corrections, 12 seed corrections)
   → On success: PendingEntry.ocrOutput populated; status → 'awaiting_review'
   → On failure: PendingEntry.status → 'ocr_failed'; user shown retry option (no re-capture needed)
   → PendingEntry.ocrOutput populated; status: 'awaiting_review'
9. Review screen shown; user makes 2 corrections
   → Each correction: PendingEntry.ocrOutput.text updated; CalibrationCorrection appended to CalibrationProfile
10. User taps "Publish to Notion"
    → PendingEntry.status: 'publishing'
    → POST /api/tag with ocrOutput.text → tags[]
    → GET Notion database pages (last 50) for entry linking
    → POST /api/link-entries → relatedPageIds[]
    → POST https://api.notion.com/v1/pages (with token from IndexedDB)
       with all properties + blocks in single call
    → PendingEntry.notionPageId = response.id; status: 'published'
11. Success screen shown. PendingEntry scheduled for deletion after 24h.
```

### Observable Outcome
- Notion workspace contains one new page in "Inkwell Journal" database
- Page has: title, captured_at, tags (multi_select), word_count
- Page has: paragraph blocks for text, callout block(s) for highlights, strikethrough spans where applicable
- Related entries linked (or relations empty if no existing entries)
- CalibrationProfile.completedOnboarding = true, sampleCount = 3
- CalibrationProfile.handwritingProfile populated (character confusions, formatting style, style notes)
- CalibrationProfile.corrections contains seed corrections from onboarding diff + 2 user corrections

---

## Scenario 2: Returning User — Direct Capture → Review → Publish

### Preconditions
- `NotionAuthToken` present in IndexedDB (valid)
- `CalibrationProfile.completedOnboarding = true`, 45 corrections accumulated
- "Inkwell Journal" Notion database exists with 12 existing pages

### Flow

```
1. App loads → auth present, calibration complete → /capture (no redirects)
2. User photographs journal page
   → PendingEntry created (status: 'ocr_pending')
   → POST /api/ocr with image + calibrationHints (handwritingProfile + last 10 corrections included)
   → OCR succeeds: PendingEntry.ocrOutput populated; status → 'awaiting_review'
   → Response: OCROutput with text, 1 highlight, 0 strikethroughs, doodles: [], symbols: []
3. Review screen: user approves text as-is (no corrections)
   → User taps Publish
   → PendingEntry.status: 'publishing'
4. Parallel execution:
   → POST /api/tag
   → GET Notion pages (12 existing) → POST /api/link-entries → 2 related page IDs
5. Serial after both complete:
   → POST https://api.notion.com/v1/pages
6. Success. PendingEntry.notionPageId set, status: 'published'.
```

### Observable Outcome
- Notion page created with 1 callout block (highlight)
- 2 relation links created to existing entries
- No corrections → CalibrationProfile unchanged

---

## Scenario 3: Publish Failure Recovery (FR-025 — Zero Data Loss)

### Preconditions
- User has confirmed OCR review (PendingEntry.status = 'confirmed')
- Notion API is returning 503 at time of publish attempt

### Flow

```
1. User taps "Publish to Notion"
   → PendingEntry.status: 'publishing'
2. POST /api/tag succeeds (tags populated)
3. POST /api/link-entries succeeds (relatedPageIds populated)
4. POST https://api.notion.com/v1/pages → 503 error
   → PendingEntry.status: 'failed'
   → PendingEntry persisted to IndexedDB (ocrOutput preserved)
   → Error shown: "Couldn't reach Notion. Your entry is saved — tap Retry when ready."
5. [Later] Network restored. User returns to app.
   → App detects PendingEntry.status === 'failed' on load
   → Banner shown: "1 entry waiting to publish"
6. User taps Retry
   → PendingEntry.status: 'publishing'
   → Skip OCR (ocrOutput already present)
   → Skip tagging (tags already computed)
   → Skip linking (relatedPageIds already computed — note: these may be stale if new entries were added)
   → POST https://api.notion.com/v1/pages → success
   → PendingEntry.status: 'published'
```

### Observable Outcome
- Entry published to Notion on retry without requiring re-capture or re-review
- If `PendingEntry.notionPageId` is set from a previous partial attempt, PATCH is used instead of POST (no duplicate page)
- CalibrationCorrections from the original review session are preserved regardless of publish outcome

---

## Scenario 4: Calibration Progression — Accuracy Feedback Loop

### Preconditions
- User has completed onboarding (sampleCount = 3)
- CalibrationProfile.handwritingProfile is populated (e.g., notes that user's 'e' resembles 'c')
- User has made 9 user corrections + ~12 seed corrections from onboarding diff

### Flow

```
1. User captures a page with a word written in their distinctive style
2. POST /api/ocr with calibrationHints:
   → handwritingProfile included (styleNotes, characterConfusions, formattingStyle)
   → recentCorrections: last 10 of the 21 total corrections
   → Vision model uses the profile to pre-correct known confusion patterns
3. OCR still misreads one word (e.g., "fleeting" → "feeting")
4. User taps the word in review screen, types "fleeting"
   → CalibrationCorrection appended:
     { correctionType: 'text', original: "feeting", corrected: "fleeting",
       context: "...a fleeting...", source: 'user_correction' }
   → CalibrationProfile.corrections.length = 22
5. User also notices a highlight was missed (passage detected as plain text)
   → User taps the passage, marks it as highlight
   → CalibrationCorrection appended:
     { correctionType: 'formatting',
       formattingMismatch: { detected: null, actual: 'highlight', passageText: "...the passage..." },
       source: 'user_correction' }
   → CalibrationProfile.corrections.length = 23
6. User publishes entry
7. Next capture:
   → POST /api/ocr with calibrationHints containing handwritingProfile + last 10 corrections
   → The formatting correction is included; OCR is primed to look for this user's highlight style
```

### Observable Outcome
- Both text and formatting corrections are included in OCR calibration hints from the next capture
- Corrections array FIFO-evicts at 200 items (oldest removed when 201st is added)
- Corrections are never sent to Notion — they are local-only
- handwritingProfile is stable across sessions; only re-generated if user explicitly triggers re-calibration (post-MVP)
