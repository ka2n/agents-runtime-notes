# Claude Code Integration

Last verified: 2026-04-08

This file summarizes Claude Code behavior relevant to authors building tools that integrate with Claude sessions, hooks, and transcripts.

## Official References

- Anthropic Claude Code hooks reference: https://code.claude.com/docs/en/hooks
- Anthropic Claude Code hooks guide: https://code.claude.com/docs/en/hooks-guide

## Relevant External Hook Surface

Current documented Claude hook events include:

- `PreToolUse`
- `PostToolUse`
- `PermissionRequest`
- `UserPromptSubmit`
- `InstructionsLoaded`
- `Notification`
- `Stop`
- `SubagentStop`
- `CwdChanged`
- `FileChanged`
- `PreCompact`
- `PostCompact`
- `SessionStart`
- `SessionEnd`

Important practical point:

The Anthropic docs now distinguish two related permission surfaces:

- `PermissionRequest` runs when a permission dialog is about to be shown and can allow or deny on the user's behalf
- `Notification` still runs for notification delivery, including `permission_prompt`

This gives Claude a more explicit approval-related hook surface than tools that only expose pre-tool execution hooks.

## Execution and output model

Claude supports:

- command hooks
- HTTP hooks
- prompt hooks

Notable output behaviors:

- `SessionStart` and `UserPromptSubmit` can inject context
- `PreToolUse` can allow, deny, ask, or defer
- `PermissionRequest` can allow or deny, and can update input or permissions
- `Stop` and `SubagentStop` can influence whether work continues

## Subagent Detection

Hooks are useful for live behavior, but transcript files are still useful for session browsing and subagent discovery.

Observed local path layout:

- parent session: `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`
- subagent transcripts: `~/.claude/projects/<encoded-cwd>/<session-id>/subagents/agent-<agent-id>.jsonl`
- subagent metadata: `~/.claude/projects/<encoded-cwd>/<session-id>/subagents/agent-<agent-id>.meta.json`

Observed transcript hints:

- the parent session JSONL contains `tool_use` entries for the `Agent` tool
- the corresponding `tool_result` entry includes `toolUseResult.agentId`
- `.meta.json` includes fields such as `agentType` and `description`

Practical design takeaway:

- use hooks as the low-latency source for live events
- use transcripts for recovery, browsing, and subagent linkage
- prefer `PermissionRequest` over notification heuristics when you need approval-aware integrations

## Notes About JSON Output

The Claude hooks reference supports structured JSON responses for richer control, including:

- `PreToolUse` allow, deny, ask, and defer decisions
- `PermissionRequest` allow and deny decisions, plus input and permission updates
- `Stop` and `SubagentStop` continuation control
- `SessionStart` and `Notification` context injection

Claude also documents command hooks, HTTP hooks, and prompt hooks.

## Integration guidance

- If you need approval interception, Claude provides a first-class hook surface for it.
- If you need environment bootstrapping, use `CLAUDE_ENV_FILE` with `SessionStart`, `CwdChanged`, and `FileChanged`.
- If you need subagent discovery, combine hook events with transcript parsing.
- Treat transcript file layout as observed behavior unless it is explicitly documented upstream.
