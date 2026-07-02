# FeeZoapp — Cloudflare Pages Functions Setup

## What this adds

Three server-side functions that hide your Supabase keys from source code
and add rate limiting to auth endpoints:

| Endpoint | What it does |
|---|---|
| `GET /api/config` | Returns Supabase URL + anon key at runtime (from env vars) |
| `POST /api/auth/login` | Proxies login — rate limited (5/min per IP) |
| `POST /api/auth/logout` | Proxies logout |
| `POST /api/signup` | Proxies signup — rate limited (3/hr per IP), blocks disposable emails |

## Deploy steps

### Step 1 — Push to GitHub

Push your entire feezoapp folder to GitHub (replace old files).

### Step 2 — Connect to Cloudflare Pages

1. Cloudflare dashboard → **Workers & Pages** → **Create** → **Pages**
2. Connect your GitHub repo
3. Build settings:
   - Build command: (leave empty)
   - Build output directory: `/`
4. Click **Save and Deploy**

### Step 3 — Add environment variables

In Cloudflare Pages → your project → **Settings** → **Environment variables**:

| Variable | Value |
|---|---|
| `SUPABASE_URL` | `https://bwrvhrxfacoiheetistl.supabase.co` |
| `SUPABASE_ANON_KEY` | `sb_publishable_3FY3aWZ4mScvAsThTGOzlA_PQNzOWeB` |
| `ALLOWED_ORIGIN` | `https://yourapp.pages.dev` (your CF Pages URL) |

Add these for **both** Production and Preview environments.

### Step 4 — Create KV namespace for rate limiting

1. Cloudflare dashboard → **Workers & Pages** → **KV** → **Create namespace**
2. Name it: `RATE_LIMIT_KV`
3. Click Create
4. Copy the **Namespace ID**
5. Go back to your Pages project → **Settings** → **Functions**
6. Under **KV namespace bindings** → Add:
   - Variable name: `RATE_LIMIT_KV`
   - KV namespace: select `RATE_LIMIT_KV`

### Step 5 — Test the functions

After deploy, test in browser:

```
https://yourapp.pages.dev/api/config
```

Should return:
```json
{ "url": "https://bwrvhr...supabase.co", "key": "sb_publishable_..." }
```

### Step 6 — Verify keys are gone from source

1. Open your app in the browser
2. DevTools → Sources → feezoapp → js → config.js
3. Search for "bwrvhr" — should find nothing ✅
4. Search for "sb_publishable" — should find nothing ✅

The keys now only appear in the network tab (`/api/config` response),
not in any source file.

## What's protected

| Risk | Before | After |
|---|---|---|
| Keys visible in source code | ❌ Exposed | ✅ Hidden in env vars |
| Brute force login | ❌ No limit | ✅ 5 attempts/min/IP |
| Signup spam | ❌ No limit | ✅ 3 signups/hr/IP |
| Disposable email signups | ❌ Allowed | ✅ Blocked |
| Rate limit bypass via VPN | ⚠️ Still possible | ℹ️ Use Cloudflare Bot Fight mode for extra protection |

## What's NOT changed

- All data reads/writes still go direct to Supabase (protected by RLS)
- Realtime subscriptions still go direct (can't proxy WebSockets)
- The anon key is still visible in the `/api/config` network response
  (this is unavoidable — Supabase needs it on the client to work)

## The real security: your RLS policies

The anon key alone is worthless without a valid JWT. Your RLS policies
(Stage 2 of the rebuild) ensure:
- A logged-out user can read nothing
- A logged-in user can only see their own academy's data
- A superadmin can only see metadata unless elevation is granted

These cannot be bypassed from the browser regardless of key visibility.
