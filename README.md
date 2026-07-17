# Token Tracker (token-tracker)

Agentic skill that measures **real** coding-agent token usage and writes it to a Jira ticket.

After a planning or implementation phase finishes, the skill pulls actual session telemetry
with [`ccusage`](https://ccusage.com), computes how many tokens are new since the last run,
and adds them to two custom fields on the Jira issue — one for planning, one for
implementation — plus an audit comment.

Its core rule: **the agent never estimates token counts.** If telemetry isn't available, it
stops and asks rather than guessing. (This is what the skill exists to prevent — a guessing
agent once wrote billions of fabricated tokens into a Jira field, and the bad number then
compounded on every subsequent additive run.)

## Agent support

The skill is agent-neutral: it reads whatever local usage files your CLI already writes, via
ccusage. Supported sources are Claude Code (`claude`), Codex (`codex`), OpenCode
(`opencode`), Amp (`amp`), Droid (`droid`), Codebuff (`codebuff`), Hermes Agent (`hermes`),
pi-agent (`pi`), Goose (`goose`), OpenClaw (`openclaw`), Kilo (`kilo`), Kimi (`kimi`), Qwen
(`qwen`), GitHub Copilot CLI (`copilot`), and Gemini CLI (`gemini`).

Antigravity CLI, Grok CLI, and Devin CLI are **not** supported — their local files don't carry
reliable token counts. See [ccusage's source list](https://ccusage.com/guide/) for the current
state of play; SKILL.md carries the full table with each agent's data location.

## Install

```bash
# Project-level (recommended — the config is per-project anyway)
npx skills add dmitrye/token-tracker

# User-level, all projects
npx skills add dmitrye/token-tracker -g

# Specific agents only
npx skills add dmitrye/token-tracker -a claude-code
```

Manual install:

```bash
git clone https://github.com/dmitrye/token-tracker.git
mkdir -p .claude/skills/token-tracker-jira-updater      # or your agent's skills dir
cp token-tracker/SKILL.md .claude/skills/token-tracker-jira-updater/
```

## Requirements

- `npx` on PATH (`ccusage` is fetched on demand — no separate install).
- A coding agent ccusage supports (see above).
- An Atlassian MCP server configured with Jira read/write access.
- Two numeric custom fields on your Jira project, one for planning tokens and one for
  implementation tokens.

## Configure

Copy `config.example.json` to `.token-tracker/config.json` in the project you want tracked:

```json
{
  "source": "claude",
  "planningFieldId": "customfield_XXXXX",
  "implFieldId": "customfield_YYYYY",
  "planningRatio": 0.33,
  "cacheReadWeight": 0.1
}
```

| Key | Meaning |
| --- | --- |
| `source` | ccusage source id for the agent being tracked. |
| `planningFieldId` | Jira custom field ID for planning tokens. |
| `implFieldId` | Jira custom field ID for implementation tokens. |
| `planningRatio` | Share of each delta attributed to planning. Calibrate per project and agent. |
| `cacheReadWeight` | How much a cache-read token counts as real work. `0.1` = 10%. |

The example is Claude Code; swap `source` for your agent's id. Session-id format, cache-token
reporting, and the right `planningRatio` all vary by agent — SKILL.md spells out the
differences. A per-user fallback at `~/.token-tracker/config.json` is honored, with the
project file winning.

Field IDs are per-Jira-instance. Ask the agent to look them up — it can call `getJiraIssue`
with `expand=names` and match the display names to their `customfield_*` IDs.

Then gitignore the run state:

```
.token-tracker/state.json
```

Config and state deliberately live in the **consuming project**, not in the skill folder:
`npx skills` symlinks skills by default, so one canonical copy may be shared across every
project on the machine.

## Use

The skill triggers on its own when a planning or implementation phase wraps on a
Jira-tracked ticket. You can also invoke it directly:

> track tokens for this session

> update Jira with token usage for ABC-123

## How usage is counted

```
adjustedTotal  = totalTokens - (1 - cacheReadWeight) * cacheReadTokens
deltaImpl      = adjustedTotal - lastAdjustedTotal   # per session, from state.json
deltaPlanning  = round(planningRatio * deltaImpl)
```

On agents that report caching, counting `totalTokens` raw overstates real work — cache reads
are mostly free re-reads of prior context — but excluding them understates it, since they
aren't pure overhead either. The weighting splits the difference. On agents that don't report
cache tokens, `adjustedTotal` is simply `totalTokens`. Writes are always additive; a reset is
a separate, explicit request.

## License

MIT
