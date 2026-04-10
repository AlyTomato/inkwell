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
   → For each sample: POST /api/ocr with image + no calibrationHints
   → Each OCROutput stored in CalibrationProfile.samples
   → CalibrationProfile.sampleCount incremented to 3
6. User taps "Start journaling" (skip remaining 2 samples)
   → CalibrationProfile.completedOnboarding = true; written to IndexedDB
7. App redirects to /capture
8. User photographs journal page
   → PendingEntry created with status: 'awaiting_review'
   → POST /api/ocr with image + calibrationHints (0 corrections so far)
   → PendingEntry.ocrOutput populated; status: 'awaiting_review'
9. Review screen shown; user makes 2 corrections
   → Each correction: PendingEntry.ocrOutput.text updated; CalibrationCorrection appended to CalibrationProfile
10. User taps "Publish to Notion"
    → PendingEntry.status: 'publishing'
    → For each DoodleRegion in ocrOutput.doodles:
        a. Crop from PendingEntry.captureImageBlob using Canvas API
        b. POST /api/upload-doodle → DoodleRegion.uploadedUrl populated
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
- Page has: title, captured_at, tags (multi_select), has_doodles (checkbox), word_count
- Page has: paragraph blocks for text, callout block(s) for highlights, image block(s) for doodles
- Related entries linked (or relations empty if no existing entries)
- CalibrationProfile.completedOnboarding = true, sampleCount = 3, corrections.length = 2

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
   → PendingEntry created (status: 'awaiting_review')
   → POST /api/ocr with image + calibrationHints (last 10 corrections included)
   → Response: OCROutput with text, 1 highlight, 0 strikethroughs, 1 doodle, symbols: []
3. Review screen: user approves text as-is (no corrections)
   → User taps Publish
   → PendingEntry.status: 'publishing'
4. Parallel execution:
   → Crop doodle → POST /api/upload-doodle
   → POST /api/tag
   (both can run simultaneously before Notion page creation)
5. Serial after both complete:
   → GET Notion pages (12 existing) → POST /api/link-entries → 2 related page IDs
   → POST https://api.notion.com/v1/pages
6. Success. PendingEntry.notionPageId set, status: 'published'.
```

### Observable Outcome
- Notion page created with 1 callout block (highlight), 1 image block (doodle crop)
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
2. Doodle upload succeeds (DoodleRegion.uploadedUrl populated)
3. POST /api/tag succeeds (tags populated)
4. POST /api/link-entries succeeds (relatedPageIds populated)
5. POST https://api.notion.com/v1/pages → 503 error
   → PendingEntry.status: 'failed'
   → PendingEntry persisted to IndexedDB (captureImageBlob + ocrOutput preserved)
   → Error shown: "Couldn't reach Notion. Your entry is saved — tap Retry when ready."
6. [Later] Network restored. User returns to app.
   → App detects PendingEntry.status === 'failed' on load
   → Banner shown: "1 entry waiting to publish"
7. User taps Retry
   → PendingEntry.status: 'publishing'
   → Skip doodle upload (uploadedUrl already populated)
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
- User has made 9 corrections across previous sessions

### Flow

```
1. User captures a page with a word written in their distinctive style
2. POST /api/ocr → OCR misreads the word (e.g., "fleeting" → "feeting")
3. User taps the word in review screen, types "fleeting"
   → CalibrationCorrection appended: { original: "feeting", corrected: "fleeting", context: "...a fleeting..." }
   → CalibrationProfile.corrections.length = 10
4. User publishes entry
5. Next capture:
   → POST /api/ocr with calibrationHints containing the 10 corrections
   → The correction "feeting" → "fleeting" is included as a few-shot example
   → Vision model applies the hint; "fleeting" is now correctly recognized
```

### Observable Outcome
- Corrections are included in the OCR prompt from the 2nd+ capture after being added
- Corrections array FIFO-evicts at 200 items (oldest removed when 201st is added)
- Corrections are never sent to Notion — they are local-only
