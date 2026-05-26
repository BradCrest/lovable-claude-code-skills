# Lovable + Claude Code Skills

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](LICENSE)
[![Agent Skills](https://img.shields.io/badge/Agent%20Skills-standard-green.svg?style=for-the-badge)](https://agentskills.io)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-compatible-d97757.svg?style=for-the-badge)](https://claude.ai/code)
[![GitHub stars](https://img.shields.io/github/stars/BradCrest/lovable-claude-code-skills?style=for-the-badge)](https://github.com/BradCrest/lovable-claude-code-skills/stargazers)

Agent skills for teams who use **Lovable** as their visual / AI app builder
and **Claude Code** (or Cursor / Codex / etc.) for deeper customization,
backend work, audits, and cross-platform expansion.

If you've ever asked yourself:

- "How do I split work between Lovable and Claude Code without stepping on each other?"
- "My LV-generated app needs a feature LV can't do — now what?"
- "How do I keep Web (LV-managed) and iOS (Expo) in sync when they share one Supabase backend?"
- "Lovable's auto-commit pushed over my Claude Code changes — how do I prevent this?"

...this repo is for you.

## What's an "agent skill"?

A folder with a `SKILL.md` file containing YAML frontmatter (`name` +
`description`) and markdown body. AI agents that follow the
[Agent Skills standard](https://agentskills.io) — Claude Code, Cursor,
Codex, Gemini, and 40+ others — read these and auto-apply them when the
`description` matches what you're doing.

So if you install this repo, your AI pair programmer **automatically knows**
the LV + Claude Code coordination patterns without you having to type them
into every prompt.

## When this WON'T help you

Be honest — this repo is for a **specific intersection**:

- ❌ **You don't use Lovable.** Most patterns assume LV as one of the two agents. If you only use Claude Code / Cursor / Codex, only ~2 skills (`supabase-edge-function-patterns`, `ai-image-sprite-pipeline`) are generally applicable.
- ❌ **You don't use Supabase.** Edge Function patterns, RLS path checks, quota tables, JWT verification — all Supabase-specific. Firebase / Convex / PlanetScale users will need to translate.
- ❌ **You ship on one platform only.** Half this repo is about Web + Mobile coordination. Single-platform teams won't need it.
- ❌ **You want generic prompt engineering advice.** This is about a specific 2-tool workflow, not LLM tricks.

If 2+ of those apply, you'll get more value from skills designed for your stack — star this repo if you're curious, but don't force-fit.

## Skills in this repo

### Lovable workflow

| Skill | When it triggers |
|---|---|
| [`lovable-handoff-prompts`](skills/lovable-handoff-prompts/) | Writing a prompt for Lovable, choosing words LV reads well |
| [`lovable-git-race-handling`](skills/lovable-git-race-handling/) | LV concurrent push collisions, `git pull --rebase` recovery |
| [`lovable-feature-spec-pattern`](skills/lovable-feature-spec-pattern/) | Using `docs/feature-*.md` as shared contract between LV and Claude Code |
| [`lovable-vs-claude-code-allocation`](skills/lovable-vs-claude-code-allocation/) | Deciding which tasks suit LV vs which suit Claude Code |

### Claude Code as the "second pair"

| Skill | When it triggers |
|---|---|
| [`claude-code-cloud-routines`](skills/claude-code-cloud-routines/) | Setting up scheduled remote agents to audit / extend LV apps |

### Cross-platform & backend patterns

| Skill | When it triggers |
|---|---|
| [`supabase-edge-function-patterns`](skills/supabase-edge-function-patterns/) | Creating / modifying Supabase Edge Functions, error codes, daily quotas |
| [`cross-platform-dual-prompt-conventions`](skills/cross-platform-dual-prompt-conventions/) | One Supabase backend, two clients (Web + RN/iOS), keeping them in sync |
| [`ai-image-sprite-pipeline`](skills/ai-image-sprite-pipeline/) | Scaling AI-generated UI assets that LV / Claude Code can't generate themselves |

### Launch / App Store

| Skill | When it triggers |
|---|---|
| [`expo-lovable-asc-launch-workflow`](skills/expo-lovable-asc-launch-workflow/) | Planning iOS App Store launch for an Expo app with companion Lovable web, 5-phase pipeline mapping which tool to use when, pre-launch checklist |

## Bilingual prompts (English / 中文 / 日本語 / 한국어)

Lovable's agent reads CJK languages well — Traditional Chinese, Simplified
Chinese, Japanese, and Korean prompts trigger structurally identical behavior
to English ones. Many real-world prompts in these skills use Chinese with
English code identifiers mixed in:

```markdown
請按照 docs/feature-rooms.md 實作 RoomsScreen。
不要動 schema。文案沿用 docs 內表格一字不差。
```

If you only ship in English, just translate the wrapper text — the structural
patterns are language-agnostic.

## Install

### Recommended: any agent via [skills CLI](https://github.com/vercel-labs/skills)

```bash
npx skills install BradCrest/lovable-claude-code-skills
```

### Claude Code (manual)

```bash
git clone https://github.com/BradCrest/lovable-claude-code-skills.git \
  ~/.claude/skills/lovable-claude-code
```

Claude Code reads `SKILL.md` frontmatter on session start.

### Cloud routines (Anthropic-hosted Claude Code agents)

Add to your routine's `sources` array:

```json
{
  "sources": [
    {"git_repository": {"url": "https://github.com/yourorg/your-app"}},
    {"git_repository": {"url": "https://github.com/BradCrest/lovable-claude-code-skills"}}
  ]
}
```

## Workflow this repo supports

```
┌─────────────────┐
│   You (PM/dev)  │
└────────┬────────┘
         │
    ┌────┴─────┐
    ▼          ▼
 ┌─────┐   ┌───────────┐
 │ LV  │   │ Claude    │
 │     │   │ Code      │
 └──┬──┘   └─────┬─────┘
    │            │
    ▼            ▼
  Web App    Backend / iOS / Audit
    │            │
    └─────┬──────┘
          ▼
    Same Supabase
```

Lovable builds the happy path fast. Claude Code does the things LV can't:
- New backend services (Edge Functions, migrations)
- iOS native via Expo
- Periodic audits, refactors, content pipelines
- Anything that needs git surgery beyond LV's "Changes" commits

## Contributing

PRs welcome! See [CONTRIBUTING.md](CONTRIBUTING.md).

The skills in this repo come from actual production experience shipping a
cat-sleep-tracking PWA + iOS app. When you have a war story from your LV
→ Claude Code workflow, please share.

### Skill description writing tips

The `description` field decides whether your skill gets auto-triggered.
**Be specific, list scenarios, write in English** (English triggers most
reliably across agents):

- ❌ `description: "Handles Lovable stuff"` — too vague, won't trigger
- ✅ `description: "Use when LV's concurrent git push causes 'Updates were rejected' errors, or when designing branch strategies to coexist with LV's auto-commits."` — specific scenarios, will trigger correctly

## License

MIT — use, fork, adapt freely. Attribution appreciated but not required.

## Related repos

This repo lives inside a larger tool ecosystem. Reach for these as you grow:

### App Store automation (essential for iOS launch)

- [rorkai/App-Store-Connect-CLI](https://github.com/rorkai/App-Store-Connect-CLI) — `asc` CLI binary in Go. Automates every App Store Connect operation (TestFlight, submissions, metadata, analytics, screenshots, IAP). Install via `brew install asc`.
- [rorkai/app-store-connect-cli-skills](https://github.com/rorkai/app-store-connect-cli-skills) — 22 AI agent skills built on top of `asc`. Covers app creation, metadata sync, screenshot pipeline, TestFlight orchestration, submission health, ASO audit, RevenueCat sync, crash triage.

When approaching iOS launch, install these alongside our `expo-lovable-asc-launch-workflow` skill (which is the integration map).

### Native iOS knowledge (for concepts, not code)

- [dpearson2699/swift-ios-skills](https://github.com/dpearson2699/swift-ios-skills) — 84 skills for native iOS 26+ / SwiftUI. Where this repo's structure was inspired from. **For Expo / React Native devs**: read the 4-5 platform-level skills (`app-store-review`, `app-store-optimization`, `ios-accessibility`, `ios-localization`, `push-notifications`) for concepts. Skip everything Swift-specific.

### Foundations

- [Agent Skills standard](https://agentskills.io) — the spec
- [Lovable](https://lovable.dev) — the AI app builder
- [Claude Code](https://claude.ai/code) — the AI coding agent + cloud routines

## Author

Built by [@BradCrest](https://github.com/BradCrest) while shipping a
cross-platform cat-sleep-tracking app. Many lessons learned the hard way —
hopefully you don't have to.
