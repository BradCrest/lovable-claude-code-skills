---
name: claude-code-cloud-routines
description: "Use when setting up scheduled Claude Code remote agents (Routines) to extend a Lovable-built app with automation that LV cannot do, when designing weekly / daily audits of an LV codebase, when piping reports to Notion or Slack via MCP connectors, when monitoring cross-platform parity, or when creating self-maintaining documentation. Covers cron syntax + Asia/Taipei UTC conversion, structuring routine prompts that are self-contained, attaching MCP connectors, writing routines that update single living docs vs append daily snapshots, and the smart-update pattern (skip if nothing changed)."
---

# Claude Code Cloud Routines (for LV-built apps)

Lovable doesn't do cron jobs, scheduled audits, or recurring reports.
Claude Code's [cloud routines](https://claude.ai/code/routines) fill that
gap — they spin up a remote agent on a schedule, with access to your
repos, optional MCP connectors (Notion, Slack, GitHub), and full Claude
Code tooling.

This skill covers patterns for routines that **complement** an
LV-managed app: audits, reports, content pipelines, sync checks.

## Contents

- [What routines are good for](#what-routines-are-good-for)
- [Cron + timezone basics](#cron--timezone-basics)
- [Prompt structure for routines](#prompt-structure-for-routines)
- [Smart-update pattern](#smart-update-pattern)
- [MCP connector integration](#mcp-connector-integration)
- [Failure handling](#failure-handling)
- [Routine ideas](#routine-ideas)

## What routines are good for

| Use case | Frequency |
|---|---|
| Weekly cross-platform parity audit | Mon 9am |
| Daily AI cost report | Daily 8am |
| Generate sync docs from git diff | On-change (daily check) |
| Health-check critical Edge Functions | Hourly |
| Stale PR / branch reminder | Weekly Friday |
| Weekly metrics → Slack / Notion | Mon morning |
| Auto-update UI flow documentation | Daily, only if changed |
| Smoke-test deployed app | Daily |

What routines **shouldn't** do:
- ❌ Push code changes to `main` aggressively (LV will conflict)
- ❌ Long-running interactive tasks (>30 min)
- ❌ Things requiring secrets you can't put in `sources`

## Cron + timezone basics

Routines use 5-field cron in **UTC**. Minimum interval: 1 hour.

```
*    *    *    *    *
│    │    │    │    └─ day-of-week (0=Sun, 1=Mon, ..., 6=Sat)
│    │    │    └────── month
│    │    └─────────── day-of-month
│    └──────────────── hour (0-23 UTC)
└───────────────────── minute (0-59)
```

### Asia/Taipei (UTC+8) → UTC conversion

Subtract 8 from desired local hour:

| Local time (Asia/Taipei) | UTC | Cron |
|---|---|---|
| Daily 7am | 23:00 (prev day) | `0 23 * * *` |
| Daily 8am | 00:00 | `0 0 * * *` |
| Daily 9am | 01:00 | `0 1 * * *` |
| Mon 8am | Mon 00:00 | `0 0 * * 1` |
| Tue 8am | Tue 00:00 | `0 0 * * 2` |
| Fri 5pm | Fri 09:00 | `0 9 * * 5` |

**Important**: When local hour < 8, UTC date wraps to **previous day**.
Tuesday 7am Asia/Taipei = Monday 23:00 UTC, so cron `0 23 * * 1`. Easy
to get wrong.

### Other timezones

| Local TZ | Subtract |
|---|---|
| UTC+9 (JST, KST) | 9 |
| UTC+8 (Asia/Taipei, CST) | 8 |
| UTC+1 (CET) | 1 (winter) / 2 (summer) |
| UTC-5 (EST) | -5 → add 5 |
| UTC-8 (PST) | -8 → add 8 |

## Prompt structure for routines

The routine starts a fresh agent with **zero memory of your conversation**.
The prompt must be 100% self-contained.

### Template

```markdown
你是 <ProjectName> 的 <role>. <一句話描述任務>.

兩個 repo 已 checkout 在工作目錄:
- <repo1>
- <repo2>

## 步驟一: 檢查條件 (optional, for smart-update)

1. <skip condition check>
2. 如果條件不滿足 → 印「跳過」結束。

## 步驟二: 主要工作

具體要做什麼 (用列表，不要散文).

## 步驟三: 同步輸出

把結果輸出到:
1. <git repo path/file>
2. <Notion page_id>
3. <Slack channel / Discord webhook>

## 限制

- 不要動其他檔案
- 失敗降級: <if X fails, still do Y>
```

### Anatomy

- **Role + task**: Sets the agent's mindset
- **Sources**: Reminds it what's checked out
- **Steps**: Numbered, atomic
- **Outputs**: Where to write results (be explicit)
- **Limits**: Guardrails against scope creep

## Smart-update pattern

If you want "only update when something changed":

```markdown
## Step 1: Check if update is needed

1. Read existing `docs/auto-doc.md` (if missing, treat as "needs update")
2. Get last update time T:
   `git log -1 --format=%ai docs/auto-doc.md`
3. Check if any source file changed since T:
   `git log --since=T -- src/routes/** src/components/**`
4. If no changes → output "no updates today, skipping" and exit.
5. If changes → continue to step 2.

## Step 2: Generate new doc

[regenerate content from current code]

## Step 3: Write + commit
```

Result: routine fires daily but only commits ~3× per week (when something
actually changed). No noise.

## MCP connector integration

Many routines benefit from posting to external tools. Available connectors:
[https://claude.ai/customize/connectors](https://claude.ai/customize/connectors)

Common patterns:

### Notion: weekly report → child page

```markdown
## Step 4: Sync to Notion

Parent page ID: `<your-parent-page-uuid>`

Use notion-create-pages:
- parent: { type: "page_id", page_id: "<parent>" }
- title: "YYYY-MM-DD Report"
- icon: "📊"
- content: <markdown body>

If Notion fails → still commit markdown but add 
"⚠️ Notion 未同步" note to the file.
```

### Notion: living doc → update same page

```markdown
Use notion-update-page:
- page_id: "<fixed page ID>"
- 覆寫該頁全部 content
- 保留 icon 與 title
```

### Slack: status update

```markdown
Use slack-post-message:
- channel: "#sleepycat-ops"
- text: <one-line summary>
```

### GitHub: open issue if problem found

```markdown
Use github-create-issue:
- repo: "owner/repo"
- title: "Sync drift detected"
- body: <details>
```

## Failure handling

Routines run unattended. **Failure should not block other outputs**:

```markdown
## 限制

- 如果 Notion 同步失敗 → markdown 仍要 commit (但加 "⚠️ Notion 未同步")
- 如果 git push 失敗 (LV race) → 印錯誤訊息但不要 retry 太多次 (max 3)
- 如果上游 API timeout → 跳過該檢查項，繼續其他項
- 如果整個 routine 50% 以上失敗 → 在 commit message 加 "(partial)"
```

Always specify what "failed gracefully" looks like.

## Routine ideas

For an LV-built app, here are routines worth setting up:

### 1. Cross-platform sync audit (weekly)
- Compare Web routes vs iOS screens
- Flag features that drift
- Output: weekly markdown + Notion page

### 2. UI flow auto-doc (daily smart-update)
- Scan routes + screens
- Regenerate Mermaid navigation tree
- Output: single living doc

### 3. AI cost / quota usage (daily)
- Query `daily_ai_usage` table
- Generate yesterday's cost report
- Output: Notion + warn if approaching limit

### 4. Stale PR / branch cleanup (Fridays)
- List PRs open >7 days
- List branches not merged in 30 days
- Output: Slack ping

### 5. Edge Function health check (hourly)
- Call each critical EF with test input
- Output: alert if any fail

### 6. Backlog grooming (weekly Sun)
- Scan BACKLOG.md, recent commits, recent prompts
- Suggest reordering / new items
- Output: updated BACKLOG.md PR

### 7. Smoke test (daily)
- Visit production homepage
- Check critical user paths
- Output: status badge / alert

## Cheat sheet — minimal viable routine

```json
{
  "name": "Daily X check",
  "cron_expression": "0 23 * * *",
  "enabled": true,
  "job_config": {
    "ccr": {
      "environment_id": "<env_id>",
      "session_context": {
        "model": "claude-sonnet-4-6",
        "sources": [
          {"git_repository": {"url": "https://github.com/your/repo"}}
        ],
        "allowed_tools": ["Bash", "Read", "Write", "Edit", "Glob", "Grep"]
      },
      "events": [
        {"data": {
          "uuid": "<v4-uuid>",
          "session_id": "",
          "type": "user",
          "parent_tool_use_id": null,
          "message": {"role": "user", "content": "<your prompt>"}
        }}
      ]
    }
  }
}
```

Manage via [https://claude.ai/code/routines](https://claude.ai/code/routines).

## Companion skills

- `lovable-vs-claude-code-allocation` — what to put in a routine vs leave to LV
- `lovable-git-race-handling` — routines that commit must handle LV races
