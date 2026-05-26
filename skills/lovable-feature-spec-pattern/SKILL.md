---
name: lovable-feature-spec-pattern
description: "Use when designing a feature that needs to be implemented by both Lovable AND Claude Code (or other AI agents), when LV keeps drifting from your intent across multiple prompts, when handing off a feature from one AI tool to another, when iOS / Android / Web need feature parity, or when you want a stable contract that survives many LV regenerations. Covers writing docs/feature-*.md as a shared source-of-truth, structuring spec sections (scope, schema, copy, error states, acceptance), versioning specs over time, and the rule 'specs change before code changes'."
---

# Lovable Feature Spec Pattern

When LV and Claude Code (or any other AI agent) need to work on the same
feature, the **spec file is your contract**. Each agent reads from it,
implements its part, and the spec stays authoritative even when LV
regenerates code 10 times.

This skill teaches you to write `docs/feature-*.md` files that are:
- Detailed enough that LV can implement the right thing first try
- Clear enough that Claude Code can mirror it on iOS / backend
- Stable enough to survive many implementation iterations

## Contents

- [Why specs beat free-form prompts](#why-specs-beat-free-form-prompts)
- [The 6-section template](#the-6-section-template)
- [Worked example](#worked-example)
- [Spec versioning](#spec-versioning)
- [Common pitfalls](#common-pitfalls)

## Why specs beat free-form prompts

| Approach | Result |
|---|---|
| You type new prompt to LV every time | LV reinterprets; details drift |
| You type new prompt to Claude Code every time | Inconsistent with LV's version |
| Spec in `docs/feature-X.md` + both AIs reference it | Both stay aligned to the same source |

The spec **outlives** any individual prompt or AI session. Three months
from now when a new Claude session looks at this codebase, it'll read
the spec and understand the design intent — not have to reverse-engineer
from code.

## The 6-section template

```markdown
# Feature: <Name>

One-paragraph what & why. Who benefits. What it replaces or adds.

## 1. Scope (in / out)

**In scope** (this feature includes):
- ...

**Out of scope** (explicitly NOT this feature):
- ...

## 2. UI / UX

[Component name] — [where it appears]

### Layout

ASCII diagram or screenshot mockup.

### Interactions

| User action | Result |
|---|---|
| Tap X | Y happens |
| Swipe left | Z slides in |

### Empty / error / loading states

- Loading: skeleton placeholder
- Empty: <empty illustration> + <button>
- Error: toast + retry button

## 3. Copy (verbatim, no paraphrasing)

| Location | String |
|---|---|
| Header title | 編輯個人檔案 |
| Save button | 儲存 |
| Success toast | 已更新 |
| Error toast | 儲存失敗 |
| Confirm dialog | 確定刪除？此操作無法復原。 |

## 4. Data

### Schema changes

```sql
ALTER TABLE profiles ADD COLUMN IF NOT EXISTS avatar_url text;
```

### Queries

- `profiles` SELECT by `auth.uid()` (RLS handles)
- `profiles` UPDATE SET avatar_url, display_name

### Storage

- Bucket: `avatars`
- Path: `{user_id}/avatar.jpg`
- Image compression: 256px / quality 0.85

## 5. Error states

| Condition | UX |
|---|---|
| Network failure | Toast "網路錯誤，請稍後再試" + keep form open |
| Validation: name empty | Inline error "暱稱不可空白" |
| Upload too large | Toast "圖片過大，請壓縮後重試" |

## 6. Acceptance

- [ ] User can navigate to /profile from home avatar
- [ ] Edit sheet opens, pre-fills current values
- [ ] Save updates DB + closes sheet
- [ ] Upload replaces avatar, refresh shows new image
- [ ] Cache busting works (no stale image cache)
- [ ] Empty display_name shows fallback "還沒命名的飼主"

## Change history

- 2026-05-23: Initial spec
- 2026-05-30: Added cache busting per implementation review
```

## Worked example

When implementing a "profile page + nav restructure" feature:

1. Write `docs/feature-profile-and-nav.md` with all 6 sections.
2. Give LV this prompt:
   ```
   請按照 docs/feature-profile-and-nav.md 實作。
   Repo: cozy-cat-naps
   不要動 backend 以外的 schema。文案沿用 docs 內表格一字不差。
   ```
3. Give Claude Code (iOS session) this prompt:
   ```
   按照 <project>/docs/feature-profile-and-nav.md 在 ios/ 實作。
   只當 client，不要動 backend。文案一字不差。
   ```
4. Both AIs read the same source. Outputs align.

If LV botches it, you don't rewrite the prompt — you point at the spec
again with what went wrong:
```
LV did <X>, but spec section 3 says <Y>. Please re-implement that 
section per the spec.
```

## Spec versioning

Specs evolve. Keep a `## Change history` section at the bottom:

```markdown
## Change history

- 2026-05-23: Initial spec (sections 1-6)
- 2026-05-25: Added cache busting requirement (section 4)
- 2026-06-01: display_name max length 20 chars (section 5)
```

Why this matters:
- Implementations done before a change might lack the newer requirement
- Reviewers (or future-you) can tell if code matches latest spec
- If two AIs disagree, the change history shows who's behind

## Common pitfalls

### Pitfall: spec drift

You wrote the spec on day 1, but LV's actual implementation diverged on
day 4 (you didn't notice), and now the spec is fiction.

**Fix**: at end of each LV session, diff implementation vs spec. Update
either one to reconcile. Don't let drift accumulate.

### Pitfall: too vague to implement

Spec says: "Show a nice loading state."

**Fix**: be specific — "Skeleton placeholder using the existing
`<SkeletonText>` component, 3 rows, same height as final content."

### Pitfall: too detailed (implementation in disguise)

Spec dictates exact React component structure, prop names, internal state.

**Fix**: spec covers WHAT the user sees / experiences and WHAT data flows.
HOW (component structure) belongs in code.

### Pitfall: missing copy section

LV will invent UI strings and they'll drift across iterations.

**Fix**: always include copy section with table. Force LV to copy verbatim.

### Pitfall: no acceptance checklist

You can't tell if LV is done.

**Fix**: every spec ends with `## Acceptance` — explicit testable
conditions. Make Claude Code go through this checklist as a verification
step.

## Spec file naming

| Pattern | Use |
|---|---|
| `docs/feature-<name>.md` | New user-facing feature |
| `docs/<endpoint>-spec.md` | API contract (Edge Function, public API) |
| `docs/conventions.md` | Cross-cutting workflow rules |
| `docs/feature-<name>-mvp1.md` | Phased feature, MVP 1 |
| `docs/feature-<name>-mvp2.md` | Phased feature, MVP 2 (additions) |

## Companion skills

- `lovable-handoff-prompts` — how to write the prompt that references the spec
- `cross-platform-dual-prompt-conventions` — dual-platform spec strategy
