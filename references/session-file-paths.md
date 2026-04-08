# Agent Session File Paths Reference

Last verified: 2026-04-08

This document lists known local session and transcript file locations for coding agents.

Use it when you are building:

- session browsers
- activity monitors
- subagent discovery
- fallback integrations for agents with limited hook support

## Why this exists

Hooks are the best low-latency integration point when an agent exposes them.

But multi-agent tools often still need transcript files for:

- session recovery after restart
- historical browsing
- subagent parent-child linking
- agents that do not expose a sufficient hook surface

## Claude Code

Main session transcript:

```text
~/.claude/projects/<encoded-cwd>/<session-id>.jsonl
```

Observed subagent layout:

```text
~/.claude/projects/<encoded-cwd>/<session-id>/subagents/agent-<agent-id>.jsonl
~/.claude/projects/<encoded-cwd>/<session-id>/subagents/agent-<agent-id>.meta.json
```

Observed useful fields and hints:

- `<encoded-cwd>` is derived from the working directory path
- parent transcript contains `tool_use` entries for the `Agent` tool
- the corresponding `tool_result` may contain `toolUseResult.agentId`
- subagent `.meta.json` may contain fields such as `agentType` and `description`

See also: [`agents/claude-code.md`](../agents/claude-code.md)

## Codex

Observed session transcript layout:

```text
~/.codex/sessions/YYYY/MM/DD/rollout-<timestamp>-<session-id>.jsonl
```

Observed subagent-related fields inside the initial `session_meta` entry:

```json
{
  "type": "session_meta",
  "payload": {
    "id": "<child-session-id>",
    "forked_from_id": "<parent-session-id>",
    "source": {
      "subagent": {
        "thread_spawn": {
          "parent_thread_id": "<parent-session-id>",
          "depth": 1
        }
      }
    }
  }
}
```

Other observed hints:

- `agent_nickname`
- `agent_role`
- spawn-side events such as `collab_agent_spawn_end` may reference the child thread

See also: [`agents/codex.md`](../agents/codex.md)

## Other agents

These are useful reference points for broader multi-agent tooling, but are not yet documented in depth here.

| Agent | Session directory | Notes |
| --- | --- | --- |
| Gemini CLI | `~/.gemini/tmp/` | directory names are cwd-derived; JSONL-like session artifacts |
| GitHub Copilot CLI | `~/.copilot/session-state/` | format not documented here yet |
| Droid | `~/.factory/sessions/` | format not documented here yet |
| OpenCode | `~/.local/share/opencode/` | format not documented here yet |

## Discovery patterns

### Watch-based discovery

Use filesystem notifications when:

- you need low-latency updates
- the platform supports inotify, fsevents, or similar APIs

### Polling-based discovery

Use polling when:

- the platform lacks watch APIs
- the agent stores files on filesystems with poor notification support
- implementation simplicity matters more than latency

Minimal polling strategy:

```text
snapshot1 = {path: (mtime, size)}
sleep N seconds
snapshot2 = {path: (mtime, size)}
diff = paths whose tuple changed
reparse changed files only
```

## Design advice

- Treat transcript paths and field names as observed behavior unless the vendor documents them.
- Keep transcript parsing lazy. Read only the minimal prefix needed for classification when possible.
- Prefer hook events for current state and transcript files for recovery and enrichment.
- Split your internal model into `live event state` and `transcript-derived metadata`.

## Prior art

- agent-sessions
  - https://github.com/jazzyalex/agent-sessions
  - Useful for surveying path conventions across multiple agents.
