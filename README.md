# Lovable + Claude Code Skills

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Agent Skills](https://img.shields.io/badge/Agent%20Skills-standard-green.svg)](https://agentskills.io)

Agent skills for teams who use **Lovable** as their visual / AI app builder
and **Claude Code** (or Cursor / Codex / etc.) for deeper customization,
backend work, audits, and cross-platform expansion.

If you've ever asked yourself:

- 「Lovable 跟 Claude Code 一起用怎麼分工？」
- "My LV-generated app needs a feature LV can't do — now what?"
- "How do I keep Web (LV-managed) and iOS (Expo) in sync when they share a Supabase backend?"
- "Lovable pushed over my Claude Code changes — how do I prevent this?"

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
| [`ai-image-sprite-pipeline`](skills/ai-image-sprite-pipeline/) | Using Nano Banana / DALL-E / Midjourney to generate UI assets at scale |

### Launch / App Store

| Skill | When it triggers |
|---|---|
| [`expo-lovable-asc-launch-workflow`](skills/expo-lovable-asc-launch-workflow/) | Planning iOS App Store launch for an Expo app with companion Lovable web, 5-phase pipeline mapping which tool to use when, pre-launch checklist |

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

The skills in this repo come from actual production experience shipping
[SleepyCat](https://sleepin.cat) (a cat-sleep-tracking PWA + iOS app). When
you have a war story from your LV → Claude Code workflow, please share.

### Skill description writing tips

The `description` field decides whether your skill gets auto-triggered.
**Be specific, list scenarios, write in English** (English triggers most
reliably across agents):

- ❌ `description: "Handles Lovable stuff"` — too vague, won't trigger
- ✅ `description: "Use when LV's concurrent git push causes 'Updates were rejected' errors, when you need to recover after working on the same branch as LV, or when designing branch strategies to avoid race conditions with LV's auto-commits."` — specific, triggers on the right scenarios

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

Built by [@BradCrest](https://github.com/BradCrest) while shipping SleepyCat.
Many lessons learned the hard way — hopefully you don't have to.
