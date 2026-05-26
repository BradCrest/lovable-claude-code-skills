---
name: cross-platform-dual-prompt-conventions
description: "Use when a feature must ship on both a web app AND a mobile app (e.g. Lovable-managed PWA + Expo/React Native iOS) that share a single Supabase backend, when UI copy or UX needs to stay perfectly aligned across platforms, when backend schema changes affect both clients, or when writing prompts for two different AI agents (one for Web, one for mobile). Covers three rules — docs as source of truth, dual prompts for UI copy, backend-first changes — plus prompt templates, when NOT to dual-prompt (visual polish, native APIs), and verification via scheduled audits."
---

# Cross-Platform Dual-Prompt Conventions

For teams shipping the same product on multiple clients (Web + iOS, or
Web + Android, etc.) where:
- One backend (e.g. Supabase) is shared
- One AI builds the Web app (e.g. Lovable)
- Another AI builds the mobile app (e.g. Claude Code with Expo)
- The two AIs don't see each other's conversation

How do you keep them in sync? **Three rules**, written down.

## Contents

- [The three rules](#the-three-rules)
- [Prompt templates](#prompt-templates)
- [When NOT to dual-prompt](#when-not-to-dual-prompt)
- [Verification](#verification)
- [Example: rooms management feature](#example-rooms-management-feature)

## The three rules

### Rule 1 — `docs/` is the shared source of truth

Anything implemented on both platforms gets a spec in `docs/feature-X.md`.

```
project-root/
├── web-app/                     ← Lovable-managed
├── mobile-app/                  ← Expo / Claude Code
└── docs/
    ├── edge-function-spec.md    ← Backend API contract
    ├── feature-rooms.md         ← Shared spec
    ├── feature-profile.md       ← Shared spec
    └── conventions.md           ← This whole rulebook
```

Both AIs read from `docs/`. Neither rewrites these files casually.

### Rule 2 — UI copy: write **paired prompts**, content identical

When you want a button text changed, you write **two prompts** — one for
each AI — using the **exact same copy strings**.

```markdown
# Both prompts contain this table:

| Location | String |
|---|---|
| Save button | 儲存 |
| Cancel button | 取消 |
| Success toast | 已儲存 |
| Error toast | 儲存失敗，請重試 |

"一字不差" / "verbatim". 
```

Copying the same table to both ensures no drift.

### Rule 3 — Backend changes: update `docs/` FIRST, code AFTER

Order matters:

1. **First**: edit `docs/feature-X.md` or `docs/<endpoint>-spec.md`
2. **Second**: write the migration / Edge Function code
3. **Third**: tell each client AI "see the updated spec, implement client side"

Why: if you change schema before updating docs, both clients drift from
each other and from the spec. Future you / new contributors have no
reliable source to consult.

## Prompt templates

### To Lovable (Web)

```markdown
請按照 docs/feature-<name>.md 實作 <feature>。
Repo: <web-repo>

## 範圍
- <change 1, with file path>
- <change 2, with file path>

## 文案
照 docs 內表格一字不差。

## 注意
- 與 <mobile platform> 共用同一個 Supabase
- 不要改 schema（除非 docs 標記要改）
- 與 <mobile platform> 對齊：另一邊會做一樣的版型 + 文案
```

### To Claude Code (Mobile / iOS / Expo)

```markdown
請按照 <project>/docs/feature-<name>.md 實作 <feature>。
Repo: <mobile-repo>

## 範圍
- <change 1, with file path>
- <change 2, with file path>

## 文案
照 docs 內表格一字不差。

## 注意
- 與 Web 共用同一個 Supabase；只當 client 呼叫，不要動 backend
- 與 Web 對齊：Web 已實作 / 將實作一樣的版型 + 文案
- <platform-specific tips, e.g. expo-image-picker permissions>
```

The two prompts share 80% structure and 100% of the copy table.

## When NOT to dual-prompt

Some things genuinely diverge per platform. Don't force these:

| Concern | Why it diverges |
|---|---|
| Pure visual polish (animations, micro-interactions) | Tailwind ≠ RN StyleSheet, different idioms |
| Platform-native APIs (Haptics, Share, Widget, PWA install) | Each platform has its own |
| Performance optimization | Architecture-specific (Hermes vs JS engine, RN vs Vite, etc.) |
| Dev tooling (testing, linting) | Each platform picks its own stack |
| Build / deploy pipeline | EAS for Expo, Vercel/Cloudflare for Web |

For these, write **single-platform** prompts. Don't try to dual-target.

## Verification

How do you know dual-prompting is working? Set up a **weekly audit
routine** (see `claude-code-cloud-routines`):

```
Every Monday 9am:
- Compare web routes vs mobile screens
- Check docs/feature-*.md implementation status on both sides
- Sample 5 UI strings, verify identical
- Output report + Notion page
```

If audit shows drift → fix it that week.

Example audit output:

```markdown
## Coverage matrix
| Feature | Web | Mobile | Spec |
|---|---|---|---|
| Login | ✅ | ✅ | — |
| Cat list | ✅ | ✅ | feature-cats.md |
| Rooms management | ✅ | ❌ | feature-rooms.md |
| Stylize cat photo | ✅ | ❌ | feature-stylize.md |
| ... | | | |

## Copy drift
- ❌ "儲存" (web) vs "保存" (mobile) — fix mobile
- ✅ All other sampled strings match
```

## Example: rooms management feature

### Step 1: Write spec

`docs/feature-rooms-management.md`:

```markdown
# Rooms Management

Page where users see / edit / delete / merge their rooms.

## Layout
[Header] [Plus button]
[Hint bar] 💡 重複的房間可以用「合併」整併在一起。
[Room list] each with [⋯ menu] → Edit / Merge to other room / Delete
[Add room button]

## Copy
| Location | String |
|---|---|
| Hint | 💡 重複的房間可以用「合併」整併在一起。 |
| Merge dialog title | 合併房間 |
| Merge dialog body | 合併「A」到「B」? N 筆紀錄將移過去。 |
| Delete confirm | 刪除「A」？相關紀錄將失去地點資訊。 |

## Acceptance
- [ ] List displays all user's rooms
- [ ] Merge transfers sleep_logs.room_id from source to target
- [ ] Merge deletes source room after transfer
- [ ] Only-one-room case: merge option shows "no merge target"
```

### Step 2: Web prompt (to Lovable)

```markdown
請按照 docs/feature-rooms-management.md 實作。
Repo: cozy-cat-naps

## 範圍
1. src/routes/rooms.tsx (new) — 主列表頁
2. 每個列項加 ⋯ DropdownMenu (編輯/合併/刪除)
3. Merge sheet + AlertDialog
4. Settings 頁加「房間管理」入口

## 文案
照 docs 內表格一字不差。

## 注意
- 與 iOS 端共用同一個 Supabase
- 與 iOS 對齊：iOS 也會做一樣版型 + 文案
```

### Step 3: Mobile prompt (to Claude Code iOS)

```markdown
請按照 SleepyCat/docs/feature-rooms-management.md 在 ios/ 實作。
Repo: SleepyCat/ios (Expo)

## 範圍
1. app/(settings)/rooms.tsx (new) — RoomsScreen
2. 每個列項加 contextual menu (編輯/合併/刪除)
3. Merge bottom sheet + Alert dialog
4. Settings 加 "房間管理" row

## 文案
照 docs 內表格一字不差。

## 注意
- 與 Web 共用同一個 Supabase；不要動 backend
- 與 Web 對齊：Web 已實作一樣版型 + 文案
- iOS 用 expo-router 命名 慣例 (app/(settings)/...)
```

Two prompts. Same spec. Same copy table.

Result: identical UX on both platforms, week 1.

## Common pitfalls

| Pitfall | Fix |
|---|---|
| Wrote prompt to Web but forgot Mobile | Always write both at the same session |
| Copy table in only one prompt | Copy-paste the same table to both |
| One platform implements an extra "improvement" not in spec | Audit catches it, refactor to remove |
| Spec gets stale because devs only update code | Routine compares docs vs code, flags drift |
| New schema column added by Web AI without updating spec | Rule 3: backend changes need spec update first |

## Companion skills

- `lovable-feature-spec-pattern` — how to write the spec
- `lovable-handoff-prompts` — how to format the Web prompt
- `claude-code-cloud-routines` — automated drift audit
