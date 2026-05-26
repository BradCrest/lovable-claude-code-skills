# Contributing

Got a war story from your Lovable + Claude Code workflow? Help others avoid
the same trap.

## How to add a new skill

1. Pick a clear, kebab-case name. Examples that work:
   - `lovable-deploy-monitoring`
   - `supabase-rls-debugging-with-claude-code`
   - `expo-eas-build-after-lv`
   - `vercel-deploy-from-claude-code-pr`

2. Create the folder + SKILL.md:

```bash
mkdir -p skills/<your-skill-name>
$EDITOR skills/<your-skill-name>/SKILL.md
```

3. SKILL.md template:

```markdown
---
name: your-skill-name
description: "Use when [scenario A], [scenario B], [scenario C]. Covers [topic 1], [topic 2]. Avoids [common mistake]."
---

# Your Skill Title

One-paragraph intro: when this matters, who it's for.

## Contents

- [Section 1](#section-1)
- [Section 2](#section-2)
- [Common mistakes](#common-mistakes)

## Section 1

Concrete advice. Code samples. Real examples beat abstract description.

## Common mistakes

| ❌ Doing | ✅ Try instead |
|---|---|
| ... | ... |
```

4. Update `README.md` skills table

5. Open a PR

## Quality bar

| Yes | No |
|---|---|
| Concrete code samples | Hand-wavy advice |
| War stories from production | Hypothetical scenarios |
| Specific commands, file names | Vague references |
| English description (auto-trigger) | 中文 description（trigger 較不穩，body 可以中英混合）|
| Acknowledged trade-offs | Religious "always X" rules |

## Description field — the most important part

The `description` is what makes the skill auto-trigger. Be specific.

**Too vague** (won't trigger):
```yaml
description: "Helpful tips for Lovable users"
```

**Just right** (triggers on the right scenarios):
```yaml
description: "Use when working in a repo managed by Lovable where LV's concurrent auto-commits cause 'Updates were rejected' on git push, when you need to recover after your local branch diverges from LV's, or when planning branching strategy to coexist with LV. Covers `git pull --rebase` recovery, branch naming to avoid conflicts, and the 'don't amend LV's commits' rule."
```

Rule of thumb: list 2-3 trigger scenarios + 2-3 topics covered. ~50-80 words.

## Naming conventions

- Skill folder: kebab-case (`lovable-handoff-prompts`)
- File: always `SKILL.md` (uppercase)
- Sections use `## H2` not `### H3` for top-level (keeps TOC clean)
- Code blocks: language-tagged when possible (`````ts`, `````bash`)

## License

By contributing, you agree your contributions are licensed under MIT (see
[LICENSE](LICENSE)).
