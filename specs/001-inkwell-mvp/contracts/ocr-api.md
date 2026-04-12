# Contract: OCR API Proxy

**Endpoint**: `POST /api/ocr`
**Consumer**: PWA frontend (`src/services/ocr.ts`)
**Provider**: Vercel Edge Function (`api/ocr.ts`) → Anthropic Messages API

---

## Request

```typescript
interface OCRRequest {
  imageBase64: string;             // Base64-encoded JPEG of the captured page
  mimeType: 'image/jpeg' | 'image/png';
  calibrationHints?: CalibrationHints;
}

interface CalibrationHints {
  // Stable profile generated once during onboarding analysis.
  // Describes character confusion patterns and formatting conventions for this user.
  // Absent for onboarding sample captures (profile doesn't exist yet).
  handwritingProfile?: {
    characterConfusions: Array<{ misread: string; correct: string; frequency: 'high' | 'medium' | 'low' }>;
    formattingStyle: {
      highlightDescription: string;
      strikethroughDescription: string;
      doodleCharacteristics: string;
    };
    styleNotes: string;
  };
  // Max 10 most recent corrections (text + formatting combined).
  recentCorrections: Array<{
    correctionType: 'text' | 'formatting';
    original?: string;
    corrected?: string;
    context?: string;
    formattingMismatch?: {
      detected: string | null;
      actual: string | null;
      passageText: string;
    };
  }>;
}
```

**Headers**:
- `Authorization: Bearer <notion_access_token>` — used to verify the user is authenticated (proxy validates token is present; does not call Notion API)
- `Content-Type: application/json`

**Constraints**:
- Max image size: 5MB base64-encoded (~3.75MB raw)
- Max corrections in `calibrationHints`: 10

---

## Response (success)

```typescript
// HTTP 200
interface OCRResponse {
  ocrOutput: OCROutput;
  modelUsed: string;               // e.g. "claude-opus-4-6"
  processingMs: number;            // server-side latency for diagnostics
}

interface OCROutput {
  text: string;
  formatting: FormattingAnnotation[];
  doodles: DoodleRegion[];
  symbols: EditorialSymbol[];      // always [] for MVP
}
```

See `data-model.md` for `FormattingAnnotation`, `DoodleRegion`, `EditorialSymbol` type definitions.

---

## Response (error)

```typescript
// HTTP 4xx / 5xx
interface OCRError {
  error: 'illegible_image'         // OCR returned no extractable text
       | 'image_too_large'
       | 'provider_error'          // upstream AI API failure
       | 'unauthorized';
  message: string;
  retryable: boolean;
}
```

| Status | Error code | Retryable |
|--------|-----------|-----------|
| 400 | `image_too_large` | No — user must recapture |
| 422 | `illegible_image` | No — user must recapture |
| 401 | `unauthorized` | No — re-authenticate |
| 502 | `provider_error` | Yes — transient AI API issue |
| 503 | `provider_error` | Yes |

---

## Claude Prompt Contract

The proxy constructs this prompt. The shape of the prompt is part of the contract — changing it changes the OCR output shape.

**System prompt** (stable, not per-request):
```
You are a handwriting OCR system. Extract the text and formatting semantics from the provided handwritten page image. You MUST return valid JSON matching the OCROutput schema exactly. Do not include any text outside the JSON object.
```

**User message structure**:
```
[image block: base64 JPEG]

Extract the handwritten text from this journal page and return a JSON object with this exact shape:

{
  "text": "<full transcribed text in linear reading order>",
  "formatting": [
    { "type": "highlight"|"strikethrough"|"paragraph_break"|"indent", "startOffset": <int>, "endOffset": <int>, "level": <int|null> }
  ],
  "doodles": [],
  "symbols": []
}

Rules:
- startOffset and endOffset are character positions in the "text" field (0-indexed, endOffset exclusive)
- Highlights: passages marked with highlighter or colored underline
- Strikethrough: text with a horizontal line through it
- Paragraph breaks: use type "paragraph_break" with startOffset = endOffset = position of break
- doodles array MUST always be an empty array [] — doodle detection is not active in this version
- symbols array MUST always be an empty array []
- If no formatting is detected, return an empty formatting array

<handwriting_profile>
[IF calibrationHints.handwritingProfile provided:]
Style: {styleNotes}
Character confusions (correct these automatically):
{characterConfusions mapped as "- '{misread}' → '{correct}' ({frequency} frequency)"}
Formatting conventions for this user:
- Highlights appear as: {highlightDescription}
- Strikethroughs appear as: {strikethroughDescription}
- Doodles appear as: {doodleCharacteristics}
</handwriting_profile>

<corrections>
[IF calibrationHints.recentCorrections provided: serialized corrections as XML here,
 including both text and formatting corrections]
</corrections>
```

---

## Handwriting Analysis Endpoint

**Endpoint**: `POST /api/analyze-handwriting`
**Called once**: At the end of calibration onboarding, after all samples are captured. Never called at regular capture time.
**Consumer**: `src/services/calibration.ts`
**Provider**: Vercel Edge Function (`api/analyze-handwriting.ts`) → Anthropic Messages API

```typescript
// Request
interface AnalyzeHandwritingRequest {
  samples: Array<{
    imageBase64: string;           // Base64-encoded JPEG of the sample capture
    mimeType: 'image/jpeg' | 'image/png';
    promptText: string;            // The exact text the user was asked to write
  }>;                              // 3–5 samples; matches CalibrationProfile.samples
}

// Response 200
interface AnalyzeHandwritingResponse {
  handwritingProfile: HandwritingProfile;
  seedCorrections: Array<{         // Auto-generated from OCR-vs-promptText diffs
    correctionType: 'text';
    original: string;
    corrected: string;
    context: string;
    source: 'onboarding_diff';
  }>;
  processingMs: number;
}

// Response 4xx/5xx
interface AnalyzeHandwritingError {
  error: 'insufficient_samples'    // fewer than 3 samples provided
       | 'provider_error'
       | 'unauthorized';
  message: string;
  retryable: boolean;
}
```

**Claude prompt contract**:

The analysis call sends all sample images in a single multi-image message. The prompt instructs Claude to:
1. Compare each image to its `promptText` and identify misread characters
2. Observe formatting mark style (how highlights/strikethroughs appear for this specific person)
3. Return a concise `HandwritingProfile` JSON **and** a flat array of text corrections derived from diffs

```
System: You are a handwriting analysis system. Analyze the provided handwriting samples and
return a JSON object describing this person's handwriting style and common OCR errors.
Be concise — the profile must fit in 600 tokens. Do not include any text outside the JSON.

User: [image 1] [image 2] ... [image N]

Each image shows handwriting of a specific prompt. Compare each image to its ground truth text
and identify: (1) characters or words frequently misread, (2) formatting mark style
(how highlights/strikethroughs/doodles appear for this person), (3) overall style notes.

Ground truth texts:
Sample 1: "<promptText>"
Sample 2: "<promptText>"
...

Return JSON with this exact shape:
{
  "handwritingProfile": {
    "characterConfusions": [{ "misread": "...", "correct": "...", "frequency": "high|medium|low" }],
    "formattingStyle": {
      "highlightDescription": "...",
      "strikethroughDescription": "...",
      "doodleCharacteristics": "..."
    },
    "styleNotes": "..."
  },
  "seedCorrections": [
    { "original": "...", "corrected": "...", "context": "..." }
  ]
}
```

**Constraints**:
- Max payload: 5 images × 200KB each + prompt overhead. Fits within Vercel Edge Function limits.
- `seedCorrections` MUST only include word-level diffs, not character-level. Max 50 seed corrections.
- If all samples were transcribed perfectly, `seedCorrections` is `[]`.

---

## Doodle Upload Endpoint *(deferred post-MVP)*

The `POST /api/upload-doodle` endpoint and Vercel Blob integration are not part of the MVP. They will be added when doodle detection and embedding ships. See Deferred Features in `spec.md`.
