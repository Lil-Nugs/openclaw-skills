---
name: janitor-sequencer
description: Create and maintain a sequential OpenClaw cron runner that processes a Linear issue range one-at-a-time with anti-concurrency guardrails. Use when setting up a janitor/ops bot that should: (1) check issue states first, (2) work only one issue per run, (3) keep Linear + repo + plans in sync, and (4) post results to a dedicated reporting channel.
---

# Janitor Sequencer

Implement a deterministic cron-driven sequencer for operational tasks.

## Inputs required

- `agentId` (target bot/agent id)
- `jobName`
- `issueRange` (example: `GAM-33..GAM-37`)
- `repoPath` (example: `/home/mattc/projects/game-hub`)
- `linearCredPath` (default: `~/.openclaw/credentials/linear.json`)
- `reportChannel` (Discord channel id)
- `interval` (example: `45m`)

## Behavior contract

Always enforce:

1. Query issue states in range first.
2. If any issue in range is `In Progress`, stop.
3. Else pick the lowest-numbered `Backlog/Todo` issue only.
4. Move that issue to `In Progress`.
5. Implement only that issue in the repo.
6. Run checks and collect evidence.
7. Sync Linear (notes + Done only if AC passed).
8. Sync plan status if affected.
9. Never start a second issue in the same run.

## Setup command template

```bash
openclaw cron add \
  --name "${JOB_NAME}" \
  --description "Sequential janitor for ${ISSUE_RANGE}" \
  --agent "${AGENT_ID}" \
  --session isolated \
  --every "${INTERVAL}" \
  --message "[Janitor Sequencer] Credential rule: read Linear API key from ${LINEAR_CRED_PATH} (field: apiKey); do not require LINEAR_API_KEY env var. Work only ${ISSUE_RANGE}. Sequence rules: query states first; if any In Progress stop; else pick lowest-numbered Backlog/Todo, set In Progress, implement only that issue in ${REPO_PATH}, run checks, add evidence, set Done only if acceptance criteria pass, sync plans.json if affected, never start next issue in same run, if blocked set blocked/comment and stop." \
  --announce --channel discord --to "channel:${REPORT_CHANNEL}"
```

## Operations

- List jobs: `openclaw cron list`
- View runs: `openclaw cron runs --id <job-id> --limit 5`
- Trigger now: `openclaw cron run <job-id>`
- Move reporting channel:
  `openclaw cron edit <job-id> --announce --channel discord --to "channel:<id>"`

## Safety checks

Before enabling long-term:

1. Confirm Linear auth works inside cron run (not just main session).
2. Confirm one-run/one-issue behavior in run summary.
3. Confirm issue transitions are reversible and evidence is attached.
4. Confirm reports only post to the configured janitor channel.
