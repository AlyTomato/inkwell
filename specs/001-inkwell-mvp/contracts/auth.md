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
// 1. Validate state matches session (CSRF)
// 2. POST to https://api.notion.com/v1/oauth/token with:
//    - client_id, client_secret (env var)
//    - grant_type: 'authorization_code'
//    - code, redirect_uri
// 3. Receive access_token, workspace_id, workspace_name, bot_id
// 4. Redirect to: {origin}/#token=<access_token>&workspace=<workspace_id>
//    Note: hash fragment is never sent to any server
```

**Response contract** (redirect target URL hash):
```
/#token=<access_token>&workspace=<workspace_id>&workspace_name=<encoded_name>
```

### 3. Token Storage (browser-side)

```typescript
// PWA reads hash on load, stores in IndexedDB, clears hash from URL
interface NotionAuthToken {
  accessToken: string;
  workspaceId: string;
  workspaceName: string;
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
