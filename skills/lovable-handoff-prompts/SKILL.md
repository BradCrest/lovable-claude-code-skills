---
name: lovable-handoff-prompts
description: "Use when writing prompts to give to Lovable (lovable.dev) for app changes, when an existing LV-generated app needs new features, when LV is misinterpreting your intent, when handoff between you (or Claude Code) and LV needs to be clean, or when you need LV to follow project-specific conventions. Covers vocabulary that LV reads reliably vs misreads, structure for multi-task prompts, how to anchor LV to existing files / specs, when to break a big task into multiple LV prompts, and avoiding common LV failure modes like over-scoping or schema drift."
---

# Lovable Handoff Prompts

Lovable (lovable.dev) is an AI app builder. Its agent reads your prompt,
makes changes across many files, and pushes a commit. The quality of the
result depends massively on prompt structure.

This skill covers patterns that **make LV ship the right thing first try**
and **avoid the common failure modes**.

## Contents

- [Anatomy of a good LV prompt](#anatomy-of-a-good-lv-prompt)
- [Vocabulary that works](#vocabulary-that-works)
- [Vocabulary that misfires](#vocabulary-that-misfires)
- [Anchoring LV to existing files](#anchoring-lv-to-existing-files)
- [Breaking big tasks](#breaking-big-tasks)
- [Common failure modes](#common-failure-modes)

## Anatomy of a good LV prompt

```markdown
請按照 docs/feature-<name>.md 實作 <feature>。
（or: Please implement <feature> per docs/feature-<name>.md.）

## 範圍 / Scope

1. **<concrete thing 1>**
   - 檔案：`<exact file path>`
   - 改什麼：<one-line description>
2. **<concrete thing 2>**
   - ...

## 文案 / Copy

| 位置 | 文案 |
|---|---|
| 某處 | 某句話（一字不差）|

## 注意 / Constraints

- 不要改 backend schema（已在 docs 內標記）
- 與 iOS 端對齊（共用 Supabase）
- 沿用既有 component（不要重新發明）

## 驗收 / Acceptance

- [ ] <thing 1 works>
- [ ] <thing 2 works>
```

The 4-section structure (scope / copy / constraints / acceptance) is the
single most reliable LV pattern. LV's agent reads sequentially and the
section headers act as guard rails.

## Vocabulary that works

| Phrase | Why it works |
|---|---|
| "按照 docs/foo.md 實作" / "implement per docs/foo.md" | LV reads referenced file, anchors to that spec |
| "一字不差" / "verbatim" / "exactly as written" | Stops LV from rephrasing copy |
| "沿用既有 X" / "reuse existing X" | Prevents reinventing components |
| "不要改 schema" / "don't modify schema" | Holds the backend line |
| "檔案: `src/...`" / "file: `src/...`" | Direct file pointer beats vague description |
| "**對齊** iOS / **align** with iOS" | Hints at the dual-platform constraint |
| "minimum viable" / "最簡可行" | Caps scope creep |
| "如果不存在請建" / "create if missing" | Handles idempotent migrations |

## Vocabulary that misfires

| Phrase | What LV does instead | Fix |
|---|---|---|
| "make it better" | Wholesale rewrites random files | Be concrete: "improve <X> by doing <Y>" |
| "fix the bug" | Touches unrelated code | Quote the error message + line number |
| "modern look" / "更現代" | Changes design tokens you didn't want changed | Reference existing styles to preserve |
| "responsive" without breakpoints | Inserts random media queries | Specify breakpoints |
| "add tests" | Generates shallow / wrong tests | Skip — testing belongs in Claude Code |
| "best practices" | Refactors per generic advice, breaks app | Be specific about what changes |
| "while you're at it..." | Scope explodes | One task per prompt |

## Anchoring LV to existing files

LV agents are more accurate when you point them at concrete files:

```markdown
# Vague (LV guesses, often wrong)
"Update the cat list page to show photos"

# Anchored (LV edits the right file)
"In `src/routes/cats.tsx`, replace the icon-only list item with a 
SignedImage thumb (40px) when `photo_url` exists. Keep icon fallback. 
See existing pattern in `src/routes/rooms.tsx` line 135-148."
```

**Always** include:
- Exact file path
- A working example LV can mirror (line numbers if possible)
- What to keep unchanged

## Breaking big tasks

**Bad**: one giant prompt with 8 things ("add profile page + change tab bar
+ migrate avatars bucket + update theme + ...") — LV will do some, skip
some, break others.

**Good**: one prompt = one coherent change. Specifically:

| Size of change | Strategy |
|---|---|
| Single file edit | One LV prompt |
| 2-5 file feature | One LV prompt with clear sections |
| New page + new schema | One spec doc + one LV prompt referencing it |
| Refactor across many files | Don't use LV — use Claude Code |
| Multiple unrelated features | Multiple LV prompts, one per feature |

If you find yourself writing "and also...", split the prompt.

## Common failure modes

### 1. LV reverts your Claude Code changes

LV starts from `main` HEAD. If you forgot to `git pull` before opening LV,
LV's "Changes" commit can undo your work.

**Prevention**: always `git pull --rebase` your local before working;
always check `git log -5` before assuming your code is still there.

See: `lovable-git-race-handling` skill.

### 2. LV invents files that don't exist

"Add a new `useTheme` hook" → LV creates `src/hooks/useTheme.ts` even
though `src/lib/theme.ts` already exists.

**Prevention**: explicitly say "the file already exists at `<path>`,
modify it" or "check `<path>` first, only create if missing".

### 3. LV changes UI copy slightly

You wrote: "💡 重複的房間可以用「合併」整併在一起。"
LV ships: "💡 重複的房間可以使用合併功能整併。"

**Prevention**: put copy in a markdown table inside the prompt + write
"一字不差" / "verbatim" + put the same copy in `docs/feature-*.md`.

### 4. LV touches things you didn't ask about

You asked for a button color change. LV "improved" the entire form layout.

**Prevention**: end every LV prompt with:
> 不要動其他檔案 / Do not modify other files.

### 5. LV invents schema columns

You asked for "add display_name to profiles". LV adds `display_name`,
`avatar_url`, `bio`, `notification_pref`, ...

**Prevention**: explicitly list every column. Or write the migration SQL
inside the prompt and say "use this exact migration".

## Cheat sheet — minimal viable LV prompt

```
請按照 docs/feature-<name>.md 實作以下：

1. <change 1 with file path>
2. <change 2 with file path>

文案：照 docs 內表格一字不差。

不要動其他檔案。
不要改 schema（除非 docs 標記要改）。
```

If your prompt isn't this disciplined, edit before sending.
