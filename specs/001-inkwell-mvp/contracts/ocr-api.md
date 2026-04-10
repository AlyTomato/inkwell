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
  // Serialized subset of CalibrationProfile for few-shot OCR guidance.
  // Max 10 most recent corrections to keep prompt size manageable.
  recentCorrections: Array<{
    original: string;
    corrected: string;
    context: string;
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
  "doodles": [
    { "id": "<uuid>", "boundingBox": { "x": <int>, "y": <int>, "w": <int>, "h": <int> }, "pageZone": "top"|"bottom"|"left-margin"|"right-margin"|"inline" }
  ],
  "symbols": []
}

Rules:
- startOffset and endOffset are character positions in the "text" field (0-indexed, endOffset exclusive)
- Highlights: passages marked with highlighter or colored underline
- Strikethrough: text with a horizontal line through it
- Paragraph breaks: use type "paragraph_break" with startOffset = endOffset = position of break
- Doodles: any marks that are drawings, sketches, or symbols not part of readable text
- boundingBox coordinates are in pixels relative to the top-left of the image
- symbols array MUST always be an empty array []
- If no formatting/doodles are detected, return empty arrays

<corrections>
[IF calibrationHints provided: serialized corrections as XML here]
</corrections>
```

---

## Doodle Upload Endpoint

**Endpoint**: `POST /api/upload-doodle`

```typescript
// Request: multipart/form-data
// Field: "doodle" — Blob (cropped JPEG region)
// Field: "doodleId" — string (DoodleRegion.id from OCROutput)

// Response 200:
interface DoodleUploadResponse {
  doodleId: string;
  url: string;                     // stable Vercel Blob public URL
}

// Response 4xx/5xx:
interface DoodleUploadError {
  error: 'upload_failed' | 'too_large';
  doodleId: string;
  retryable: boolean;
}
```

**Constraints**:
- Max doodle crop size: 1MB
- Blob storage path: `doodles/{userId}/{entryId}/{doodleId}.jpg`
- Access: `public`, no expiry
