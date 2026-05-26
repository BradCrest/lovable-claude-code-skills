---
name: lovable-git-race-handling
description: "Use when 'Updates were rejected' on git push in a Lovable-managed repo, when LV's concurrent auto-commits conflict with your local work, or when planning branch strategy to coexist with LV. Covers `git pull --rebase` recovery, why `--force` push is dangerous, and pre-flight `git fetch` checks."
---

# Lovable Git Race Handling

Lovable's agent commits to `main` whenever a user submits a prompt or
clicks "Apply changes". If you're working in the same repo from Claude
Code / your local machine, you WILL hit push collisions.

This skill covers the playbook for **co-existing with LV's concurrent
commits without losing work**.

## Contents

- [Why this happens](#why-this-happens)
- [Recovery: the 30-second fix](#recovery-the-30-second-fix)
- [Prevention: pre-flight checks](#prevention-pre-flight-checks)
- [Branch strategy options](#branch-strategy-options)
- [What NOT to do](#what-not-to-do)
- [War stories](#war-stories)

## Why this happens

LV's architecture:
- User opens LV → submits prompt → LV runs agent → agent commits to `main`
- LV pushes to GitHub immediately, no PR
- No way to "lock" the repo from LV's side

Your architecture:
- You / Claude Code work locally
- You commit, then `git push`
- Push rejected if LV pushed something newer

The collision window:
```
T+0   You: git pull (clean)
T+10  You: edit files, commit
T+20  LV user: submits prompt
T+25  LV: commits + pushes
T+30  You: git push → REJECTED
```

## Recovery: the 30-second fix

When push fails with "Updates were rejected":

```bash
# 1. Pull LV's changes onto your branch
git pull --rebase

# 2. If clean (no conflict) → push
git push

# 3. If conflict → resolve, then:
git add <conflicted-files>
git rebase --continue
git push
```

**Don't**:
- ❌ `git push --force` — overwrites LV's work, breaks LV's view of the repo
- ❌ `git pull` (no `--rebase`) — creates ugly merge commits in main history
- ❌ `git reset --hard origin/main` — discards your local commit

## Prevention: pre-flight checks

Before any local commit:

```bash
# 1. See what changed remotely since you last pulled
git fetch
git log HEAD..origin/main --oneline

# If output is empty → safe to work
# If output shows LV commits → pull first, then work
```

Make this a habit. 10 seconds upfront saves 5 minutes of conflict resolution.

For Claude Code sessions, you can add this to your project's CLAUDE.md:

```markdown
## Before editing any file in this Lovable-managed repo

Run `git fetch && git log HEAD..origin/main --oneline` first.
If there are remote commits, run `git pull --rebase` before editing.
```

## Branch strategy options

### Option A: Always work on main (default)

- **Pros**: Simple. Works with LV's view of the repo.
- **Cons**: Push collisions are common.
- **Use when**: Small changes, low-frequency LV usage.

### Option B: Work on a feature branch, merge after

- **Pros**: No collisions while developing. LV unaffected.
- **Cons**: Merging back to main can still conflict with LV's recent changes.
- **Use when**: Multi-day work, large refactors.

```bash
git checkout -b feat/my-thing
# work, commit, push to remote feat/my-thing
# When ready:
git checkout main
git pull --rebase
git merge feat/my-thing
git push
```

### Option C: Disable LV writes while you work

Most LV plans have a way to pause / lock changes. If yours does, use it
for big multi-day work.

## What NOT to do

| Action | Why bad |
|---|---|
| `git push --force` | Overwrites LV's commits |
| `git commit --amend` after LV pushed | You'd be amending LV's work, not yours |
| `git rebase -i` interactive | Hard to recover if LV pushes mid-rebase |
| Working on the same file LV is currently editing | Guaranteed conflict |
| Long-running local branches without `git fetch` | Drift accumulates |

## War stories

### "I lost my Edge Function changes"

I wrote a new Supabase Edge Function locally + pushed it. Next morning, LV
user clicked "make the home page faster". LV's agent restructured several
files including `supabase/functions/_shared/`. LV's commit didn't delete
my EF file, but it added a new shared util that broke my EF's import.

**Fix**: I would have noticed faster with the morning `git fetch` check.
The actual recovery was simple — rebase, fix import path, push.

### "git push --force erased the Friday LV changes"

A teammate hit collision, panicked, ran `git push --force`. This erased
about 6 LV commits worth of UI tweaks from Friday afternoon. LV's prompt
history still showed the prompts that triggered those commits, but the
code was gone.

**Recovery**: had to re-submit each prompt to LV again. Lost ~2 hours.

**Lesson**: never `--force` push to a LV-managed `main`. Use `--force-with-lease`
at worst, but really, just rebase.

### "LV undid my migration"

I added `supabase/migrations/0042_new_column.sql` and pushed. The next LV
session ran `supabase db reset` as part of its setup (LV regenerates types
sometimes). My migration was in the migration log but the LV pipeline's
local Supabase didn't have it applied.

**Fix**: Run migrations in Supabase Cloud dashboard, not just via git. LV
respects cloud-applied migrations, doesn't try to re-run them.

## Cheat sheet

```bash
# Morning routine before any work in LV repo
git fetch && git log HEAD..origin/main --oneline

# Push collision recovery
git pull --rebase
git push

# Conflict during rebase
git status        # see which files
# edit files
git add <files>
git rebase --continue
git push

# If you really need to abandon your local
git fetch && git reset --hard origin/main   # NUKES local changes
```

## Companion skills

- `lovable-handoff-prompts` — write prompts that minimize LV's blast radius
- `lovable-vs-claude-code-allocation` — choose tool to reduce concurrent work on same area
