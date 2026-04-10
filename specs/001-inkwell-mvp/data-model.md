# Data Model: Inkwell MVP

**Feature**: 001-inkwell-mvp
**Date**: 2026-04-10

All types are expressed in TypeScript. IndexedDB entities are stored via Dexie.js. Notion entities live exclusively in the user's Notion workspace — Inkwell never stores their content locally beyond the pre-publish pending state.

---

## On-Device Entities (IndexedDB via Dexie.js)

### `CalibrationProfile`

Represents the user's personalized handwriting model. One record per device. Created during onboarding; updated on each post-capture correction.

```typescript
interface CalibrationProfile {
  id: string;                       // UUID, generated on first install
  completedOnboarding: boolean;     // gates access to capture screen
  sampleCount: number;              // 0–5; ≥3 = onboarding complete
  samples: CalibrationSample[];     // embedded array
  corrections: CalibrationCorrection[];
  createdAt: number;                // Unix timestamp ms
  updatedAt: number;
  schemaVersion: number;            // Dexie migration version
}
```

**Indexes**: `id` (primary), `completedOnboarding`
**Constraints**: Only one record may exist (`id` is a fixed singleton key `'local'`)

---

### `CalibrationSample`

One of the 3–5 handwriting samples captured during onboarding. Stored embedded within `CalibrationProfile.samples`.

```typescript
interface CalibrationSample {
  id: string;                       // UUID
  promptId: string;                 // which of the provided writing prompts
  promptText: string;               // the text the user was asked to write
  imageDataUrl: string;             // compressed JPEG data URL (≤200KB target)
  capturedAt: number;
}
```

**Validation**: `promptId` must be one of the 5 predefined onboarding prompt IDs. Max 5 samples total.

---

### `CalibrationCorrection`

A single user correction from the post-capture review screen. Accumulated over time to provide few-shot OCR guidance.

```typescript
interface CalibrationCorrection {
  id: string;                       // UUID
  entryId: string;                  // which PendingEntry this came from
  original: string;                 // what the OCR model produced
  corrected: string;                // what the user typed
  surroundingContext: string;       // 20 chars before + after for disambiguation
  appliedAt: number;
}
```

**Constraints**: Max 200 corrections retained (FIFO eviction). Included as few-shot examples in OCR prompts.

---

### `PendingEntry`

Temporary storage for an entry that has been OCR-processed but not yet successfully published to Notion (FR-025 — zero data loss on API failure). Deleted on successful publish.

```typescript
interface PendingEntry {
  id: string;                       // UUID
  captureImageBlob: Blob;           // original full-resolution capture
  ocrOutput: OCROutput;             // structured result from vision model
  status: 'awaiting_review'         // user hasn't confirmed yet
         | 'confirmed'              // user confirmed, not yet published
         | 'publishing'             // publish in progress
         | 'published'              // successfully in Notion
         | 'failed';                // publish failed, user can retry
  notionPageId?: string;            // set after successful publish
  createdAt: number;
  updatedAt: number;
}
```

**Indexes**: `id` (primary), `status`, `createdAt`
**Retention**: Entries with `status: 'published'` are deleted after 24 hours (configurable post-MVP). `failed` entries are retained indefinitely until user retries or discards.

---

## In-Memory Types (not persisted to IndexedDB)

### `OCROutput`

The structured result returned by the Claude vision pipeline. Stored inside `PendingEntry.ocrOutput` after processing.

```typescript
interface OCROutput {
  text: string;                     // full transcribed text, linear reading order
  formatting: FormattingAnnotation[];
  doodles: DoodleRegion[];
  symbols: EditorialSymbol[];       // FR-028: always [] for MVP; reserved for Editorial Marks Intelligence
}
```

---

### `FormattingAnnotation`

A typed mark on the transcribed text. Uses character offsets into `OCROutput.text`.

```typescript
interface FormattingAnnotation {
  type: 'highlight'
      | 'strikethrough'
      | 'paragraph_break'
      | 'indent';
  startOffset: number;              // inclusive, index into OCROutput.text
  endOffset: number;                // exclusive
  level?: number;                   // indent depth (1–4); only for type 'indent'
}
```

**Validation**: `startOffset < endOffset`. Annotations of the same type MUST NOT overlap.

---

### `DoodleRegion`

A detected non-text mark region in the captured image. Used to crop and upload doodle images to Vercel Blob before Notion publish.

```typescript
interface DoodleRegion {
  id: string;                       // UUID, stable within OCROutput lifecycle
  boundingBox: BoundingBox;         // pixel coordinates in the original capture image
  pageZone: 'top' | 'bottom' | 'left-margin' | 'right-margin' | 'inline';
  uploadedUrl?: string;             // populated after Vercel Blob upload
  notionBlockId?: string;           // populated after Notion page creation
}

interface BoundingBox {
  x: number;                        // px from left edge of original image
  y: number;                        // px from top edge
  w: number;                        // width in px
  h: number;                        // height in px
}
```

---

### `EditorialSymbol`

Reserved for Editorial Marks Intelligence (post-MVP). Present in `OCROutput.symbols` but always an empty array in MVP.

```typescript
interface EditorialSymbol {
  type: 'swap_arrow'
      | 'insertion_caret'
      | 'encircle'
      | 'double_underline'
      | 'margin_star'
      | 'crossout_replacement';
  from?: string;                    // source text (swap_arrow, crossout_replacement)
  to?: string;                      // target text
  insertedText?: string;            // text to insert (insertion_caret)
  targetText?: string;              // text being acted on (encircle, double_underline, margin_star)
  confidence: number;               // 0.0–1.0
  boundingBox: BoundingBox;
}
```

---

## Notion Entities (live in user's Notion workspace)

Inkwell never stores these locally beyond the pre-publish `PendingEntry`. After publish, Inkwell reads Notion pages only for entry linking.

### Notion Database Schema

Inkwell creates (or verifies) a Notion database with these properties on first publish:

| Property Name | Notion Type | Purpose |
|---|---|---|
| `Title` | title | Entry title (auto-generated: date + first line) |
| `Captured At` | date | Capture timestamp |
| `Tags` | multi_select | AI-extracted themes and named entities |
| `Related Entries` | relation (self) | Links to topically related entries |
| `Has Doodles` | checkbox | True if any doodle regions were detected |
| `Word Count` | number | Approximate word count of transcribed text |
| `Calibration Version` | number | Schema version of CalibrationProfile at capture time |

### Notion Page Block Structure

```
Page
├── [paragraph] Entry text — first paragraph of transcribed text
├── [callout]   ⭐ Highlighted: "...highlighted passage..."   ← one per highlight annotation
├── [paragraph] Entry text — continued prose
├── [strikethrough paragraph] Struck-through text             ← inline rich text strikethrough
├── [image]     <doodle-crop-url>                             ← one per DoodleRegion
│   └── caption: "Doodle — [pageZone] margin"
├── [paragraph] Entry text — continued
└── ...
```

**Ordering**: Blocks are interleaved with text in reading order. Doodle image blocks are inserted at the paragraph position closest to the detected `pageZone`. Highlighted callouts immediately follow the paragraph containing the highlighted text.

---

## State Transitions

### PendingEntry Lifecycle

```
photo captured
      │
      ▼
awaiting_review ──[user corrects / confirms]──► confirmed
                                                    │
                                          [publish triggered]
                                                    │
                                                    ▼
                                              publishing ──[success]──► published ──[24h]──► (deleted)
                                                    │
                                               [failure]
                                                    │
                                                    ▼
                                                 failed ──[retry]──► publishing
                                                        ──[discard]──► (deleted)
```

### CalibrationProfile Completion

```
created (completedOnboarding: false, sampleCount: 0)
      │
      ├─ [sample captured] → sampleCount++
      │
      ├─ sampleCount ≥ 3: user may skip or continue
      │
      └─ sampleCount = 3–5 OR user skips → completedOnboarding: true
                                                  │
                                        [post-capture corrections]
                                                  │
                                          corrections[] grows (max 200, FIFO)
```
