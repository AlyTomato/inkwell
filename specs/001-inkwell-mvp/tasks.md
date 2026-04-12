# Tasks: Inkwell MVP

**Input**: Design documents from `specs/001-inkwell-mvp/`
**Prerequisites**: plan.md ✅ spec.md ✅ data-model.md ✅ contracts/ ✅ research.md ✅ quickstart.md ✅

**Tests**: Not requested in spec. Test tasks omitted. Integration and E2E test scaffolding included in Polish phase.

**Organization**: Tasks grouped by user story to enable independent implementation and testing.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no incomplete task dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Initialize the Vite + React PWA project and configure all tooling before any feature work begins.

- [ ] T001 Scaffold Vite 6 + React 19 + TypeScript 5 project at repo root with `npm create vite@latest . -- --template react-ts`
- [ ] T002 [P] Install all dependencies: `dexie@4`, `@notionhq/client`, `react-router-dom@7`, `tailwindcss@4`, `vite-plugin-pwa`, `workbox-*`
- [ ] T003 [P] Configure TypeScript strict mode in `tsconfig.json` and `tsconfig.app.json`
- [ ] T004 [P] Configure Tailwind CSS v4 in `vite.config.ts` and `src/index.css`
- [ ] T005 [P] Configure `vite-plugin-pwa` in `vite.config.ts`: `display: standalone`, `scope: /`, `start_url: /`
- [ ] T006 [P] Create `public/manifest.json` with PWA metadata (name "Inkwell", short_name, theme_color, placeholder icons)
- [ ] T007 [P] Install and configure Vitest + React Testing Library in `vite.config.ts` and `src/setupTests.ts`
- [ ] T008 [P] Install and configure Playwright in `playwright.config.ts`
- [ ] T009 Create `vercel.json` with Edge Function routing for all `api/` routes and set `functions.runtime` to `edge`; document all required env vars in `.env.local.example` (`VITE_NOTION_CLIENT_ID`, `NOTION_CLIENT_SECRET`, `ANTHROPIC_API_KEY`)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Shared types, IndexedDB schema, routing shell, and camera abstraction that every user story depends on. No user story work begins until this phase is complete.

**⚠️ CRITICAL**: All of Phase 2 must be complete before any Phase 3+ work.

- [ ] T010 Define all shared TypeScript types in `src/types/index.ts`: `OCROutput`, `FormattingAnnotation`, `DoodleRegion`, `EditorialSymbol`, `CalibrationProfile`, `CalibrationSample`, `HandwritingProfile`, `CalibrationCorrection`, `PendingEntry`, `NotionAuthToken` — match data-model.md exactly including `PendingEntry.status` union with `'ocr_pending' | 'ocr_failed' | 'awaiting_review' | 'confirmed' | 'publishing' | 'published' | 'failed'`; note that `PendingEntry.ocrOutput` is `ocrOutput?: OCROutput` (optional — entry is created before OCR completes, so this field is absent until OCR succeeds)
- [ ] T011 Initialize Dexie.js v4 schema in `src/store/db.ts`: define `AppDatabase` class with tables `calibrationProfiles`, `pendingEntries`, `authTokens`; configure `version(1).stores()` with indexes from data-model.md; export singleton `db` instance
- [ ] T012 [P] Implement `CalibrationProfile` CRUD in `src/store/calibration.ts`: `getProfile()`, `createProfile()`, `updateProfile()`, `addSample()`, `addCorrection()` (with 200-item FIFO eviction), `setHandwritingProfile()`, `setOnboardingComplete()`
- [ ] T013 [P] Implement `PendingEntry` CRUD in `src/store/pending.ts`: `createEntry()` (writes immediately with `status: 'ocr_pending'`), `updateStatus()`, `setOcrOutput()`, `setNotionPageId()`, `getFailedEntries()`, `deleteEntry()`; implement `cleanupPublishedEntries()` that deletes entries with `status === 'published'` older than 24h — this function is called on app load (not a background job; PWA has no persistent background execution)
- [ ] T014 [P] Implement `NotionAuthToken` store in `src/store/auth.ts`: `getToken()`, `storeToken()`, `clearToken()`, `setJournalDatabaseId()` — `NotionAuthToken` shape must include `journalDatabaseId?: string` per contracts/auth.md
- [ ] T015 Implement React Router v7 routing shell in `src/pages/index.tsx` with two-level auth guard: (1) no token → `/onboarding/auth`; (2) token present + `!completedOnboarding` → `/onboarding/calibration`; (3) otherwise → `/capture`; handle Notion API 401 → clear token → unauthenticated
- [ ] T016 [P] Implement `ErrorBoundary` component in `src/components/common/ErrorBoundary.tsx` with fallback UI and reset capability
- [ ] T017 Implement camera abstraction in `src/lib/camera.ts`: feature-detect `ImageCapture` API at runtime; export `captureImage()` that uses `MediaDevices.getUserMedia()` stream on Android/desktop and programmatically triggers `<input type="file" accept="image/*" capture="environment">` on iOS Safari; both paths return `File | Blob`

**Checkpoint**: Routing shell renders, IndexedDB initialises without errors, camera abstraction detects platform correctly.

---

## Phase 3: User Story 1 — Capture & Formatting-Aware OCR (Priority: P1) 🎯 MVP

**Goal**: User photographs a handwritten page; app returns a structured `OCROutput` with full text, highlight/strikethrough/indent annotations, and reserved empty `doodles`/`symbols` arrays; user reviews and corrects inline; corrections are recorded as `CalibrationCorrection` entries.

**Independent Test**: Provide a handwritten page photo → verify `OCROutput` contains (a) transcribed text, (b) `formatting[]` with correctly typed annotations at valid character offsets, (c) `doodles: []`, (d) `symbols: []`. Test inline correction: tap word → type replacement → verify `CalibrationCorrection` appended to `CalibrationProfile`. No Notion integration required.

- [ ] T018 Implement `/api/ocr` Vercel Edge Function in `api/ocr.ts`: accept `OCRRequest` (imageBase64, mimeType, calibrationHints?); build Claude prompt per contracts/ocr-api.md including `<handwriting_profile>` and `<corrections>` XML blocks when hints provided; call Anthropic Messages API with `claude-opus-4-6`; return `OCRResponse` or typed `OCRError`; validate `Authorization` header present; enforce 5MB image size limit
- [ ] T019 [P] [US1] Implement image compression utility in `src/lib/image.ts`: `compressForOcr(file: File | Blob): Promise<string>` — use Canvas API to resize and re-encode as JPEG targeting ≤3.5MB raw (≤5MB base64); preserve aspect ratio; return base64 string with mimeType
- [ ] T020 [US1] Implement OCR service in `src/services/ocr.ts`: `runOcr(imageFile, calibrationHints?)` — compress image via `image.ts`, POST to `/api/ocr` with Notion Bearer token in `Authorization` header, return `OCROutput`; handle all `OCRError` codes with typed errors
- [ ] T021 [P] [US1] Implement `CameraCapture` component in `src/components/capture/CameraCapture.tsx`: render adaptive capture UI using `camera.ts` abstraction; show viewfinder on Android/desktop (`getUserMedia` stream); show native camera trigger on iOS; on capture, call `compressForOcr` and emit compressed image blob to parent; handle camera permission denial with user-facing error
- [ ] T022 [US1] Implement capture page in `src/pages/capture.tsx`: on image captured — (1) call `pending.createEntry()` immediately (status `'ocr_pending'`); (2) call `image.compressForOcr()` to compress the image before sending; (3) call `ocr.runOcr(compressedImage)`; (4) on success: `pending.setOcrOutput()` + `updateStatus('awaiting_review')` + navigate to `/review/:entryId`; (5) on OCR failure: `updateStatus('ocr_failed')` + show retry UI (no re-capture needed); render `CameraCapture`
- [ ] T023 [US1] Implement `OcrReviewScreen` in `src/components/capture/OcrReviewScreen.tsx`: render transcribed text with formatting annotations visually indicated (highlight passages in yellow, strikethrough text with line); support **text correction**: tap any word → inline input replaces it → on submit record `CalibrationCorrection` (`correctionType: 'text'`); support **formatting correction**: tap annotation label → popover to change type → record `CalibrationCorrection` (`correctionType: 'formatting'`); show "Publish to Notion" CTA
- [ ] T024 [US1] Implement calibration correction recording in `src/services/calibration.ts`: `recordTextCorrection(entryId, original, corrected, context)` and `recordFormattingCorrection(entryId, detected, actual, passageText)` — both call `store/calibration.addCorrection()` with correct `source: 'user_correction'`; export `serializeCalibrationHints(profile): CalibrationHints` — returns `handwritingProfile` (if present) + last 10 corrections by `appliedAt` desc. **Note**: T027 (US2) updates `serializeCalibrationHints()` to add the stripped wire-format shape for `handwritingProfile`; T030 adds `analyzeHandwriting()`; T032 integrates hints into the OCR call — all touch this file sequentially
- [ ] T025 [US1] Implement review page in `src/pages/review.tsx`: load `PendingEntry` by route param; render `OcrReviewScreen`; on "Publish" tap: `updateStatus('confirmed')` then trigger publish flow (stubbed for now — will be fully wired in US3); handle entry not found

**Checkpoint**: Full capture → OCR → review → correction flow works end-to-end. `PendingEntry` state machine transitions correctly. No Notion integration yet.

---

## Phase 4: User Story 2 — Handwriting Calibration Onboarding (Priority: P2)

**Goal**: First-time users complete a 3–5 sample onboarding flow; app runs a blocking handwriting analysis that populates `CalibrationProfile.handwritingProfile` and pre-seeds corrections; on failure auto-retries 3 times then offers retry/skip; returning calibrated users skip onboarding. From this point, all OCR calls include the profile.

**Independent Test**: Complete onboarding with 3 samples → verify `CalibrationProfile.completedOnboarding = true`, `handwritingProfile` is populated (or skipped gracefully), `corrections[]` contains `onboarding_diff` entries. Confirm the next OCR call includes `calibrationHints.handwritingProfile` in the request to `/api/ocr`.

- [ ] T026 Implement `/api/analyze-handwriting` Vercel Edge Function in `api/analyze-handwriting.ts`: accept `AnalyzeHandwritingRequest` (array of 3–5 sample images + promptTexts); build multi-image Claude prompt per contracts/ocr-api.md analysis section; call Anthropic Messages API; parse and return `AnalyzeHandwritingResponse` (handwritingProfile + seedCorrections ≤50 items); validate min 3 samples; return typed errors
- [ ] T027 [P] [US2] Update `serializeCalibrationHints()` in `src/services/calibration.ts` (created in T024) to support the full `CalibrationHints` wire shape: include `handwritingProfile` when present on the profile, stripping `generatedAt` and `generatedFromSampleCount` fields (wire format per contracts/ocr-api.md differs from the stored `HandwritingProfile` type); corrections array already handled in T024 — only the profile branch needs adding here
- [ ] T028 [P] [US2] Implement `SampleCapture` component in `src/components/calibration/SampleCapture.tsx`: display the prompt text the user should write; render camera trigger using `CameraCapture`; show preview of captured image; emit captured sample (imageBlob + promptText) to parent
- [ ] T029 [US2] Implement `OnboardingFlow` stepper component in `src/components/calibration/OnboardingFlow.tsx`: manage steps for samples 1–5 using `SampleCapture`; show progress indicator; enable "Skip remaining" button after sample 3 is captured; accumulate samples array; on completion (≥3 samples captured or skip triggered) emit samples array to trigger analysis
- [ ] T030 [US2] Implement handwriting analysis orchestration in `src/services/calibration.ts`: `analyzeHandwriting(samples)` — POST to `/api/analyze-handwriting`; implement auto-retry (up to 3 attempts, 2s / 4s / 8s backoff); on success: call `store.setHandwritingProfile()` + append `seedCorrections` via `store.addCorrection()` with `source: 'onboarding_diff'`; on all retries exhausted: return `{ success: false }` so UI can offer retry/skip; on skip: call `store.setOnboardingComplete()` without setting profile
- [ ] T031 [US2] Implement calibration onboarding page in `src/pages/onboarding/calibration.tsx`: render `OnboardingFlow`; on sample collection complete: show "Building your handwriting profile…" blocking screen and call `calibration.analyzeHandwriting()`; on analysis success: `setOnboardingComplete()` + navigate to `/capture`; on failure after retries: show "Try again" button (re-calls `analyzeHandwriting`) and "Skip for now" button (calls `setOnboardingComplete()` without profile + navigate to `/capture`)
- [ ] T032 [US2] Update `src/services/ocr.ts` `runOcr()` to call `serializeCalibrationHints(profile)` and include result in every OCR request; load `CalibrationProfile` from IndexedDB before each call; `calibrationHints` is omitted (not `{}`) when profile is absent (first-time user before onboarding completes)

**Checkpoint**: New user sees onboarding flow, completes 3 samples, analysis runs (or is skipped), `completedOnboarding = true`, subsequent OCR calls include `handwritingProfile` in the request payload.

---

## Phase 5: User Story 3 — Notion Publishing with Auto-Tagging & Entry Relations (Priority: P3)

**Goal**: User authenticates with Notion via OAuth; on "Publish" the app creates a structured Notion page with formatted blocks (callout for highlights, inline strikethrough, paragraph breaks), applies AI-extracted tags, links topically related existing entries — all in a single atomic API call; failed publishes are recoverable without re-capture.

**Independent Test**: Provide a confirmed `PendingEntry` with a highlight and a strikethrough → verify (a) Notion page created in workspace root database with correct block sequence, (b) `Tags` multi-select populated, (c) `Related Entries` relation set (or empty if no corpus), (d) `PendingEntry.status === 'published'`. Test retry: force Notion 503 → verify entry stays in `'failed'` state recoverable from `PendingEntryBanner`.

- [ ] T033 Implement `/api/auth/callback` Vercel Edge Function in `api/auth/callback.ts`: validate `code` and `state` query params present (400 if missing); POST to `https://api.notion.com/v1/oauth/token` with `client_secret`; redirect to `{origin}/#token=...&workspace=...&workspace_name=...&state=...` echoing `state` back for client-side CSRF validation
- [ ] T034 [P] [US3] Implement OAuth initiation and client-side CSRF validation in `src/lib/auth.ts`: `initiateNotionOAuth()` — generate UUID state, store in `sessionStorage`, redirect to Notion consent URL; `handleOAuthCallback()` — parse hash fragment, compare `state` vs `sessionStorage`, discard and redirect to `/onboarding/auth` with error on mismatch, call `store/auth.storeToken()` on success, clear hash with `history.replaceState`
- [ ] T035 [P] [US3] Implement `NotionConnect` screen in `src/components/auth/NotionConnect.tsx`: "Connect to Notion" button calls `initiateNotionOAuth()`; handle `?error=csrf` query param to show CSRF failure message
- [ ] T036 [US3] Implement onboarding auth page in `src/pages/onboarding/auth.tsx`: render `NotionConnect`; on app load check for hash fragment and call `handleOAuthCallback()` if token params present; navigate to `/onboarding/calibration` on success
- [ ] T037 [P] [US3] Implement `/api/tag` Vercel Edge Function in `api/tag.ts`: accept `TagRequest` (text: first 2000 chars); call Claude with tagging prompt from contracts/notion-schema.md; return `TagResponse` (3–8 tags, ≤30 chars each, Title Case); return typed errors
- [ ] T038 [P] [US3] Implement `/api/link-entries` Vercel Edge Function in `api/link-entries.ts`: accept `LinkEntriesRequest` (newEntryText, newEntryTags, existingEntries ≤50); call Claude with linking prompt from contracts/notion-schema.md; return `LinkEntriesResponse` (relatedPageIds ≤5); return typed errors
- [ ] T039 [US3] Implement Notion database bootstrap in `src/services/notion.ts`: `ensureDatabase(accessToken)` — if `NotionAuthToken.journalDatabaseId` is set, verify it exists via `GET /v1/databases/:id`; if 404 or unset, create database at workspace root with all required properties from contracts/notion-schema.md schema; call `store/auth.setJournalDatabaseId()` with result; return `databaseId`
- [ ] T040 [P] [US3] Implement block assembly algorithm in `src/services/notion.ts`: `assembleBlocks(ocrOutput: OCROutput): NotionBlock[]` — (1) split `text` at `paragraph_break` annotation positions; (2) for each segment: collect overlapping highlight and strikethrough annotations; emit callout block for highlighted sub-strings applying strikethrough as `annotations: { strikethrough: true }` within the callout when annotations overlap (highlight takes block-level precedence per spec); emit remaining text as paragraph blocks with inline strikethrough spans; (3) return ordered block array
- [ ] T041 [P] [US3] Implement tagging service in `src/services/tagging.ts`: `extractTags(text)` — POST to `/api/tag`; return `string[]`
- [ ] T042 [P] [US3] Implement entry linking service in `src/services/linking.ts`: `findRelatedEntries(newEntryText, newEntryTags, accessToken)` — fetch last 50 Notion pages from `journalDatabaseId` (`GET /v1/databases/:id/query`, sort by `Captured At` desc); serialize titles + tags; POST to `/api/link-entries`; return `relatedPageIds[]`
- [ ] T043 [US3] Implement full publish orchestration in `src/pages/review.tsx`: on "Publish to Notion" — (1) `updateStatus('publishing')`; (2) run `extractTags` and `findRelatedEntries` in parallel; (3) call `ensureDatabase()`; (4) call `assembleBlocks()`; (5) generate page title via `generateEntryTitle(ocrOutput, capturedAt)`: format `{YYYY-MM-DD} — {first 50 chars of ocrOutput.text}` (per contracts/notion-schema.md); (6) `POST /v1/pages` with all properties + blocks in single call per contracts/notion-schema.md atomicity contract; (7) on success: `setNotionPageId()` + `updateStatus('published')`; (8) on failure: `updateStatus('failed')` + show error "Couldn't reach Notion. Your entry is saved — tap Retry when ready."; implement retry path: if `notionPageId` is set (partial prior attempt) use `PATCH /v1/pages/:id` instead of POST to avoid duplicates
- [ ] T044 [US3] Implement `PendingEntryBanner` in `src/components/common/PendingEntryBanner.tsx`: on app load — (1) transition any `'ocr_pending'` entries to `'ocr_failed'` (app was closed mid-OCR; image is gone, retry means re-capture); (2) transition any `'publishing'` entries to `'failed'` (app was closed mid-publish; ocrOutput and tags are preserved, retry re-runs publish only); (3) call `store/pending.cleanupPublishedEntries()` (24h cleanup); (4) query for `'failed'` and `'ocr_failed'` entries and render banner "N entries waiting" if any exist; for `'failed'`: "Retry" re-runs publish from T043 skipping already-computed tags/links; for `'ocr_failed'`: "Retry" means user must re-capture (surface appropriate message); mount banner in root layout
- [ ] T045 [US3] Create stub `/api/auth/refresh.ts` Edge Function that returns `501 Not Implemented` with comment explaining Notion tokens do not expire; file must exist for future use

**Checkpoint**: Full capture → OCR → review → publish → Notion page created flow works end-to-end. Failed publishes surface in `PendingEntryBanner` and retry successfully.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: PWA installability, error boundary wiring, integration test scaffolding, and performance validation.

- [ ] T046 [P] Generate final PWA icon set (192×192, 512×512, maskable variants) and update `public/manifest.json`; verify Lighthouse PWA audit passes installability checks
- [ ] T047 [P] Wire `ErrorBoundary` around `/capture` and `/review` routes in the React Router layout; add fallback UI with "Go back" reset action
- [ ] T048 Write integration test scaffolding for all 4 quickstart scenarios in `tests/integration/publish-flow.test.ts` using Vitest with mocked Notion API and mocked Claude responses; scenarios must match quickstart.md observable outcomes exactly
- [ ] T049 [P] Write Playwright E2E test in `tests/e2e/auth.spec.ts`: OAuth initiation → callback handler → CSRF state validation → token stored → calibration screen shown
- [ ] T050 [P] Write Playwright E2E test in `tests/e2e/publish.spec.ts`: authenticated + calibrated user → camera capture → OCR review → publish → Notion sandbox page created with correct properties
- [ ] T051 Validate SC-001 (capture-to-publish ≤90s) on a throttled mobile viewport in Playwright; measure OCR response time (target p95 ≤10s) and Notion page creation (target p95 ≤3s); document results

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 completion — **BLOCKS all user stories**
- **US1 (Phase 3)**: Depends on Phase 2 completion — no dependency on US2 or US3
- **US2 (Phase 4)**: Depends on Phase 2 + US1 camera component (T021) — reuses `CameraCapture` for sample collection
- **US3 (Phase 5)**: Depends on Phase 2 + US1 `PendingEntry` flow (T022, T025) — extends the publish trigger stubbed in T025
- **Polish (Phase 6)**: Depends on US1 + US2 + US3 all complete

### User Story Dependencies

- **US1 → US2**: `OnboardingFlow` reuses `CameraCapture` (T021); `ocr.ts` (T020) is updated in T032 to inject hints
- **US1 → US3**: Publish flow in `review.tsx` (T025) is stubbed in US1 and fully wired in T043
- **US2 → US3**: `CalibrationHints` (T027) used in every OCR call; no direct US3 dependency on US2 internals

### Parallel Opportunities Within Each Phase

**Phase 1**: T002–T009 all parallelizable after T001.

**Phase 2**: T012, T013, T014, T016 fully parallel after T011. T017 parallel with all store tasks.

**Phase 3 (US1)**:
```
T018 (OCR Edge Function)    ← parallel with T019, T021
T019 (image.ts)             ← parallel with T018, T021
T021 (CameraCapture)        ← parallel with T018, T019
           ↓ all complete
T020 (ocr service)          ← depends on T018, T019
T022 (capture page)         ← depends on T020, T021
T023 (OcrReviewScreen)      ← parallel with T022 (different file)
T024 (correction recording) ← parallel with T022, T023
           ↓ all complete
T025 (review page)          ← depends on T022, T023, T024
```

**Phase 4 (US2)**:
```
T026 (analyze-handwriting Edge Function)  ← parallel with T027, T028
T027 (CalibrationHints serializer update) ← parallel with T026, T028
T028 (SampleCapture component)            ← parallel with T026, T027
           ↓ all complete
T029 (OnboardingFlow)    ← depends on T028
T030 (analysis service)  ← depends on T026, T027
           ↓ both complete
T031 (calibration page)  ← depends on T029, T030
T032 (update ocr.ts)     ← depends on T027; parallel with T031
```

**Phase 5 (US3)**:
```
T033 (auth callback)  T034 (auth lib)  T037 (tag Edge Fn)  T038 (link Edge Fn)  T040 (block assembly)
        ↓                  ↓                  ↓                    ↓
T035 (NotionConnect)  T036 (auth page)  T041 (tagging svc)  T042 (linking svc)
                           ↓ auth complete
                      T039 (DB bootstrap)
                           ↓ all services complete
                      T043 (publish orchestration)
                           ↓
                      T044 (PendingEntryBanner)
T045 (auth/refresh stub) ← fully parallel, no dependencies
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational
3. Complete Phase 3: User Story 1 (T018–T025)
4. **STOP and VALIDATE**: Capture a real handwritten page, verify `OCROutput` structure, test inline correction flow
5. Deploy to Vercel preview URL, test on iOS and Android

### Incremental Delivery

1. Setup + Foundational → project shell boots, routing works
2. **US1 complete** → camera capture + formatting OCR + review screen (shippable data pipeline)
3. **US2 complete** → calibration onboarding + handwriting profile improves OCR from first real capture
4. **US3 complete** → Notion publishing + tagging + relations = full product loop
5. Polish → PWA installable, E2E tests pass, performance validated

---

## Notes

- `[P]` tasks operate on different files with no shared incomplete dependencies — safe to parallelize
- Every user story is independently testable at its checkpoint without requiring later stories
- `PendingEntry` is created before the OCR call (T022) — no capture is silently lost on API failure
- The publish flow (T043) re-uses already-computed tags/links on retry — never re-charges API calls for data already in the `PendingEntry`
- The `<input capture>` iOS path (T017) and `getUserMedia` Android/desktop path produce identical `File | Blob` output — the OCR pipeline never needs to know which camera path was used
- Commit after each task or logical group; each phase checkpoint is a valid deploy point
