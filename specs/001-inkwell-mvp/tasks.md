# Tasks: Inkwell MVP

**Input**: Design documents from `specs/001-inkwell-mvp/`
**Prerequisites**: plan.md ✅ spec.md ✅ data-model.md ✅ contracts/ ✅ research.md ✅ quickstart.md ✅
**Generated**: 2026-04-13

**Tests**: Not requested in spec — test tasks omitted. E2E scaffolding included in Polish phase.

**Organization**: Tasks grouped by user story to enable independent implementation and testing.

## Format: `[ID] [P?] [Story?] Description with file path`

- **[P]**: Can run in parallel (different files, no incomplete-task dependencies)
- **[Story]**: Which user story this task belongs to ([US1], [US2], [US3])
- No story label on Setup, Foundational, or Polish phase tasks

---

## Phase 1: Setup

**Purpose**: Scaffold the project, install dependencies, configure tooling. No application logic.

- [ ] T001 Scaffold Vite 6 + React 19 + TypeScript 5 project at repo root: run `npm create vite@latest . -- --template react-ts`; delete the generated `src/App.css` and `src/assets/` boilerplate
- [ ] T002 Install all runtime dependencies: `npm install react-router-dom@7 dexie@4 @notionhq/client`; install devDependencies: `vitest @vitest/ui @testing-library/react @testing-library/user-event jsdom playwright @playwright/test`; install `tailwindcss@4 vite-plugin-pwa`
- [ ] T003 [P] Configure TypeScript strict mode in `tsconfig.json` and `tsconfig.app.json`: set `"strict": true`, `"target": "ES2022"`, `"moduleResolution": "bundler"`, `"baseUrl": "."`, `"paths": { "@/*": ["src/*"] }`
- [ ] T004 [P] Configure Tailwind CSS v4 in `src/index.css`: replace file contents with `@import "tailwindcss"`; confirm `src/main.tsx` imports `src/index.css`
- [ ] T005 [P] Configure `vite-plugin-pwa` in `vite.config.ts`: add `VitePWA({ registerType: 'autoUpdate', manifest: false, workbox: { globPatterns: ['**/*.{js,css,html,ico,png,svg}'] } })` to plugins; set `server.port: 3000`; add `resolve.alias` mapping `@/` → `src/`
- [ ] T006 [P] Create `public/manifest.json`: `name: "Inkwell"`, `short_name: "Inkwell"`, `description: "Where pen meets patterns"`, `display: "standalone"`, `start_url: "/"`, `background_color: "#ffffff"`, `theme_color: "#1a1a1a"`, `icons` pointing to `public/icons/icon-192.png` and `public/icons/icon-512.png` (placeholder PNGs added in Polish phase T043)
- [ ] T007 [P] Create `vercel.json`: set `framework: "vite"`; add `functions: { "api/**/*.ts": { runtime: "edge" } }` to declare all `api/` files as Vercel Edge Functions
- [ ] T008 [P] Create `.env.local.example` documenting every required environment variable: `ANTHROPIC_API_KEY`, `NOTION_CLIENT_SECRET`, `NOTION_CLIENT_ID`, `VITE_NOTION_CLIENT_ID` (same value as `NOTION_CLIENT_ID`; Vite bundles it into the frontend — intentional, OAuth client IDs are public), `BLOB_READ_WRITE_TOKEN`; add a comment per variable explaining its source (Anthropic console, Notion integration page, Vercel dashboard)

---

## Phase 2: Foundational

**Purpose**: Shared types, IndexedDB schema, auth flow, and routing shell. ALL of this must be complete before any user story begins — the route guard (T016) references `/onboarding/auth`, which requires auth components (T014–T015) to exist.

**⚠️ CRITICAL**: No user story work can begin until this entire phase is complete.

- [ ] T009 Define all shared TypeScript types in `src/types/index.ts`: export `OCROutput` (text, formatting, doodles, symbols), `FormattingAnnotation` (type: `'highlight' | 'strikethrough' | 'paragraph_break' | 'indent'`, startOffset, endOffset, level?), `DoodleRegion` (reserved — defined but never instantiated in MVP), `EditorialSymbol` (reserved — always `[]`), `BoundingBox`, `CalibrationProfile`, `CalibrationSample`, `HandwritingProfile` (characterConfusions, formattingStyle, styleNotes, generatedFromSampleCount, generatedAt), `CalibrationCorrection` (correctionType: `'text' | 'formatting'`, source: `'user_correction' | 'onboarding_diff'`), `PendingEntry` (status union: `'ocr_pending' | 'ocr_failed' | 'awaiting_review' | 'confirmed' | 'publishing' | 'published' | 'failed'`, `ocrOutput?: OCROutput`), `NotionAuthToken` (accessToken, workspaceId, workspaceName, journalDatabaseId?, storedAt), `CalibrationHints`, `OCRRequest`, `OCRResponse`, `OCRError` — all shapes must match `data-model.md` and `contracts/ocr-api.md` exactly
- [ ] T010 Initialise Dexie v4 schema in `src/store/db.ts`: create `class InkwellDB extends Dexie` with `version(1).stores({ calibrationProfiles: 'id, completedOnboarding', pendingEntries: 'id, status, createdAt', auth: 'id' })`; export singleton `const db = new InkwellDB()`; `CalibrationProfile.id` is always the singleton key `'local'`
- [ ] T011 [P] Implement `NotionAuthToken` CRUD in `src/store/auth.ts`: export `getToken(): Promise<NotionAuthToken | undefined>`, `setToken(token: NotionAuthToken): Promise<void>`, `clearToken(): Promise<void>`, `setJournalDatabaseId(id: string): Promise<void>` — all operate on the single record in `db.auth` with key `'local'`
- [ ] T012 [P] Implement `PendingEntry` CRUD + status transitions in `src/store/pending.ts`: export `createEntry(): Promise<PendingEntry>` (writes immediately with `status: 'ocr_pending'`, new UUID, timestamps — no image stored per data-model.md Note), `getEntry(id)`, `updateStatus(id, status)`, `setOcrOutput(id, ocrOutput)`, `setNotionPageId(id, notionPageId)`, `getFailedEntries(): Promise<PendingEntry[]>` (returns entries with status `'failed'` or `'ocr_failed'`), `deleteEntry(id)`, `cleanupPublishedEntries()` (deletes `'published'` entries older than 24 h); status transitions must match the lifecycle diagram in `data-model.md`
- [ ] T013 [P] Implement `CalibrationProfile` CRUD + sample/correction operations in `src/store/calibration.ts`: export `getProfile()`, `initProfile()` (creates singleton with `id: 'local'`, `completedOnboarding: false`, `sampleCount: 0`, `corrections: []`, `schemaVersion: 1`), `addSample(sample: CalibrationSample)` (increments `sampleCount`), `setHandwritingProfile(profile: HandwritingProfile)`, `addCorrection(correction: CalibrationCorrection)` (enforces FIFO eviction at 200 — splice oldest when adding 201st), `markOnboardingComplete()` (sets `completedOnboarding: true`)
- [ ] T014 Implement Notion OAuth flow in `src/lib/auth.ts`: export `initiateNotionOAuth(): void` (generates UUID state via `crypto.randomUUID()`, stores in `sessionStorage('oauth_state')`, redirects to `https://api.notion.com/v1/oauth/authorize` with `client_id: import.meta.env.VITE_NOTION_CLIENT_ID`, `redirect_uri`, `response_type: 'code'`, `state`, `owner: 'user'`); export `parseTokenFromHash(): NotionAuthToken | null` (parse `location.hash` for token/workspace/state params, compare state against `sessionStorage`, on mismatch return null, on match clear sessionStorage + clear hash via `history.replaceState` + return token object); export `isAuthenticated(): Promise<boolean>` (checks IndexedDB via `getToken()`)
- [ ] T015 Implement `GET /api/auth/callback` Vercel Edge Function in `api/auth/callback.ts`: validate `code` and `state` query params (return 400 if missing); POST to `https://api.notion.com/v1/oauth/token` with `client_id`, `client_secret` (env: `NOTION_CLIENT_SECRET`), `grant_type: 'authorization_code'`, `code`, `redirect_uri`; on success redirect to `{origin}/#token=<access_token>&workspace=<workspace_id>&workspace_name=<encoded_name>&state=<state>` (hash never reaches server); on Notion API error redirect to `{origin}/onboarding/auth?error=oauth_failed`; export as Edge Function (`export const config = { runtime: 'edge' }`)
- [ ] T016 [P] Build `NotionConnect` component + `/onboarding/auth` page: create `src/components/auth/NotionConnect.tsx` with a "Connect to Notion" button that calls `initiateNotionOAuth()`; on mount, check `location.hash` — if token params present call `parseTokenFromHash()`, on valid token call `setToken()` then navigate to `/onboarding/calibration`; on null (CSRF fail) or `?error=oauth_failed` show error state with retry button; create `src/pages/onboarding/auth.tsx` that renders `<NotionConnect />`
- [ ] T017 [P] Create `ErrorBoundary` component in `src/components/common/ErrorBoundary.tsx`: React class component, catches render errors, renders fallback with error message and "Reload app" button that calls `window.location.reload()`; export as default
- [ ] T018 Set up React Router v7 app shell in `src/main.tsx` + `src/App.tsx`: define routes — `/` (root guard), `/onboarding/auth`, `/onboarding/calibration`, `/capture`, `/review`; root guard reads `NotionAuthToken` and `CalibrationProfile.completedOnboarding` from IndexedDB and redirects: no token → `/onboarding/auth`; token present but `!completedOnboarding` → `/onboarding/calibration`; both satisfied → `/capture`; wrap router in `<ErrorBoundary>`; create `index.html` with `<div id="root">` if not already present from scaffold

**Checkpoint**: `vercel dev` serves the app. Visiting `/` redirects to `/onboarding/auth`. The NotionConnect page renders. OAuth initiation redirects to Notion's consent screen. Callback handler exchanges the code and redirects back with the token in the hash.

---

## Phase 3: User Story 1 — Capture & Formatting-Aware OCR (Priority: P1) 🎯 MVP

**Goal**: User photographs a handwritten page and receives a structured `OCROutput` with text + formatting annotations (highlights, strikethroughs, paragraph breaks). User reviews the result, makes inline corrections that are saved as calibration signals.

**Independent Test**: Provide a handwritten journal photo → verify `OCROutput` contains (a) transcribed `text`, (b) `formatting[]` with highlight and strikethrough annotations at correct character offsets, (c) `doodles: []`, (d) `symbols: []`. Make an inline correction → verify `CalibrationCorrection` appended to `CalibrationProfile.corrections`. No Notion integration required.

- [ ] T019 [P] [US1] Implement camera abstraction in `src/lib/camera.ts`: export `getCaptureMethod(): 'getUserMedia' | 'inputCapture'` (feature-detects `ImageCapture` support at runtime — if `typeof ImageCapture !== 'undefined'` → `getUserMedia`, else → `inputCapture`); export `captureViaUserMedia(): Promise<Blob>` (requests `{ video: { facingMode: 'environment' } }` stream, grabs frame to offscreen canvas, returns JPEG blob, stops stream tracks after capture); export `createInputCapture(): HTMLInputElement` (returns `<input type="file" accept="image/*" capture="environment">` for iOS Safari); both paths produce a `Blob` compatible with `compressImage()`
- [ ] T020 [P] [US1] Implement canvas-based image compression in `src/lib/image.ts`: export `compressImage(blob: Blob, maxBytes?: number): Promise<{ base64: string; mimeType: 'image/jpeg' }>` — draw image to offscreen canvas, scale down iteratively while `base64.length > maxBytes * 1.37` (base64 overhead factor), export as JPEG at quality 0.85; default `maxBytes` is 3_750_000 (~5 MB base64); throw `new Error('image_too_large')` if still over limit after scaling to 20% of original
- [ ] T021 [P] [US1] Implement `POST /api/ocr` Vercel Edge Function in `api/ocr.ts`: parse `OCRRequest` body (imageBase64, mimeType, calibrationHints?); validate `Authorization` header present (return 401 `{ error: 'unauthorized' }`); validate `imageBase64.length ≤ 5_000_000` (return 400 `{ error: 'image_too_large', retryable: false }`); construct system prompt and user message per `contracts/ocr-api.md` Claude prompt contract — inject `<handwriting_profile>` block if `calibrationHints.handwritingProfile` present, inject `<corrections>` block if `calibrationHints.recentCorrections` non-empty; call Anthropic Messages API with model `claude-opus-4-6`, image block + text; parse JSON response as `OCROutput`; if `ocrOutput.text` is empty return 422 `{ error: 'illegible_image', retryable: false }`; on Anthropic API error return 502 `{ error: 'provider_error', retryable: true }`; return `{ ocrOutput, modelUsed: 'claude-opus-4-6', processingMs }` on success; `export const config = { runtime: 'edge' }`
- [ ] T022 [US1] Implement OCR service in `src/services/ocr.ts`: export `runOcr(blob: Blob, pendingEntryId: string): Promise<OCROutput>` — call `compressImage(blob)`, call `getProfile()` to read calibration hints (build `CalibrationHints` with `handwritingProfile` if present + last 10 corrections by `appliedAt` desc), read auth token for Bearer header, POST to `/api/ocr`; on success call `setOcrOutput(pendingEntryId, ocrOutput)` + `updateStatus(pendingEntryId, 'awaiting_review')` + return `ocrOutput`; on failure call `updateStatus(pendingEntryId, 'ocr_failed')` + throw typed `OCRError`
- [ ] T023 [US1] Build `CameraCapture` component in `src/components/capture/CameraCapture.tsx`: on mount call `getCaptureMethod()`; if `'getUserMedia'` render live `<video>` preview (stream from `navigator.mediaDevices.getUserMedia`) with circular shutter button; if `'inputCapture'` render a styled full-screen tap area that programmatically clicks the hidden `<input capture>` element on tap; on image captured: (1) call `createEntry()` — writes `PendingEntry` with status `'ocr_pending'` BEFORE any API call (zero data loss guarantee); (2) call `runOcr(blob, entry.id)`; (3) on success navigate to `/review?id={entry.id}`; (4) on `ocr_failed` show retry button and call `runOcr` again with the same blob held in component state (no re-capture needed); on `image_too_large` show "Photo too large — try moving closer or in better lighting"; on camera permission denied show "Camera access required — check browser settings"
- [ ] T024 [US1] Build `OcrReviewScreen` component in `src/components/capture/OcrReviewScreen.tsx`: accept `entry: PendingEntry` as prop; render `ocrOutput.text` as a content-editable-like view with formatting visually applied (highlight spans use `bg-yellow-200`, strikethrough spans use `line-through`); **text correction**: tap any word → inline `<input>` replaces it → on blur/Enter: update `ocrOutput.text` in the entry via `setOcrOutput()`, append `CalibrationCorrection` (`correctionType: 'text'`, original, corrected, surroundingContext: 20 chars each side, source: `'user_correction'`) via `addCorrection()`; **formatting correction**: tap annotation label → popover with type options → on change: re-emit updated `ocrOutput` + append `CalibrationCorrection` (`correctionType: 'formatting'`) via `addCorrection()`; "Confirm & Publish" button calls `updateStatus(entry.id, 'confirmed')` and invokes `onConfirmed` prop callback
- [ ] T025 [US1] Build capture page in `src/pages/capture.tsx`: auth-guard (redirect to `/onboarding/auth` if no token); render `<CameraCapture />`; on mount call `getFailedEntries()` — if any exist render `<PendingEntryBanner />` (component created in T042; import it here, it won't throw if the file is a stub)
- [ ] T026 [US1] Build review page skeleton in `src/pages/review.tsx`: read `id` from `location.search`; load `PendingEntry` from IndexedDB; if missing or status not in `['awaiting_review', 'confirmed']` redirect to `/capture`; render `<OcrReviewScreen entry={entry} onConfirmed={() => { /* publish wired in T038 (US3) */ setShowPublishPlaceholder(true) }} />`; show "Publish to Notion" placeholder state after confirm (full publish flow wired in Phase 5)

**Checkpoint**: User Story 1 is independently functional. Can photograph a handwritten page, receive structured OCR output, make inline corrections, and verify `CalibrationProfile.corrections` grows in IndexedDB. No Notion publish required.

---

## Phase 4: User Story 2 — Handwriting Calibration Onboarding (Priority: P2)

**Goal**: First-time users complete a 3–5 sample onboarding flow. The app runs a blocking handwriting analysis that produces `HandwritingProfile` + seed corrections. All subsequent OCR calls include these calibration hints. Returning users bypass onboarding.

**Independent Test**: Complete onboarding flow with 3 samples → verify `CalibrationProfile.completedOnboarding === true`, `handwritingProfile` populated with `characterConfusions`, `formattingStyle`, `styleNotes`, and `corrections[]` contains ≥ 1 entry with `source: 'onboarding_diff'`. Then trigger an OCR call and inspect the `/api/ocr` request body — confirm `calibrationHints.handwritingProfile` is present.

- [ ] T027 [P] [US2] Implement `POST /api/analyze-handwriting` Vercel Edge Function in `api/analyze-handwriting.ts`: parse `AnalyzeHandwritingRequest` (samples array with imageBase64, mimeType, promptText per sample); validate `Authorization` header (401); validate 3–5 samples (return 400 `{ error: 'insufficient_samples' }` if fewer than 3); construct multi-image Anthropic Messages API call per `contracts/ocr-api.md` analysis prompt — all sample images in a single message, ground truth texts listed, instruct Claude to return `{ handwritingProfile, seedCorrections }` JSON concisely within 600 tokens; parse response; validate `seedCorrections.length ≤ 50`; return `AnalyzeHandwritingResponse`; on Anthropic error return 502 `{ error: 'provider_error', retryable: true }`; `export const config = { runtime: 'edge' }`
- [ ] T028 [US2] Implement calibration analysis service in `src/services/calibration.ts`: export `buildCalibrationHints(profile: CalibrationProfile): CalibrationHints` — returns `{ handwritingProfile: { characterConfusions, formattingStyle, styleNotes } }` (strips `generatedAt` and `generatedFromSampleCount` — wire shape per `contracts/ocr-api.md` differs from stored `HandwritingProfile`) + `recentCorrections`: last 10 corrections by `appliedAt` desc, mapped to wire shape; export `runOnboardingAnalysis(samples: CalibrationSample[]): Promise<void>` — reads auth token, POSTs to `/api/analyze-handwriting`; implements exponential backoff retry: up to 3 attempts with delays 2 s / 4 s / 8 s (use `setTimeout` wrapped in a Promise); on success: call `setHandwritingProfile()` + append each `seedCorrection` via `addCorrection()` with `source: 'onboarding_diff'` + call `markOnboardingComplete()`; on all retries exhausted: throw `{ retryable: true, message: 'Analysis failed after 3 attempts' }` so caller can offer Try Again / Skip; export `skipOnboardingAnalysis(): Promise<void>` — calls `markOnboardingComplete()` without setting `handwritingProfile`
- [ ] T029 [US2] Update `src/services/ocr.ts` `runOcr()` to inject calibration hints: after loading profile via `getProfile()`, call `buildCalibrationHints(profile)` and include the result as `calibrationHints` in the OCR request body (omit the field entirely — not `{}`  — when profile is absent or `corrections` is empty, to match `OCRRequest.calibrationHints?` contract)
- [ ] T030 [P] [US2] Build `SampleCapture` component in `src/components/calibration/SampleCapture.tsx`: accept props `promptId: string`, `promptText: string`, `sampleIndex: number`, `totalSamples: number`, `onCaptured: (sample: CalibrationSample) => void`; display the prompt text prominently for the user to handwrite; use `getCaptureMethod()` to render `<video>` shutter or `<input capture>`; on capture call `compressImage(blob, 200_000)` (200 KB target — preserve quality for analysis); call `onCaptured({ id: crypto.randomUUID(), promptId, promptText, imageDataUrl: base64, capturedAt: Date.now() })`; show captured image preview with "Retake" button
- [ ] T031 [US2] Build `OnboardingFlow` component in `src/components/calibration/OnboardingFlow.tsx`: define 5 hardcoded prompt objects `{ id, text }` (short varied sentences, e.g., "The quick brown fox.", "Today I feel grateful for...", "My goal this week is", "I notice that I often", "One thing I want to remember:"); manage `currentStep` (0–4) and `capturedSamples: CalibrationSample[]` state; render `<SampleCapture>` for the current step; after each capture: call `addSample()` to persist + advance step; once `capturedSamples.length >= 3` show both "Continue to sample {n+1}" and "Start journaling" (skip) buttons; on "Start journaling" or completing all 5: show full-screen blocking `"Building your handwriting profile…"` state with spinner; call `runOnboardingAnalysis(capturedSamples)`; on success navigate to `/capture`; on throw (all retries failed): show "Analysis failed" with "Try again" (re-calls `runOnboardingAnalysis`) and "Skip for now" (calls `skipOnboardingAnalysis()` then navigates to `/capture`)
- [ ] T032 [US2] Build calibration onboarding page in `src/pages/onboarding/calibration.tsx`: on mount: if `CalibrationProfile.completedOnboarding === true` redirect to `/capture` (returning user guard); if no profile exists call `initProfile()`; render `<OnboardingFlow />`

**Checkpoint**: First-time user sees onboarding flow, photographs 3 samples, analysis runs (with retry on failure), `completedOnboarding: true` in IndexedDB. Subsequent OCR request bodies include `calibrationHints.handwritingProfile`.

---

## Phase 5: User Story 3 — Notion Publishing with Auto-Tagging & Entry Relations (Priority: P3)

**Goal**: After confirming OCR review, user publishes the entry to Notion. A page is created with formatting-preserving blocks (callout for highlights, inline strikethrough), AI-extracted tags as multi-select properties, and relation links to topically related existing entries — all atomically in one API call. Publish failures are surfaced and recoverable without re-capture.

**Independent Test**: Provide a `PendingEntry` with `status: 'confirmed'`, `ocrOutput` containing 1 highlight and 1 strikethrough → verify (a) Notion page created in "Inkwell Journal" database, (b) callout block present for highlight, (c) paragraph block with `annotations: { strikethrough: true }` present, (d) `Tags` property populated with ≥ 1 value, (e) `Related Entries` set if ≥ 1 existing page exists. Force a 503 → verify `PendingEntry.status === 'failed'` and entry is recoverable.

- [ ] T033 [P] [US3] Implement `POST /api/tag` Vercel Edge Function in `api/tag.ts`: validate `Authorization` header (401); truncate request `text` to first 2000 chars; call Anthropic `claude-opus-4-6` with tagging prompt from `contracts/notion-schema.md`; parse response as `string[]` (3–8 tags, Title Case, ≤ 30 chars each); return `TagResponse { tags, processingMs }`; on parse failure return `{ tags: [], processingMs }` (graceful degradation — tagging failure must not block publish); `export const config = { runtime: 'edge' }`
- [ ] T034 [P] [US3] Implement `POST /api/link-entries` Vercel Edge Function in `api/link-entries.ts`: validate `Authorization` header (401); serialize `existingEntries` as compact list per linking prompt in `contracts/notion-schema.md`; call Claude with `claude-opus-4-6`; parse response as `string[]` of Notion page IDs (cap at 5); return `LinkEntriesResponse { relatedPageIds, processingMs }`; on any error return `{ relatedPageIds: [], processingMs }` (graceful — linking failure must not block publish); `export const config = { runtime: 'edge' }`
- [ ] T035 [P] [US3] Implement tagging service in `src/services/tagging.ts`: export `extractTags(text: string, accessToken: string): Promise<string[]>` — POST `{ text }` to `/api/tag` with Bearer token; return `tags` array; on error return `[]`
- [ ] T036 [P] [US3] Implement entry linking service in `src/services/linking.ts`: export `findRelatedEntries(newEntryText: string, newEntryTags: string[], accessToken: string, databaseId: string): Promise<string[]>` — fetch last 50 pages from `https://api.notion.com/v1/databases/{databaseId}/query` (sort `Captured At` desc, `page_size: 50`); map results to `{ notionPageId, title, tags, capturedAt }`; POST to `/api/link-entries`; return `relatedPageIds`; on any error return `[]`
- [ ] T037 [US3] Implement Notion service in `src/services/notion.ts`: export `ensureDatabase(accessToken: string): Promise<string>` — read `NotionAuthToken.journalDatabaseId` from IndexedDB; if present, call `GET https://api.notion.com/v1/databases/{id}` to verify; on 404 or absent, create "Inkwell Journal" database at workspace root via `POST /v1/databases` with all properties from `contracts/notion-schema.md` (`Title`, `Captured At` date, `Tags` multi_select, `Related Entries` self-referential relation, `Word Count` number, `Calibration Version` number); call `setJournalDatabaseId()` with new ID; return database ID; export `assembleBlocks(ocrOutput: OCROutput): object[]` — implement block assembly algorithm from `contracts/notion-schema.md`: (1) split `ocrOutput.text` into segments at `paragraph_break` positions; (2) for each segment collect overlapping highlight and strikethrough annotations; (3) for each highlight: emit callout block (`type: 'callout'`, `icon: { emoji: '🖊️' }`, `color: 'yellow_background'`) with the highlighted text as rich_text; if a strikethrough overlaps the same range apply `annotations: { strikethrough: true }` within the callout rich_text span — do NOT emit a separate strikethrough paragraph for that range; (4) emit remaining text as paragraph blocks with inline strikethrough annotations where applicable; (5) return ordered block array; export `generateTitle(ocrOutput: OCROutput, capturedAt: number): string` — format `{YYYY-MM-DD} — {first 50 chars of ocrOutput.text}` using `new Date(capturedAt).toISOString().slice(0, 10)`
- [ ] T038 [US3] Wire full publish flow into review page in `src/pages/review.tsx`: replace the "Publishing coming soon" placeholder from T026 with real orchestration — on "Confirm & Publish": (1) `updateStatus(id, 'confirmed')` then `updateStatus(id, 'publishing')`; (2) read auth token; (3) run `extractTags(text, token)` and `findRelatedEntries(text, tags, token, dbId)` in parallel via `Promise.all()`; (4) call `ensureDatabase(token)`; (5) call `assembleBlocks(ocrOutput)` + `generateTitle()`; (6) `POST https://api.notion.com/v1/pages` with all properties + block children in single call per atomicity contract in `contracts/notion-schema.md`; (7) on success: call `setNotionPageId(id, pageId)` + `updateStatus(id, 'published')` + show "Published to Notion ✓" success state; (8) on Notion 401: call `clearToken()` + navigate to `/onboarding/auth`; (9) on any other failure: `updateStatus(id, 'failed')` + show error "Couldn't reach Notion. Your entry is saved — tap Retry when ready." with retry button; retry path re-uses existing `ocrOutput.text` for tags and skips re-linking (stale relation links acceptable per quickstart Scenario 3); if `PendingEntry.notionPageId` is already set from a prior partial attempt use `PATCH /v1/pages/{id}` instead of POST to prevent duplicate pages
- [ ] T039 [P] [US3] Stub `GET /api/auth/refresh` in `api/auth/refresh.ts`: return `{ status: 501, body: JSON.stringify({ message: 'Notion tokens do not expire; refresh not required in MVP' }) }`; `export const config = { runtime: 'edge' }`

**Checkpoint**: Full capture → OCR → review → publish → Notion page created flow works end-to-end. Failed publishes surface in banner and retry successfully without re-capture.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Recovery UI, PWA assets, error wiring, and build validation.

- [ ] T040 [P] Build `PendingEntryBanner` component in `src/components/common/PendingEntryBanner.tsx`: on mount: (1) transition any `'ocr_pending'` entries to `'ocr_failed'` (app was killed mid-OCR, blob is gone — user must re-capture); (2) transition any `'publishing'` entries to `'failed'` (app killed mid-publish, `ocrOutput` preserved); (3) call `cleanupPublishedEntries()`; (4) query `getFailedEntries()`; if any: render banner "N entries waiting to publish — tap to resume"; tapping navigates to `/review?id={oldest.id}`; re-queries on `visibilitychange` (user returns to tab); export as default
- [ ] T041 [P] Wire `PendingEntryBanner` into capture page in `src/pages/capture.tsx`: replace the stub import with the real component from T040; render `<PendingEntryBanner />` above `<CameraCapture />`
- [ ] T042 [P] Wire `ErrorBoundary` around route-level components in `src/App.tsx`: wrap each `<Route element>` in `<ErrorBoundary>`; confirm the fallback renders correctly when an unhandled error is thrown in a route subtree
- [ ] T043 [P] Add PWA icon assets in `public/icons/`: create `icon-192.png` (192×192) and `icon-512.png` (512×512) placeholder icons — simple SVG-to-PNG with black background and white "IW" text; add `icon-512-maskable.png` with safe-zone padding for maskable icon spec; update `public/manifest.json` icons array to reference all three with correct `sizes` and `purpose` fields
- [ ] T044 Scaffold Playwright E2E test stubs: create `tests/e2e/auth.spec.ts` with `test.todo('OAuth redirect returns token in hash and stores it in IndexedDB')` and `test.todo('CSRF state mismatch is rejected')`; create `tests/e2e/publish.spec.ts` with `test.todo('confirmed entry publishes to Notion and status transitions to published')`; configure `playwright.config.ts` with `baseURL: 'http://localhost:3000'` and `webServer: { command: 'vercel dev', port: 3000 }`
- [ ] T045 Run `npm run typecheck` and `npm run build` to harden types across all files; fix any TypeScript errors; confirm `vite build` produces `dist/` with `manifest.webmanifest` and a valid service worker; confirm `vercel dev` serves frontend + all `api/` Edge Functions without 500 errors on cold start

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 — **BLOCKS all user stories; auth flow must be complete before the route guard can redirect correctly**
- **US1 (Phase 3)**: Depends on Phase 2 — independent of US2 and US3
- **US2 (Phase 4)**: Depends on Phase 2 + T019 (`camera.ts`, from US1) — `SampleCapture` reuses camera abstraction; if implementing US2 before US1 stub `camera.ts` temporarily
- **US3 (Phase 5)**: Depends on Phase 2 + US1 `PendingEntry` flow (T023, T026) — publish flow extends the review page built in US1
- **Polish (Phase 6)**: Depends on all stories complete

### Within Each User Story

| Story | Parallel batch 1 | Then sequential |
|---|---|---|
| US1 | T019 (`camera.ts`), T020 (`image.ts`), T021 (`api/ocr.ts`) | T022 (ocr service) → T023 (CameraCapture) + T024 (OcrReviewScreen) → T025 (capture page) → T026 (review page) |
| US2 | T027 (`api/analyze-handwriting.ts`), T030 (`SampleCapture`) | T028 (calibration service) → T031 (OnboardingFlow) → T032 (calibration page); T029 (update ocr.ts) in parallel with T031 |
| US3 | T033 (`api/tag.ts`), T034 (`api/link-entries.ts`), T035 (tagging svc), T036 (linking svc), T039 (auth/refresh stub) | T037 (notion service) → T038 (wire review page) |

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1 (Setup)
2. Complete Phase 2 (Foundational) — app shell + auth flow + IndexedDB
3. Complete Phase 3 (US1) — camera + OCR + review
4. **STOP and VALIDATE**: photograph a real handwritten page, verify structured `OCROutput`, make a correction, inspect IndexedDB state
5. Deploy preview: `vercel deploy` → share URL, test on iOS Safari and Android Chrome

### Incremental Delivery

1. **Setup + Foundational** → app boots, OAuth works
2. **+ US1** → camera capture + formatting OCR + correction recording (data pipeline shippable)
3. **+ US2** → calibration onboarding; first real capture includes `HandwritingProfile` hints
4. **+ US3** → Notion publishing + auto-tagging + entry relations = full product loop closed
5. **+ Polish** → PWA installable, recovery banner live, build validated

---

## Constitution Check

| Principle | Enforced by |
|---|---|
| I. Formatting semantics are the product | T009 (`FormattingAnnotation` type), T021 (OCR prompt contract enforces `formatting[]`), T037 (block assembly: callout for highlights, inline strikethrough — not merged with plain text) |
| II. Calibration is first-class | T013 (`CalibrationProfile` store), T027–T032 (onboarding flow + analysis), T028 (OCR service injects hints on every call), T024 (corrections recorded on every inline edit) |
| III. Notion is the database | T012 (`PendingEntry` deleted post-publish), T037 (no content stored in proxy or Blob in MVP) |
| IV. Every entry feeds the knowledge graph | T038 (`extractTags` + `findRelatedEntries` run atomically before Notion page creation; failures surface as errors, not silent drops) |
| V. PWA-only | T005 (vite-plugin-pwa), T019 (`getUserMedia` / `input capture` — no Capacitor) |
| VI. MVP scope is fixed | `DoodleRegion` and `EditorialSymbol` defined in T009 but never instantiated; `doodles: []` and `symbols: []` hardcoded in OCR prompt (T021); no doodle upload endpoint |
