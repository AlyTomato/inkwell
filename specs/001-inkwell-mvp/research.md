# Research: Inkwell MVP

**Feature**: 001-inkwell-mvp
**Phase**: 0 — Architecture & Integration Research
**Date**: 2026-04-10

---

## 1. API Key Security — Can Vision AI Be Called From the Browser?

**Decision**: No. A stateless serverless proxy is required.

**Rationale**: Both the Anthropic and OpenAI APIs require an `Authorization` header with a secret API key. Embedding this key in a browser bundle exposes it to any user who inspects network traffic. The proxy holds zero persistent state and stores no user content — it receives a request, forwards it to the AI provider with the secret key, and returns the response. This does not violate Constitution Principle III (no proprietary content storage) or Principle V (PWA-only distribution) because it is not a content database and not a native wrapper — it is a stateless key vault.

**Chosen stack for proxy**: Vercel Edge Functions (co-located in `/api/` alongside the frontend). Alternatives: Cloudflare Workers (equally valid, lower cold-start latency), Netlify Functions (less ergonomic for TypeScript).

**Alternatives considered**:
- Call APIs directly from the browser: Rejected — exposes secret key.
- Service Worker as proxy: Rejected — service workers are client-side; the key would still be in the bundle.
- User-provided API key stored in IndexedDB: Viable alternative but degrades UX significantly (setup friction). Revisit post-MVP if serverless hosting cost becomes a concern.

---

## 2. Vision AI Provider Selection

**Decision**: Anthropic Claude (`claude-opus-4-6` vision) as the primary provider, with GPT-4o vision as a documented fallback.

**Rationale**: Claude has strong structured output capabilities (JSON mode) needed for the OCR output contract. The `claude-opus-4-6` model supports image input natively. The formatting detection task (identifying highlights, strikethroughs, doodles) benefits from Claude's instruction-following fidelity on complex extraction prompts. GPT-4o vision is equivalent in capability; the serverless proxy interface is intentionally AI-agnostic so the provider can be swapped without frontend changes.

**Integration pattern**: The PWA POSTs a base64-encoded image and a structured prompt to `/api/ocr`. The proxy calls the Anthropic Messages API with the image and returns the structured OCR output JSON. The prompt instructs the model to return the OCR output contract shape exactly.

**Calibration integration**: The calibration profile's correction history is serialized and included as few-shot examples in the OCR prompt (`<corrections>` XML block). This guides the vision model toward the user's specific letterforms without fine-tuning.

**Alternatives considered**:
- Google Document AI: Good OCR baseline but no formatting semantic detection out of the box; would require post-processing.
- Azure Computer Vision: Similar limitation.
- On-device OCR (Tesseract WASM): Cannot detect formatting semantics; too slow for mobile; accuracy requires handwriting-specific training data we don't have.

---

## 3. Notion OAuth for a PWA

**Decision**: Standard OAuth 2.0 Authorization Code flow with the token exchange handled by the serverless proxy.

**Rationale**: Notion's OAuth requires a `client_secret` for the authorization code → access token exchange, which cannot be done browser-side. The flow is:
1. PWA redirects user to Notion's OAuth consent screen with `client_id`, `redirect_uri`, `state`
2. Notion redirects to `https://{app}/api/auth/callback?code=...&state=...`
3. The serverless callback handler exchanges `code` for an `access_token` using the `client_secret`
4. The callback redirects the user back to the PWA with the token in a short-lived secure cookie or in the URL hash (hash is preferred — it is never sent to any server)
5. The PWA stores the token in IndexedDB and uses it for all subsequent Notion API calls directly (Notion API supports CORS for browser calls once you have a token)

**Key point**: After OAuth completes, all Notion API calls (page creation, property updates, relation linking) are made directly from the browser to `api.notion.com` using the user's access token. The proxy is only needed for the code exchange and AI API calls.

**Alternatives considered**:
- PKCE without a client_secret: The Notion OAuth docs do not currently support PKCE; client_secret is required.
- Storing the access token in localStorage: Acceptable for MVP (low XSS surface for a single-user PWA); upgrade to HttpOnly cookie post-MVP.

---

## 4. Entry-to-Entry Semantic Similarity

**Decision**: Claude prompt-based similarity at publish time. No embedding infrastructure for MVP.

**Rationale**: At MVP corpus size (tens to low hundreds of entries), a prompt-based approach is fast enough and avoids embedding storage complexity. The flow: on each publish, fetch the titles, dates, and tag arrays of the 50 most recent existing Notion pages via the Notion API, serialize them as a compact list, and ask Claude to identify which are topically related to the new entry. Claude returns a list of Notion page IDs to link. This adds ~1-2 seconds to the publish time at small corpus sizes.

**Fallback threshold**: If the Notion database exceeds 200 entries, query the most recent 50 by date (recency bias acceptable at MVP scale).

**Alternatives considered**:
- Voyage AI / OpenAI embeddings + cosine similarity: Correct long-term solution. Requires storing embeddings per entry (Notion property or external vector DB). Adds API dependency, storage complexity, and another secret to manage. Revisit at ~500 entries.
- Notion's built-in search: Too broad — searches across the entire workspace, not semantically.
- Tag intersection only (no AI): Simple but misses semantic synonyms and topic relationships not captured in exact tag matches.

---

## 5. Doodle Image Hosting for Notion Image Blocks *(deferred post-MVP)*

**Decision**: Deferred. Doodle detection and embedding have been descoped from MVP (2026-04-11). The research below is retained for when this feature ships.

**Original decision**: Vercel Blob (indefinite retention) for MVP doodle image storage.

**Rationale**: The Notion API's `image` block type supports only `external` URLs — Notion does not expose a file upload API endpoint. Doodle crops must be hosted somewhere before they can be embedded in a Notion page. Vercel Blob provides a simple `put()` API co-located with the serverless functions, generates a stable `blob.vercel-storage.com` URL, and has a generous free tier for small binary assets. Blobs are set to `access: 'public'` with no expiry so Notion image URLs remain valid indefinitely.

**Data classification note**: Doodle image storage is outside-the-lines of Principle III but does not violate it. Principle III prohibits a "proprietary long-term storage layer for entry content." Doodle crops are binary media assets attached to entry content that lives in Notion; they are not the entry itself. The tradeoff is documented here in case the interpretation needs revisiting.

**Upload flow**:
1. Browser crops doodle region from the capture photo using the Canvas API (`canvas.getContext('2d').drawImage(...)`)
2. Crop is converted to a Blob and POSTed to `/api/upload-doodle`
3. Serverless function calls `put(filename, blob, { access: 'public' })` via `@vercel/blob`
4. Returns the stable public URL
5. PWA constructs the Notion `image` block with the URL

**Alternatives considered**:
- Cloudinary free tier: Works but adds another third-party dependency and requires account setup.
- Base64 inline in Notion: Not supported by Notion's block API.
- User's own S3/R2 bucket: Too much setup friction for a general user.
- Skip image hosting; use text callout fallback for doodles: Rejected — Constitution Principle I requires formatting fidelity; the user explicitly chose Option B for doodles in the spec clarification.

---

## 6. Calibration Storage: Dexie.js Over Raw IndexedDB

**Decision**: Dexie.js v4 as the IndexedDB abstraction layer.

**Rationale**: Raw IndexedDB is verbose and requires manual transaction management. Dexie provides a typed, Promise-based interface with schema versioning (critical for calibration model evolution), compound indexes, and excellent TypeScript support. Dexie v4 supports live queries for reactive UI updates. The bundle size is ~22KB gzipped — acceptable for a PWA.

**Schema versioning strategy**: CalibrationProfile schema changes (e.g., adding symbol correction history post-MVP) are handled via Dexie's `version().upgrade()` migration pattern, preserving existing user data.

**Alternatives considered**:
- Raw IndexedDB: Correct but verbose. No advantage over Dexie for this use case.
- localStorage: Too small (5MB limit); cannot store image blobs for calibration samples.
- Origin Private File System (OPFS): Better for large blobs; overkill for calibration sample sizes at MVP.

---

## 7. iOS Safari Camera API Limitations

**Decision**: Use `<input type="file" accept="image/*" capture="environment">` as the primary capture mechanism on iOS, with `MediaDevices.getUserMedia()` as a progressive enhancement for desktop/Android.

**Rationale**: iOS Safari has notable `MediaDevices` constraints:
- `ImageCapture` API is not supported on iOS as of iOS 17
- `getUserMedia()` works in Safari 14.5+ but camera streams cannot be easily paused/resumed without re-requesting permission
- The `<input capture>` approach uses the native iOS camera app, bypassing all browser API limitations, and is universally supported

**Detection strategy**: Feature-detect `ImageCapture` support at runtime. If available (Chrome on Android / desktop), use the stream-based capture flow for a richer in-app camera experience. If not (Safari/iOS), fall back to `<input capture>` which launches the native camera. Both paths produce the same `File`/`Blob` output for the OCR pipeline.

**PWA installability note**: The `<input capture>` approach works correctly in installed PWA mode on iOS (home screen icon). Camera permission is requested at first use and remembered.

**Alternatives considered**:
- Force `getUserMedia()` everywhere: Breaks on iOS Safari when `ImageCapture` is unavailable; stream handling is more complex.
- Capacitor/Cordova for native camera: Rejected — Constitution Principle V prohibits native wrappers.

---

## 8. Frontend Framework

**Decision**: React 19 + TypeScript 5.x + Vite 6 + `vite-plugin-pwa`.

**Rationale**: React has the broadest PWA ecosystem, best Vite integration, and widest availability of Notion API client examples. TypeScript 5 provides the type safety needed for the OCR output contract. `vite-plugin-pwa` (powered by Workbox) handles service worker generation, manifest injection, and offline shell caching with minimal configuration.

**UI component approach**: Tailwind CSS for styling (utility-first, no runtime overhead, works well with Vite). No component library for MVP — the screen count is small enough (capture, review, calibration onboarding, settings) that a component library adds more than it saves.

**Routing**: React Router v7 (file-based routing optional, but component-based routing is simpler for ~4 screens).

**Alternatives considered**:
- Vue 3 + Vite: Equivalent capability, slightly smaller ecosystem for Notion integrations.
- SvelteKit: Lighter weight but steeper learning curve; less relevant when "web dev background" implies React familiarity.
- Next.js: App Router + API Routes would eliminate the separate serverless setup. Considered seriously — rejected for MVP because Next.js adds SSR complexity that a fully client-side PWA doesn't need; also makes PWA service worker management harder. Revisit if the proxy API surface grows significantly.
