# Contract: Authentication Flow

**Consumer**: PWA frontend (`src/lib/auth.ts`)
**Provider**: Vercel Edge Functions (`api/auth/callback.ts`, `api/auth/refresh.ts`) + Notion OAuth

---

## OAuth Flow

### 1. Initiation (browser-side)

```typescript
// src/lib/auth.ts
function initiateNotionOAuth(): void {
  const state = crypto.randomUUID();          // CSRF protection
  sessionStorage.setItem('oauth_state', state);

  const params = new URLSearchParams({
    client_id: import.meta.env.VITE_NOTION_CLIENT_ID,
    redirect_uri: `${window.location.origin}/api/auth/callback`,
    response_type: 'code',
    state,
    owner: 'user',
  });

  window.location.href = `https://api.notion.com/v1/oauth/authorize?${params}`;
}
```

### 2. Callback Handler (serverless)

**Endpoint**: `GET /api/auth/callback?code=<code>&state=<state>`

```typescript
// api/auth/callback.ts (Vercel Edge Function)
// 1. Verify 'code' and 'state' query params are present (400 if missing)
// 2. POST to https://api.notion.com/v1/oauth/token with:
//    - client_id, client_secret (env var)
//    - grant_type: 'authorization_code'
//    - code, redirect_uri
// 3. Receive access_token, workspace_id, workspace_name, bot_id
// 4. Redirect to: {origin}/#token=<access_token>&workspace=<workspace_id>&state=<state>
//    Note: hash fragment is never sent to any server.
//    'state' is echoed back so the PWA can perform CSRF validation client-side.
```

**Note**: The serverless function has no access to browser sessionStorage and therefore cannot
validate the state value against the original request. CSRF protection is enforced client-side
in step 3 below, which is the correct location for this check.

**Response contract** (redirect target URL hash):
```
/#token=<access_token>&workspace=<workspace_id>&workspace_name=<encoded_name>&state=<state>
```

### 3. Token Storage (browser-side)

```typescript
// PWA reads hash on load:
// 1. Parse token, workspace_id, workspace_name, state from hash fragment
// 2. CSRF check: compare state against sessionStorage.getItem('oauth_state')
//    → If mismatch or missing: discard token, redirect to /onboarding/auth with error
//    → If match: proceed
// 3. Clear sessionStorage 'oauth_state'
// 4. Store token in IndexedDB
// 5. Clear hash from URL (history.replaceState)
interface NotionAuthToken {
  accessToken: string;
  workspaceId: string;
  workspaceName: string;
  journalDatabaseId?: string;      // populated after first successful bootstrap; undefined until first publish
  storedAt: number;                // Unix ms — tokens do not expire per Notion docs
}
```

**Storage**: IndexedDB (same Dexie instance), `auth` table, single record.
**Note**: Notion OAuth tokens do not have an expiry as of the current API version. No refresh flow is required for MVP. `api/auth/refresh.ts` is stubbed for future use.

---

## Auth State Machine

```
unauthenticated
      │
      ├─ [user opens app, no token in IndexedDB]
      │         → show "Connect to Notion" screen
      │
      ├─ [user taps Connect] → initiateNotionOAuth()
      │         → Notion consent screen
      │         → /api/auth/callback
      │         → token stored in IndexedDB
      │
      └─ authenticated
                │
                ├─ [token present in IndexedDB on app load] → skip auth screen
                └─ [Notion API returns 401] → clear token, return to unauthenticated
```

---

## Protected Routes Contract

All routes except `/` (auth check redirect) and `/onboarding/auth` require `NotionAuthToken.accessToken` to be present in IndexedDB. If absent, redirect to `/onboarding/auth`.

Route guard pseudocode:
```typescript
if (!authToken && route !== '/onboarding/auth') {
  navigate('/onboarding/auth');
}
if (authToken && !calibrationProfile?.completedOnboarding && route !== '/onboarding/calibration') {
  navigate('/onboarding/calibration');
}
```

---

## Environment Variables Contract

| Variable | Where | Description |
|---|---|---|
| `NOTION_CLIENT_ID` | Vercel env (also exposed as `VITE_NOTION_CLIENT_ID`) | Public Notion OAuth client ID |
| `NOTION_CLIENT_SECRET` | Vercel env (server-only, never in bundle) | Secret for token exchange |
| `ANTHROPIC_API_KEY` | Vercel env (server-only) | Claude API key |
| `BLOB_READ_WRITE_TOKEN` | Vercel env (server-only) | Vercel Blob access token |
