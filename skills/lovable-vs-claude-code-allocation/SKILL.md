---
name: lovable-vs-claude-code-allocation
description: "Use when deciding whether to make a change via Lovable or via Claude Code, when planning a multi-step feature and figuring out which AI does which step, when you're frustrated because LV can't do something, or when a teammate / new contributor asks 'should I use LV for this?'. Covers the decision matrix (UI vs backend vs cross-platform vs refactor), tasks LV does best, tasks Claude Code does best, hybrid workflows where they coordinate, and anti-patterns (e.g. asking LV to do a giant refactor)."
---

# Lovable vs Claude Code — Task Allocation

Lovable and Claude Code are both AI coding agents but their sweet spots
differ. Using the right tool for each task ships features 2-5× faster
than forcing the wrong one.

This skill is the **decision matrix** for "should I LV this or CC this?"

## Contents

- [The 30-second rule](#the-30-second-rule)
- [What Lovable is best at](#what-lovable-is-best-at)
- [What Claude Code is best at](#what-claude-code-is-best-at)
- [Decision matrix](#decision-matrix)
- [Hybrid workflows](#hybrid-workflows)
- [Anti-patterns](#anti-patterns)

## The 30-second rule

If the change is:

- **Visible UI change in the LV-managed app** → Lovable
- **Anything else** → Claude Code

Exceptions exist (see matrix below), but this gets you 80% right.

## What Lovable is best at

| Strength | Why |
|---|---|
| **UI iteration** | Visual feedback loop, sees the result instantly |
| **Page composition** | Knows the existing layouts, designs new ones consistently |
| **Theme + styling** | Tailwind / CSS variable systems are LV's home turf |
| **Adding a CRUD page** | Has templates for list / detail / form pages |
| **Connecting Supabase tables to UI** | Wires up queries automatically |
| **Form validation** | Standard patterns, knows the library |
| **Empty / loading / error states** | Has component library for these |

If you can describe it like:
- "Add a settings page with these options"
- "Make the cat list a grid"
- "Add a stats card showing N sleeps this week"

→ **Use LV.**

## What Claude Code is best at

| Strength | Why |
|---|---|
| **Refactoring across many files** | Reads codebase, plans, executes consistently |
| **Backend / Edge Functions** | LV writes basic ones, CC writes complex / debugged ones |
| **Schema migrations** | CC writes correct SQL, applies cleanly |
| **Native iOS / Android (Expo)** | LV is Web-only |
| **Testing** | CC writes meaningful tests; LV's tests are shallow |
| **CI / DevOps** | GitHub Actions, scripts, automation |
| **Debugging across the stack** | CC follows error → cause via logs / stack traces |
| **Reading entire codebase** | CC's `Read`/`Grep` is more thorough than LV's scanning |
| **Following docs / specs precisely** | CC obeys structured docs; LV improvises |
| **Cron jobs / scheduled tasks** | LV doesn't do these; CC does via Routine |
| **Cross-tool integrations (MCP)** | CC can talk to Notion / Linear / Slack |

If you can describe it like:
- "Audit which routes lack the new auth pattern"
- "Refactor all API calls to use the new SDK"
- "Add an Edge Function that does X with retries and error handling"
- "Set up a daily Notion sync of yesterday's metrics"

→ **Use Claude Code.**

## Decision matrix

| Task type | LV | CC | Notes |
|---|---|---|---|
| Add a page / route | ✅ | ⚠️ | LV faster, knows conventions |
| Add a form field | ✅ | ⚠️ | LV |
| Change button colors | ✅ | ❌ | LV |
| Refactor 20 files to use new API | ❌ | ✅ | CC |
| Add new Supabase Edge Function | ⚠️ | ✅ | CC. LV's EFs often skip auth / error handling |
| Modify existing EF | ❌ | ✅ | CC |
| Migration: add column | ⚠️ | ✅ | CC. LV forgets IF NOT EXISTS, RLS |
| Migration: drop / rename | ❌ | ✅ | CC |
| Setup i18n from scratch | ⚠️ | ✅ | CC. LV does setup but misses extraction patterns |
| Translate existing strings | ✅ | ⚠️ | LV |
| iOS / Android (Expo) work | ❌ | ✅ | CC. LV is Web-only |
| Write Playwright tests | ❌ | ✅ | CC |
| Add Sentry / PostHog | ⚠️ | ✅ | CC. LV adds SDK but misses error boundaries |
| Update routeTree.gen.ts cache | ❌ | ✅ | CC. LV breaks this regularly |
| Document codebase | ❌ | ✅ | CC |
| Setup cron jobs | ❌ | ✅ | CC via cloud routines |
| Generate sprites / assets via AI | ❌ | ⚠️ | Neither directly; you + manual + CC for integration |
| Audit / inventory work | ❌ | ✅ | CC |

Legend: ✅ great fit, ⚠️ acceptable but suboptimal, ❌ avoid

## Hybrid workflows

The biggest wins come from **using both, in sequence**.

### Workflow 1: Spec → LV → CC audit

```
1. Write docs/feature-X.md (you + maybe CC)
2. LV: implement per spec  
3. CC: audit implementation vs spec, fix gaps
4. CC: write tests
5. Optional: CC implements iOS version
```

### Workflow 2: LV draft → CC polish

```
1. LV: rapid prototype, get UI + flow working
2. You ship internally, gather feedback
3. CC: refactor / harden / add missing edge cases
4. CC: write proper tests now that flow is stable
```

### Workflow 3: CC backend → LV UI

```
1. CC: design + implement Edge Function with proper error codes
2. CC: write docs/feature-X.md including the API contract
3. LV: implement UI consuming that EF (referencing the spec)
```

### Workflow 4: LV builds, CC schedules

```
1. LV: build the app
2. CC: set up cloud routines that periodically:
   - Audit code quality
   - Generate reports
   - Sync to Notion / Slack
   - Run health checks
```

## Anti-patterns

### Anti-pattern 1: "LV, refactor the whole codebase"

LV's strength is forward construction, not transformation. Asking it to
refactor → it rewrites things you didn't want changed.

**Use**: Claude Code for refactors.

### Anti-pattern 2: "Claude Code, add a fancy animation to this button"

CC can do it but you'll spend more time describing the animation than LV
would spend building it. Visual design = LV's home turf.

**Use**: LV for visual polish.

### Anti-pattern 3: "Both AIs working on the same files simultaneously"

You're guaranteed to hit git conflicts.

**Use**: divide files / pages cleanly. LV gets `src/routes/`, CC gets
`supabase/functions/` and `docs/`. Don't overlap.

### Anti-pattern 4: "LV does the new Edge Function"

LV's EFs typically:
- Skip CORS preflight
- Forget JWT verification
- No proper error codes
- No quota tracking
- Hardcode timeout values

**Use**: CC for new Edge Functions. See `supabase-edge-function-patterns`.

### Anti-pattern 5: "LV adds tests"

LV's tests are shallow ("test that the page renders without error"). Real
bugs slip through.

**Use**: CC for tests. Skip tests in LV phase entirely.

## Cost considerations

| Tool | Cost shape |
|---|---|
| Lovable | Monthly subscription, included usage |
| Claude Code | Per-token API or Anthropic plan |
| Both | If teammate uses LV + you use CC, parallel work doubles velocity |

For solo founders: usually LV for product, CC for ops + content + audits.

## Companion skills

- `lovable-handoff-prompts` — prompting LV effectively
- `lovable-feature-spec-pattern` — writing specs both tools share
- `claude-code-cloud-routines` — using CC for what LV can't (cron, audits)
