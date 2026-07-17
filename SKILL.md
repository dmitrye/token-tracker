---
name: token-tracker-jira-updater
description: Reports real coding-agent token usage (measured via ccusage) to custom fields on a Jira issue after a planning or implementation phase completes. Works with any agent CLI ccusage supports — Claude Code, Codex, Gemini CLI, Copilot CLI, OpenCode, Amp, Droid, Goose, Qwen and more. Use whenever the user finishes a planning or implementation phase on a Jira-tracked ticket, or says anything like "track tokens", "log token usage", "update Jira with token usage", or asks how many tokens this session or ticket consumed. Never estimate token counts from memory — always measure.
license: Apache-2.0
metadata:
  version: 1.0.0
  requires: npx (runs ccusage on demand), an Atlassian MCP server with Jira read/write
---

# Token Usage Tracking & Jira Update

Measures real token usage for the current agent session and writes it, additively, to two
custom fields on a Jira issue — one for planning tokens, one for implementation tokens.

This skill is agent-neutral. It reads whatever local telemetry your coding agent already
writes, via [`ccusage`](https://ccusage.com), so it works the same way under Claude Code,
Codex, Gemini CLI, and the other supported CLIs listed below.

## Ground rule — no fabrication

The agent has no reliable memory of token counts. Never estimate, guess, or infer a token
count from context length, task complexity, or "feel". This is not a stylistic preference:
earlier versions of this workflow wrote billions of fabricated tokens into a Jira field this
way, and the corrupted value then compounded because each run added on top of it.

The only trustworthy source is real usage telemetry pulled via `ccusage`.

- `ccusage` unavailable, or returns no data for the current session → **stop and ask the user
  for the number.** Do not substitute a guess.
- A field's current value is wildly inconsistent with what a fresh `ccusage` pull could
  plausibly produce (e.g. billions of tokens for one ticket) → **stop and ask the user how to
  reconcile it** rather than silently adding on top of a corrupted number.

## Supported agents

`ccusage` reads local usage files, so the tracked agent must be one it knows. As of ccusage's
current docs, the sources are:

| Agent | Source id | Default data location |
| --- | --- | --- |
| Claude Code | `claude` | `~/.config/claude/projects/`, `~/.claude/` |
| Codex | `codex` | `${CODEX_HOME:-~/.codex}` |
| OpenCode | `opencode` | `${OPENCODE_DATA_DIR:-~/.local/share/opencode}` |
| Amp | `amp` | `${AMP_DATA_DIR:-~/.local/share/amp}` |
| Droid | `droid` | `${DROID_SESSIONS_DIR:-~/.factory/sessions}` |
| Codebuff | `codebuff` | `${CODEBUFF_DATA_DIR:-~/.config/manicode}` |
| Hermes Agent | `hermes` | `${HERMES_HOME:-~/.hermes}/state.db` |
| pi-agent | `pi` | `${PI_AGENT_DIR:-~/.pi/agent/sessions}` |
| Goose | `goose` | Standard Goose data roots, or `GOOSE_PATH_ROOT` |
| OpenClaw | `openclaw` | `${OPENCLAW_DIR:-~/.openclaw}` |
| Kilo | `kilo` | `${KILO_DATA_DIR:-~/.local/share/kilo}` |
| Kimi | `kimi` | `${KIMI_DATA_DIR:-~/.kimi}` (also `~/.kimi-code`) |
| Qwen | `qwen` | `${QWEN_DATA_DIR:-~/.qwen}` |
| GitHub Copilot CLI | `copilot` | `~/.copilot/otel/*.jsonl` |
| Gemini CLI | `gemini` | `${GEMINI_DATA_DIR:-~/.gemini/tmp}` |

Some agents are investigated but unsupported because their local files don't carry reliable
token counts — Antigravity CLI, Grok CLI, and Devin CLI at time of writing. If the running
agent isn't in this table, say so and ask the user for the number rather than guessing.

This table tracks ccusage and may drift. If a source id is rejected, run
`npx ccusage@latest --help` or check <https://ccusage.com/guide/> for the current list, and
tell the user this skill's table needs updating.

## Setup (once per repo)

Config and state live in the **consuming project**, at `.token-tracker/config.json`. A
per-user fallback at `~/.token-tracker/config.json` is also honored; the project file wins
where both exist.

Do not write config or state into the skill's own folder — installers such as `npx skills`
symlink skills by default, so one canonical copy may be shared across every project on the
machine.

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
| `source` | ccusage source id for the agent being tracked (see table above). |
| `planningFieldId` | Jira custom field ID for planning tokens. |
| `implFieldId` | Jira custom field ID for implementation tokens. |
| `planningRatio` | Share of a delta attributed to planning (step 4). Default `0.33`. |
| `cacheReadWeight` | Fraction of cache-read tokens counted as real work (step 2). Default `0.1`. |

**Configuration varies by agent.** The example above is Claude Code. For another agent, set
`source` to its id from the table, and expect these to differ:

- **Session id format and where to find it** — see step 1.
- **Cache token reporting** — some agents report `cacheCreationTokens` / `cacheReadTokens`,
  others don't. Where cache fields are absent or zero, `cacheReadWeight` has no effect and
  `adjustedTotal` collapses to `totalTokens`. That's expected, not a failure.
- **`planningRatio`** — calibrate per project *and* per agent; it isn't portable between them.
- **Data location** — if the agent stores data somewhere non-default, set its environment
  variable from the table before invoking `ccusage`.

**Finding the Jira field IDs:** custom field IDs are per-Jira-instance and often per-project.
Call `getJiraIssue` on any issue in the target project with `expand=names`, or use the Jira
field API, and match the human-readable field names to their `customfield_*` IDs. Confirm with
the user before writing. If no config file exists, ask the user for the source id and field
IDs, and offer to create it.

Add to the project's `.gitignore`:

```
.token-tracker/state.json
```

## Workflow

### 1. Identify the session and the ticket

Session identifiers are agent-specific. Determine `<session-id>` for the configured `source`:

- **Claude Code** — the UUID in the scratchpad path
  (`/tmp/claude-.../<project-slug>/<session-uuid>/scratchpad`; on macOS `/private/tmp/…`), or
  the most recently modified `*.jsonl` under `~/.claude/projects/<project-slug>/`. The id is
  the filename without the `.jsonl` extension.
- **Any source** — list today's sessions and take the one with the latest `lastActivity`:

  ```bash
  npx ccusage@latest <source> session --json --since <YYYYMMDD>
  ```

  Each entry carries a `sessionId`. If more than one session was active and the current one
  is ambiguous, ask the user rather than assuming.

Then identify the target Jira issue key from the active branch name, recent commit messages,
or ask the user if ambiguous. Never skip tracking silently because the ticket was unclear.

### 2. Pull real usage for this session

```bash
npx ccusage@latest <source> session --id <session-id> --json --since <YYYYMMDD>
```

Add `--offline` to use cached pricing in air-gapped environments. Omitting `<source>`
aggregates every detected agent — don't do that here; a session belongs to one agent.

Sum across all entries for the session: `totalTokens` (= `inputTokens` + `outputTokens` +
`cacheCreationTokens` + `cacheReadTokens`) and `cacheReadTokens`. Treat missing cache fields
as `0`.

Compute the session's adjusted usage:

```
adjustedTotal = totalTokens - (1 - cacheReadWeight) * cacheReadTokens
```

At the default `cacheReadWeight` of 0.1, this credits input, output, and cache-creation tokens
in full but only 10% of cache-read tokens. Cache reads are mostly free re-reads of prior
context rather than new work, but they aren't pure overhead either. Do not use raw
`inputTokens + outputTokens` alone and do not use raw `totalTokens` alone — on agents that
report caching, both under- or over-count real usage.

### 3. Compute the delta since the last tracked run

This skill can run more than once per session (e.g. once after planning, once after
implementation). To avoid double-counting, keep a running baseline in
`.token-tracker/state.json`, keyed by source and session id:

```json
{
  "claude:<session-id>": { "ticket": "ABC-123", "lastAdjustedTotal": 0 }
}
```

- No entry for this session → baseline is 0 (first run this session).
- `deltaImpl = adjustedTotal - lastAdjustedTotal`
- `deltaImpl <= 0` → nothing new to report; stop without writing.
- After a **successful** Jira update, write the new `adjustedTotal` back to the state file.

### 4. Derive planning tokens

```
deltaPlanning = round(planningRatio * deltaImpl)
```

`planningRatio` comes from the config. It's a per-project, per-agent constant, calibrated once
by comparing observed planning-phase usage to implementation-phase usage. Don't re-derive it
per run; if it needs revising, update the config.

### 5. Update Jira (additive)

- `getJiraIssue` for the two fields' current values **first**, add the deltas, then
  `editJiraIssue` with the new totals. Never overwrite outright — these fields accumulate
  across sessions and tickets. A full baseline *reset* is a separate, explicit user request,
  not something this workflow does on its own.
- Add a short comment to the issue recording what changed: the agent, the deltas added, and
  the new field totals, so history is auditable without re-running `ccusage`.

## Failure & edge cases

| Situation | Action |
| --- | --- |
| Running agent isn't a ccusage source | Say so; ask the user for the number. Don't guess. |
| `ccusage` returns nothing for the session id | Ask the user for the real number; don't guess. |
| No config file / unknown source or field IDs | Ask the user; offer to create the config. |
| Source id rejected by ccusage | Check current docs; report that this skill's table is stale. |
| Agent reports no cache tokens | Normal — `adjustedTotal` equals `totalTokens`. Proceed. |
| No Jira ticket identifiable | Stop and ask; don't skip tracking silently. |
| Delta is negative or zero | Skip; don't write. |
| Existing field value looks corrupted | Stop and ask how to reconcile before adding. |
| Jira write fails | Do **not** update the state file, so the delta is retried next run. |
