---
name: expo-lovable-asc-launch-workflow
description: "Use when planning the full App Store launch workflow for a React Native / Expo iOS app where the corresponding web version is built in Lovable, when integrating EAS Build with App Store Connect, when orchestrating TestFlight beta → production submission via CLI, or when deciding which tool (Lovable / Claude Code / asc / Expo / Xcode) handles each launch phase. Covers the 5-phase launch pipeline (build / test / metadata / submit / monitor), which existing AI skill repos to reach for at each phase (asc-skills, swift-ios-skills, our own), pre-launch checklists specific to Expo apps (PrivacyInfo, ATT, EAS Build), and the handoff points between Lovable-built marketing site and the actual iOS app."
---

# Expo + Lovable + asc CLI — iOS Launch Workflow

This is an **integration skill** — it doesn't replace any specific tool's
documentation. It maps "what tool to use when" across the **5 phases** of
shipping an Expo iOS app to the App Store, alongside a Lovable-built web
app sharing the same backend.

## Contents

- [The stack at a glance](#the-stack-at-a-glance)
- [Phase 1: Build (EAS)](#phase-1-build-eas)
- [Phase 2: Metadata & assets (asc skills)](#phase-2-metadata--assets-asc-skills)
- [Phase 3: TestFlight (asc + EAS Submit)](#phase-3-testflight-asc--eas-submit)
- [Phase 4: Production submission](#phase-4-production-submission)
- [Phase 5: Monitoring & ASO iteration](#phase-5-monitoring--aso-iteration)
- [Pre-launch checklist](#pre-launch-checklist)
- [Companion resources](#companion-resources)

## The stack at a glance

```
┌──────────────────┐         ┌──────────────────┐
│  Lovable (Web)   │         │  Expo (iOS)      │
│  cozy-cat-naps   │         │  SleepyCat/ios   │
└────────┬─────────┘         └────────┬─────────┘
         │                            │
         └──────────┬─────────────────┘
                    ▼
            ┌──────────────┐
            │  Supabase    │
            │  (Shared)    │
            └──────────────┘

         ┌──────────────────────────┐
         │  Claude Code (Orchestrator) │
         │  Reads SKILL.md from:    │
         │  • lovable-claude-code-skills  ←─ this repo
         │  • app-store-connect-cli-skills │
         │  • swift-ios-skills (concepts) │
         └──────────────┬───────────────┘
                        │
              ┌─────────┴──────────┐
              ▼                    ▼
        ┌──────────┐         ┌──────────────┐
        │   EAS    │         │   asc CLI    │
        │  Build   │         │  (Go binary) │
        └────┬─────┘         └──────┬───────┘
             │                      │
             └──────────┬───────────┘
                        ▼
              ┌────────────────────┐
              │ App Store Connect  │
              └────────────────────┘
```

## Phase 1: Build (EAS)

Tool: **Expo EAS Build** (cloud).

Claude Code's job: orchestrate `eas build`, manage versioning, handle native
config plugins.

```bash
# Increment build number
eas build:configure
eas build --platform ios --profile production
```

Configure in `app.json` / `eas.json`:
- `ios.bundleIdentifier`
- `ios.buildNumber` (auto-incrementing)
- `ios.config.usesNonExemptEncryption: false`
- Privacy manifest entries (declared APIs)

**Lovable's role**: none. iOS builds don't go through LV.

**Claude Code skill to use**: write your own `eas-build-workflow` skill
(not yet in this repo — contributions welcome).

## Phase 2: Metadata & assets (asc skills)

Tool: **asc CLI + asc-skills** ([rorkai/app-store-connect-cli-skills](https://github.com/rorkai/app-store-connect-cli-skills))

Key skills from that repo to install when ready:
- `asc-metadata-sync` — sync `metadata/` git folder ↔ App Store Connect
- `asc-localize-metadata` — multi-language metadata
- `asc-screenshot-resize` — auto-resize screenshots to all device sizes
- `asc-shots-pipeline` — screenshot generation pipeline
- `asc-whats-new-writer` — generate release notes from commit history
- `asc-aso-audit` — keyword and conversion audit

### Recommended metadata folder structure

```
SleepyCat/ios/metadata/
├── en-US/
│   ├── name.txt
│   ├── subtitle.txt
│   ├── description.txt
│   ├── promotional_text.txt
│   ├── keywords.txt
│   ├── support_url.txt
│   └── privacy_url.txt
├── zh-Hant/
│   └── ... same files
└── screenshots/
    ├── iPhone-6.5/
    └── iPhone-6.7/
```

Then:
```bash
asc metadata push --app-id <your-app-id>
```

Metadata is **version-controlled in git**. No more "who changed the
description last and when".

## Phase 3: TestFlight (asc + EAS Submit)

Two paths to TestFlight:

### Path A: EAS Submit (simplest)

```bash
eas submit --platform ios --latest
```

Uploads the build, App Store Connect processes it (10-30 min), then
manually configure beta testers in ASC web UI.

### Path B: asc CLI (more automation)

```bash
# Upload .ipa
asc builds upload --file ./build.ipa --app-id <id>

# Add testers programmatically
asc testflight testers add \
  --emails alice@example.com,bob@example.com \
  --group "External Beta"

# Submit for beta review (External testers need Apple beta review)
asc testflight submit-for-review --build-id <build-id>

# Pre-fill "What to Test" notes
asc testflight notes set --build-id <id> --notes-file ./testflight-notes.md
```

**Companion skill**: `asc-testflight-orchestration`

## Phase 4: Production submission

Tool: **asc CLI** with submission health check first.

```bash
# Pre-submission audit (catches 80% of rejection reasons)
asc submission health --app-id <id>

# Common issues this catches:
# - Missing privacy manifest entries
# - Missing localized metadata
# - Screenshots wrong dimensions
# - Missing app icon at required sizes
# - PrivacyInfo.xcprivacy declared APIs not match actual code
# - Subscriptions without localized descriptions

# Fix what it flags, then:
asc release submit --app-id <id> --build-id <id>
```

**Companion skill**: `asc-submission-health` + `asc-release-flow`

### Top rejection reasons for Expo apps

| Reason | Frequency | Fix |
|---|---|---|
| Privacy manifest missing API reasons | High | Add to `app.json` privacy plugin |
| ATT prompt before purpose visible | Medium | `expo-tracking-transparency` with clear UX |
| Subscription terms not in metadata | Medium | Add to App Description + EULA URL |
| Login required without way to delete account | High | **Apple requires in-app account deletion (Guideline 5.1.1)** |
| Empty / broken app on first launch | Low | Always have onboarding / demo data |
| Sign in with Apple absent when other social logins present | High | Add `expo-apple-authentication` |

The **account deletion** requirement bites everyone. SleepyCat-style apps
with Google OAuth must implement in-app account deletion, not just sign-out.

## Phase 5: Monitoring & ASO iteration

Tools: **asc CLI for data**, **Claude Code routines** for periodic checks.

```bash
# Daily crash report
asc analytics crashes --since yesterday --output json

# Weekly ASO audit
asc aso audit --app-id <id> --format markdown > weekly-aso.md

# Subscription revenue summary
asc analytics subscriptions --period last-7-days
```

### Cloud routine ideas (using `claude-code-cloud-routines` skill from this repo)

| Routine | Cron | Action |
|---|---|---|
| Daily crash digest | `0 23 * * *` | `asc analytics crashes` → Notion page |
| Weekly ASO audit | `0 23 * * 0` | `asc aso audit` → markdown + Slack |
| Pre-release health check | `0 23 * * 4` | `asc submission health` if pending build |
| Monthly subscription report | `0 0 1 * *` | `asc analytics subscriptions` → email |

## Pre-launch checklist

### 6 weeks before submission

- [ ] App Store Connect app record created (`asc-app-create-ui`)
- [ ] Bundle ID provisioned
- [ ] EAS Build pipeline works (test build → ad-hoc install on your device)
- [ ] Privacy manifest (`PrivacyInfo.xcprivacy`) configured via Expo Config Plugin
- [ ] `app.json` declares all required APIs (e.g., `userDefaults`, `fileTimestamp`)
- [ ] Sign in with Apple added (required if other OAuth is present)

### 4 weeks before

- [ ] Metadata in `ios/metadata/<locale>/`
- [ ] App icons at all 14 required sizes (Expo can generate)
- [ ] Screenshots for at least 6.5" + 6.7" iPhone (use `asc-screenshot-resize` to fill rest)
- [ ] Privacy policy URL live
- [ ] Support URL live  
- [ ] EULA reviewed (if doing subscriptions)
- [ ] Test purchase flow in StoreKit testing (if doing IAP)

### 2 weeks before

- [ ] TestFlight beta (External — needs Apple beta review, ~24h)
- [ ] At least 5 testers reporting "looks good"
- [ ] **In-app account deletion** working and tested (top rejection cause)
- [ ] All ATT / push / camera / location permission strings localized
- [ ] All metadata localized for every supported language
- [ ] App Tracking Transparency prompt has clear purpose copy

### 1 week before

- [ ] Run `asc submission health --app-id <id>` — fix all warnings
- [ ] Re-record screenshots if UI changed in last 2 weeks
- [ ] "What's New" copy ready
- [ ] Hold for review at midnight your launch day Pacific time (Apple reviews in PT)

### Day of submission

- [ ] One last `asc submission health` run
- [ ] Submit via `asc release submit` or App Store Connect UI
- [ ] Watch status: `asc submission status --watch`

## Companion resources

### Other repos to know about

| Repo | What it gives you | When to install |
|---|---|---|
| [rorkai/App-Store-Connect-CLI](https://github.com/rorkai/App-Store-Connect-CLI) | `asc` CLI binary (Go) | Now: `brew install asc` |
| [rorkai/app-store-connect-cli-skills](https://github.com/rorkai/app-store-connect-cli-skills) | 22 AI skills for asc workflows | 4 weeks before submission |
| [dpearson2699/swift-ios-skills](https://github.com/dpearson2699/swift-ios-skills) | 84 native iOS skills | Read 4-5 concept-level skills (review/ASO/accessibility/localization), skip Swift-specific |

### Companion skills in THIS repo

- `lovable-handoff-prompts` — how to prompt LV for the Web app side
- `lovable-vs-claude-code-allocation` — which tasks need which tool
- `claude-code-cloud-routines` — automated daily/weekly checks via routine
- `cross-platform-dual-prompt-conventions` — keeping Web + iOS UX aligned
- `supabase-edge-function-patterns` — backend that both clients call

## Tips from production

1. **`asc submission health` is your best friend** — run it weekly during
   final 4 weeks. It catches 80% of rejection causes.

2. **Account deletion is non-optional** — Apple started enforcing
   Guideline 5.1.1(v) aggressively in 2022. If you have user accounts,
   you MUST have in-app deletion.

3. **Sign in with Apple is non-optional if you have OAuth** — Guideline
   4.8. Add `expo-apple-authentication` even if you don't think users
   want it.

4. **Privacy manifest is non-optional since iOS 17** — Expo's privacy
   plugin handles most of this, but verify your `app.json` declares all
   APIs your dependencies use. Run `eas build --profile production` and
   check the generated `PrivacyInfo.xcprivacy`.

5. **Apple review hates "demo accounts" without real data** — provide a
   real demo account in the review notes, populated with actual data
   that lets the reviewer see core features in 30 seconds.

6. **Screenshots are 70% of conversion** — invest in 5-10 polished
   screenshots, A/B test via Custom Product Pages later.

7. **Submit Tuesday-Thursday at 5pm Pacific** — empirically faster
   review than Friday submissions (which sit over weekend).

## What this skill deliberately doesn't cover

- **Detailed asc commands** — see `asc --help` or asc-skills repo
- **Detailed Swift APIs** — see swift-ios-skills (mostly irrelevant for Expo anyway)
- **Lovable prompts** — see `lovable-handoff-prompts` in this repo
- **EAS specifics** — see Expo docs / write your own `eas-build-workflow` skill

This skill is the **map**, not the territory.
