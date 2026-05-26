---
name: supabase-edge-function-patterns
description: "Use when creating a new Supabase Edge Function, modifying an existing one, designing the client-side API contract (request shape, error codes, response normalization), implementing per-user daily quotas for AI / expensive operations, handling JWT verification with verify_jwt=true, or writing CORS preflight handling. Covers the standard structure, the normalized error code table (401/403/400/429/402/502/500), how to distinguish daily-limit-exceeded from upstream rate limits, the convention 'Edge Functions never write to DB' (clients persist response themselves), and quota table design."
---

# Supabase Edge Function Patterns

A reusable template + convention set for Supabase Edge Functions that:
- Get called from multiple clients (Web, iOS, Android)
- Handle AI / expensive upstream calls
- Need per-user daily quotas
- Must give consistent error UX across clients

Distilled from production patterns (SleepyCat's analyze-sleep-photo,
stylize-cat, etc.).

## Contents

- [Core conventions](#core-conventions)
- [Standard structure](#standard-structure)
- [Error code table](#error-code-table)
- [Quota table pattern](#quota-table-pattern)
- [Photo path RLS check](#photo-path-rls-check)
- [AI prompt requirements (i18n)](#ai-prompt-requirements-i18n)
- [Checklist before shipping](#checklist-before-shipping)

## Core conventions

1. **`verify_jwt = true`** in `supabase/config.toml` — all EFs verify JWT.
   No auth = 401. No anon access.
2. **CORS `*`** — Web PWA and iOS both call. Always handle OPTIONS preflight.
3. **EF never writes DB** — pure compute / API gateway. Clients receive the
   response and decide what to persist. Keeps EFs simple, retryable, fast.
4. **Errors are normalized JSON** — `{ error: "code", ... }`, HTTP status
   matches the code semantically.
5. **RLS-aware path checks** — if EF accepts `photo_path` or similar
   user-scoped paths, must enforce path prefix matches `auth.uid()`.
6. **AI failure doesn't burn quota** — only count successful calls.
7. **Response shape is always normalized** — regardless of upstream AI's
   weird outputs, server unifies the structure.

## Standard structure

```ts
// supabase/functions/<name>/index.ts
import { createClient } from "https://esm.sh/@supabase/supabase-js";

const CORS = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Methods": "POST, OPTIONS",
  "Access-Control-Allow-Headers": "authorization, apikey, content-type",
};

Deno.serve(async (req) => {
  // 1. CORS preflight
  if (req.method === "OPTIONS") {
    return new Response(null, { headers: CORS });
  }

  // 2. Auth check
  const authClient = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_ANON_KEY")!,
    {
      global: {
        headers: { Authorization: req.headers.get("Authorization") ?? "" },
      },
    }
  );
  const { data: { user } } = await authClient.auth.getUser();
  if (!user) return jsonError(401, "unauthorized");

  // 3. Parse body + validate
  const body = await req.json().catch(() => null);
  if (!body) return jsonError(400, "invalid_body");

  // 4. RLS path check (if applicable)
  if (body.photo_path && !body.photo_path.startsWith(`${user.id}/`)) {
    return jsonError(403, "forbidden");
  }

  // 5. Quota check
  const quotaResult = await checkAndIncrementQuota(user.id, "<feature_key>");
  if (!quotaResult.allowed) {
    return jsonError(429, "daily_limit_exceeded", { 
      reset_at: quotaResult.resetAt 
    });
  }

  // 6. Main work
  try {
    const result = await doTheWork(body);
    return new Response(JSON.stringify(normalize(result)), {
      status: 200,
      headers: { ...CORS, "Content-Type": "application/json" },
    });
  } catch (e) {
    // Map upstream errors to standard codes
    if (isAiFailure(e)) return jsonError(502, "ai_failed");
    if (isRateLimited(e)) return jsonError(429, "ai_rate_limited");
    if (isCreditsExhausted(e)) return jsonError(402, "ai_credits_exhausted");
    if (e.code === "invalid_base64") return jsonError(400, "invalid_base64");
    
    // Roll back quota on failure (caller didn't get what they paid for)
    await rollbackQuota(user.id, "<feature_key>");
    
    return jsonError(500, "internal", { message: String(e) });
  }
});

function jsonError(status: number, code: string, extra: object = {}) {
  return new Response(JSON.stringify({ error: code, ...extra }), {
    status,
    headers: { ...CORS, "Content-Type": "application/json" },
  });
}
```

## Error code table

Stick to this table. Adding new codes is OK but reuse when possible.

| HTTP | `error` code | When | Client UX hint |
|---|---|---|---|
| 401 | `unauthorized` | JWT missing / invalid | Redirect to login |
| 403 | `forbidden` | RLS violation (wrong user's path) | "You don't have permission" |
| 400 | `missing_<field>` | Required field absent | "Please provide <field>" |
| 400 | `invalid_<field>` | Enum / format wrong | "<field> is invalid" |
| 400 | `invalid_base64` | Base64 too big / unparseable | "Image too large, try smaller" |
| 400 | `invalid_body` | JSON parse failed | dev bug — fix client |
| 429 | `daily_limit_exceeded` | **Our** per-user daily quota | "Daily limit hit, resets at {reset_at}" |
| 429 | `ai_rate_limited` | **Upstream** AI rate limit (transient) | "AI is busy, try again in a moment" |
| 402 | `ai_credits_exhausted` | Service account out of credits | "Service temporarily unavailable" |
| 502 | `ai_failed` | Upstream returned weird/5xx | "Couldn't process, please retry" |
| 500 | `internal` | Anything else | "Unknown error, please retry" |

**Critical distinction**: `daily_limit_exceeded` and `ai_rate_limited` are
**both 429** but mean different things:

| Code | Wait | Reset time |
|---|---|---|
| `daily_limit_exceeded` | Yes, hours | Tomorrow at midnight (local TZ) |
| `ai_rate_limited` | No, seconds | Retry immediately works |

Client UI must distinguish. The `reset_at` payload field tells clients
when daily limit resets (ISO 8601 UTC).

## Quota table pattern

Shared table for all AI features:

```sql
CREATE TABLE daily_ai_usage (
  user_id uuid REFERENCES auth.users NOT NULL,
  day date NOT NULL,           -- in your TZ, e.g. Asia/Taipei
  count int DEFAULT 0,         -- feature 1
  stylize_count int DEFAULT 0, -- feature 2
  -- add new columns per new EF
  PRIMARY KEY (user_id, day)
);

ALTER TABLE daily_ai_usage ENABLE ROW LEVEL SECURITY;

-- Users can read their own usage (for "remaining quota" UI)
CREATE POLICY "own_usage_select" ON daily_ai_usage 
  FOR SELECT USING (auth.uid() = user_id);

-- Only service role writes (Edge Functions use service_role_key)
```

### Day boundary calculation

If your users are in one timezone, use that TZ for the "day" cutoff:

```ts
// Asia/Taipei midnight as the day boundary
function todayInTaipei(): string {
  const now = new Date();
  // Add 8 hours to UTC to get Taipei time, then format as YYYY-MM-DD
  const taipei = new Date(now.getTime() + 8 * 60 * 60 * 1000);
  return taipei.toISOString().slice(0, 10);
}
```

If multi-region, fall back to UTC.

### Incrementing logic

```ts
async function checkAndIncrementQuota(
  userId: string, 
  feature: string,
  limit: number
): Promise<{allowed: boolean, resetAt?: string}> {
  const day = todayInTaipei();
  
  // Upsert + atomic increment
  const { data, error } = await admin.rpc("increment_quota", {
    p_user_id: userId,
    p_day: day,
    p_feature: feature,
  });
  
  if (data.count > limit) {
    // Roll back the increment that pushed us over
    await admin.rpc("decrement_quota", { ... });
    return { 
      allowed: false, 
      resetAt: tomorrowMidnightTaipei() 
    };
  }
  
  return { allowed: true };
}
```

You can also use a check-first approach but atomic increment + rollback is
safer under concurrent calls.

## Photo path RLS check

When EF receives a `photo_path` referencing storage:

```ts
// Path format: "{user_id}/{uuid}.jpg"
if (!body.photo_path.startsWith(`${user.id}/`)) {
  return jsonError(403, "forbidden");
}
```

This prevents users from passing other users' paths to your EF (since
your EF runs with `service_role_key` and CAN read others' files).

Matching storage RLS:

```sql
-- bucket-level RLS for "photos" bucket
CREATE POLICY "own_photos" ON storage.objects FOR ALL 
  USING (bucket_id = 'photos' AND (storage.foldername(name))[1] = auth.uid()::text);
```

## AI prompt requirements (i18n)

If your AI returns text that goes to UI, plan for localization from day 1:

### Option A: Hardcode one language

```ts
// Works for single-market app
const prompt = `Reply in Traditional Chinese (Taiwan).
Avoid simplified or mainland terms.
Use 「軟體」not 「软件」, 「資料」not 「数据」, ...`;
```

**Pro**: Simple. **Con**: Re-deploying EF for i18n later.

### Option B: Take locale param

```ts
// Future-proof: client passes locale
const locale = body.locale ?? "zh-TW";
const prompt = `Reply in ${LOCALE_NAMES[locale]}. ${LOCALE_RULES[locale]}`;
```

**Pro**: Multi-locale without redeployment. **Con**: More upfront work.

### Option C: Return enum keys, translate client-side

```ts
// AI returns: { posture: "side_lying_belly_up" }
// Client maps to UI string per locale
```

**Pro**: Most flexible, AI outputs more stable. **Con**: Maintain enum
table.

Recommendation: **start with A**, plan migration to **C** when going
international.

## Checklist before shipping

- [ ] `supabase/config.toml` has `verify_jwt = true`
- [ ] CORS OPTIONS handler in place
- [ ] Photo / path RLS check (if applicable)
- [ ] Quota check + decrement on failure
- [ ] All errors return normalized JSON shape
- [ ] HTTP status matches `error` code semantically
- [ ] Wrote a client-facing spec doc (`docs/<endpoint>-spec.md`)
- [ ] Tested 3 happy paths via curl
- [ ] Tested 3 error paths via curl
- [ ] Web client + iOS / Android client both implemented per spec
- [ ] Logging includes user_id (for debugging) but no PII / image content

## Why EF never writes DB

Important design choice: **EF is pure "look at the input, return result".
Client decides what to persist.**

Benefits:
- Client can preview → user adjusts → THEN insert (better UX)
- EF logic stays simple, retry-safe
- EF perf not tied to DB latency  
- Schema changes don't require EF redeploys (only client changes)

Example (cat sleep tracking):
```
1. Client uploads photo to bucket
2. Client POSTs analyze-sleep-photo → EF returns { cat_id, room_id, posture }
3. Client shows confirmation UI; user can change cat / room / posture
4. Client INSERTs sleep_logs with ai_raw = <full EF response>
```

EF never INSERTs anything. Client orchestrates persistence.

## Companion skills

- `lovable-vs-claude-code-allocation` — why EFs should be CC's job, not LV's
- `cross-platform-dual-prompt-conventions` — sharing EF contracts across clients
