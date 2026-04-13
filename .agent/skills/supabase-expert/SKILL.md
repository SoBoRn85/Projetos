---
name: supabase-expert
description: Supabase development principles and decision-making. Auth, Row Level Security (RLS), pgcrypto encryption, Storage, Edge Functions, Realtime, and Next.js integration. Use when building with Supabase, designing RLS policies, encrypting sensitive data, configuring Supabase Auth, uploading files to Supabase Storage, or integrating Supabase with Next.js. Use PROACTIVELY whenever the project uses Supabase as backend. Do NOT use for generic PostgreSQL without Supabase, or for Firebase/AWS Amplify.
allowed-tools: Read, Write, Edit, Glob, Grep
---

# Supabase Expert

> **Learn to THINK about Supabase architecture, not copy snippets.**
> Use this skill whenever Supabase is the backend — even if the user doesn't mention it explicitly.

---

## ⚠️ Core Principle

- ASK user about Supabase project setup when unclear
- Choose client-side vs server-side based on CONTEXT
- RLS is not optional — it's the security foundation
- Never expose service_role key to the client

---

## 1. Auth Strategy

### Decision Tree

```
How to authenticate?
│
├── Email/Password
│   └── signUp() + signInWithPassword()
│
├── Magic Link
│   └── signInWithOtp({ email })
│
├── Social (Google, GitHub, etc.)
│   └── signInWithOAuth({ provider })
│
├── Phone/SMS
│   └── signInWithOtp({ phone })
│
└── SSO / SAML
    └── Enterprise only — Supabase Pro+
```

### Auth + Next.js Integration

```
Which Supabase client to use?
│
├── Client Component (browser)
│   └── createBrowserClient() from @supabase/ssr
│
├── Server Component / RSC
│   └── createServerClient() with cookies()
│
├── Route Handler / API Route
│   └── createServerClient() with cookies()
│
├── Middleware
│   └── createServerClient() with request/response
│
└── Server Action
    └── createServerClient() with cookies()
```

### Auth Best Practices

| Practice | Why |
|----------|-----|
| Use `@supabase/ssr` | Handles cookies/sessions correctly |
| Middleware for session refresh | Prevents expired tokens on navigation |
| Protect routes in middleware | Don't rely only on client checks |
| Store user metadata in `auth.users` | Use `raw_user_meta_data` for profile data |
| Redirect after auth | Handle callback URL properly |

### Auth Anti-Patterns

| ❌ Don't | ✅ Do |
|----------|-------|
| Use `@supabase/auth-helpers` (deprecated) | Use `@supabase/ssr` |
| Check auth only on client | Verify on server + middleware |
| Store passwords yourself | Let Supabase Auth handle it |
| Skip email confirmation in prod | Enable confirmation flow |

---

## 2. Row Level Security (RLS)

### RLS Decision Framework

```
Every table MUST have RLS:
│
├── Public data (read-only)
│   └── POLICY: SELECT for anon + authenticated
│
├── User-owned data
│   └── POLICY: auth.uid() = user_id
│
├── Organization/team data
│   └── POLICY: user belongs to org (via join)
│
├── Admin-only data
│   └── POLICY: user role = 'admin' (via metadata or table)
│
└── Sensitive/encrypted data
    └── POLICY: owner only + server-side decryption
```

### Common RLS Patterns

```sql
-- User can only see their own data
CREATE POLICY "Users see own data" ON user_configs
  FOR SELECT USING (auth.uid() = user_id);

-- User can insert their own data
CREATE POLICY "Users insert own data" ON user_configs
  FOR INSERT WITH CHECK (auth.uid() = user_id);

-- User can update their own data
CREATE POLICY "Users update own data" ON user_configs
  FOR UPDATE USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

-- Service role bypasses RLS (for Edge Functions)
-- No policy needed — service_role skips RLS by default
```

### RLS Anti-Patterns

| ❌ Don't | ✅ Do |
|----------|-------|
| Disable RLS on any table | Enable RLS on ALL tables |
| Use service_role on client | Use anon key on client |
| Write complex RLS (slow queries) | Keep policies simple, use helper functions |
| Forget FOR ALL vs specific operations | Be explicit: SELECT, INSERT, UPDATE, DELETE |

---

## 3. pgcrypto — Encrypting Sensitive Data

### When to Encrypt

```
Encrypt at database level when:
├── Storing third-party credentials (API keys, passwords)
├── PII that must be encrypted at rest
├── Financial data
└── Anything that would be catastrophic if DB is breached

Do NOT use pgcrypto for:
├── User passwords (Supabase Auth handles this)
├── Data you need to search/filter by
└── Large blobs (use Storage + server-side encryption)
```

### Encryption Patterns

```sql
-- Enable pgcrypto extension
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Symmetric encryption (AES-256)
-- Encrypt
UPDATE user_configs SET e2doc_creds_enc =
  pgp_sym_encrypt(
    '{"empresa":"X","login":"Y","senha":"Z"}'::text,
    current_setting('app.encryption_key')
  );

-- Decrypt (only in server-side/Edge Functions)
SELECT pgp_sym_decrypt(
  e2doc_creds_enc::bytea,
  current_setting('app.encryption_key')
) AS e2doc_creds
FROM user_configs WHERE user_id = auth.uid();
```

### Security Checklist

- [ ] Encryption key stored in Supabase Vault (not in code)
- [ ] Decryption happens only server-side (Edge Functions / API)
- [ ] RLS prevents access to encrypted columns by other users
- [ ] Key rotation plan defined
- [ ] Backup encryption strategy

---

## 4. Supabase Storage

### Storage Decision Tree

```
What to store?
│
├── User-uploaded documents (PDFs, images)
│   └── Private bucket + RLS policies
│
├── Public assets (logos, avatars)
│   └── Public bucket
│
├── Temporary files (processing queue)
│   └── Private bucket + TTL cleanup
│
└── Large files (videos, datasets)
    └── Consider resumable uploads
```

### Storage Policies

```sql
-- Private bucket: user can only access their files
CREATE POLICY "Users access own files" ON storage.objects
  FOR ALL USING (
    bucket_id = 'documents' AND
    auth.uid()::text = (storage.foldername(name))[1]
  );
```

### Best Practices

| Practice | Why |
|----------|-----|
| Organize by user_id folders | Natural RLS boundary |
| Use signed URLs for downloads | Time-limited access |
| Validate file types server-side | Prevent malicious uploads |
| Set size limits | Prevent abuse |

---

## 5. Edge Functions

### When to Use

```
Edge Functions are for:
├── Server-side logic that needs service_role
├── Decrypting pgcrypto data
├── Calling external APIs (Gemini, Google Sheets)
├── Webhooks and event processing
├── Background tasks (with pg_cron triggers)
└── Any logic that shouldn't run in the browser

Edge Functions are NOT for:
├── Simple CRUD (use PostgREST directly)
├── Static content serving
└── Long-running processes (>60s limit)
```

### Edge Function Pattern

```typescript
// supabase/functions/decrypt-creds/index.ts
import { createClient } from "@supabase/supabase-js";

Deno.serve(async (req) => {
  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")! // Server-side only!
  );

  // Verify user from JWT
  const authHeader = req.headers.get("Authorization")!;
  const { data: { user } } = await supabase.auth.getUser(
    authHeader.replace("Bearer ", "")
  );

  if (!user) return new Response("Unauthorized", { status: 401 });

  // Decrypt with service role (bypasses RLS)
  const { data } = await supabase.rpc("decrypt_user_creds", {
    p_user_id: user.id,
  });

  return new Response(JSON.stringify(data), {
    headers: { "Content-Type": "application/json" },
  });
});
```

---

## 6. Realtime

### When to Use Realtime

| Scenario | Use Realtime? |
|----------|---------------|
| Dashboard live status | ✅ Yes |
| Chat / messaging | ✅ Yes |
| Automation progress tracking | ✅ Yes |
| User settings changes | ❌ No — on-demand fetch |
| Batch data updates | ❌ No — polling or webhook |

### Realtime Patterns

```typescript
// Subscribe to automation_logs changes
const channel = supabase
  .channel("automation-status")
  .on(
    "postgres_changes",
    {
      event: "INSERT",
      schema: "public",
      table: "automation_logs",
      filter: `user_id=eq.${userId}`,
    },
    (payload) => {
      updateDashboard(payload.new);
    }
  )
  .subscribe();

// Cleanup on unmount
return () => supabase.removeChannel(channel);
```

---

## 7. Database Functions (RPCs)

### When to Use RPCs

```
Use database functions (RPCs) when:
├── Complex queries that PostgREST can't express
├── Atomic multi-table operations
├── Server-side decryption
├── Aggregations and reports
└── Business logic in the database layer
```

### RPC Pattern

```sql
-- Create a function to decrypt and return user credentials
CREATE OR REPLACE FUNCTION decrypt_user_creds(p_user_id UUID)
RETURNS JSON
LANGUAGE plpgsql
SECURITY DEFINER -- Runs with function owner's permissions
AS $$
DECLARE
  result JSON;
BEGIN
  -- Verify caller is the user
  IF auth.uid() != p_user_id THEN
    RAISE EXCEPTION 'Unauthorized';
  END IF;

  SELECT json_build_object(
    'e2doc', pgp_sym_decrypt(e2doc_creds_enc::bytea, current_setting('app.encryption_key')),
    'sic', pgp_sym_decrypt(sic_creds_enc::bytea, current_setting('app.encryption_key'))
  ) INTO result
  FROM user_configs
  WHERE user_id = p_user_id;

  RETURN result;
END;
$$;
```

---

## 8. Next.js + Supabase Architecture

### File Structure

```
app/
├── (auth)/
│   ├── login/page.tsx
│   ├── callback/route.ts      ← Auth callback handler
│   └── layout.tsx              ← Unauthenticated layout
├── (dashboard)/
│   ├── layout.tsx              ← Authenticated layout + session check
│   ├── page.tsx                ← Main dashboard
│   └── settings/page.tsx
├── lib/
│   └── supabase/
│       ├── client.ts           ← createBrowserClient()
│       ├── server.ts           ← createServerClient()
│       └── middleware.ts       ← Session refresh logic
└── middleware.ts               ← Route protection
```

### Middleware Pattern

```typescript
// middleware.ts — Session refresh + route protection
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => request.cookies.getAll(),
        setAll: (cookiesToSet) => {
          cookiesToSet.forEach(({ name, value, options }) => {
            response.cookies.set(name, value, options);
          });
        },
      },
    }
  );

  const { data: { user } } = await supabase.auth.getUser();

  // Redirect unauthenticated users
  if (!user && !request.nextUrl.pathname.startsWith("/login")) {
    return NextResponse.redirect(new URL("/login", request.url));
  }

  return response;
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"],
};
```

---

## 9. Decision Checklist

Before implementing with Supabase:

- [ ] **RLS enabled on ALL tables?**
- [ ] **Auth strategy chosen?** (email, social, magic link)
- [ ] **Client type correct?** (browser vs server vs middleware)
- [ ] **Sensitive data encrypted?** (pgcrypto for credentials)
- [ ] **Encryption key in Vault?** (not hardcoded)
- [ ] **Storage policies defined?** (per-user access)
- [ ] **Edge Functions for server logic?** (not client-side)
- [ ] **Realtime only where needed?** (don't overuse)
- [ ] **Middleware refreshing sessions?**
- [ ] **Using `@supabase/ssr`?** (not deprecated helpers)

---

## 10. Anti-Patterns

| ❌ Don't | ✅ Do |
|----------|-------|
| Disable RLS "temporarily" | Design RLS from day 1 |
| Expose service_role key to client | Use anon key + RLS on client |
| Store encryption keys in .env.local | Use Supabase Vault |
| Skip middleware session refresh | Always refresh in middleware |
| Use `@supabase/auth-helpers` | Use `@supabase/ssr` (current) |
| Put all logic in Edge Functions | Use PostgREST for simple CRUD |
| Decrypt credentials on client | Decrypt in Edge Functions only |

---

> **Remember:** Supabase is not just a database — it's a complete backend platform. Think about Auth, RLS, Storage, Edge Functions, and Realtime as an integrated system. Every decision should consider the security boundary: what runs on the client vs. what runs on the server.
