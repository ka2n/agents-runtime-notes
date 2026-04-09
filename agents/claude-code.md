# Claude Code Integration

Last verified: 2026-04-09

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

## Session File Format

Claude session transcripts are JSONL files, but the line schema is more heterogeneous than a simple chat log.

Observed parent session path:

- `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`

Observed subagent paths:

- `~/.claude/projects/<encoded-cwd>/<session-id>/subagents/agent-<agent-id>.jsonl`
- `~/.claude/projects/<encoded-cwd>/<session-id>/subagents/agent-<agent-id>.meta.json`

Observed top-level JSONL line families:

- `file-history-snapshot`
- `user`
- `assistant`
- `system`
- `progress`

Observed common envelope fields on conversation entries:

- `parentUuid`
- `uuid`
- `timestamp`
- `sessionId`
- `cwd`
- `gitBranch`
- `version`
- `isSidechain`

Observed message payload patterns:

- `type: "user"` records may contain normal user prompts, command echoes, or `tool_result`
- `type: "assistant"` records may contain plain text or `tool_use`
- `type: "progress"` records include hook progress and other execution progress
- `type: "system"` records include synthesized summaries such as stop-hook results

Observed `file-history-snapshot` shape:

```json
{
  "type": "file-history-snapshot",
  "messageId": "<uuid>",
  "snapshot": {
    "messageId": "<uuid>",
    "trackedFileBackups": {
      "path/to/file": {
        "backupFileName": "<hash>@v2",
        "version": 2,
        "backupTime": "2026-03-12T16:09:24.803Z"
      }
    },
    "timestamp": "2026-03-12T16:09:24.788Z"
  },
  "isSnapshotUpdate": false
}
```

Practical interpretation:

- the transcript interleaves conversation and file-backup metadata
- file backup metadata is session-local and keyed by message IDs
- parsers should not assume every line is a conversational turn

## Additional Local Metadata

Claude appears to maintain a per-project session index alongside raw JSONL files.

Observed file:

- `~/.claude/projects/<encoded-cwd>/sessions-index.json`

Observed entry fields include:

- `sessionId`
- `fullPath`
- `fileMtime`
- `firstPrompt`
- `summary`
- `messageCount`
- `created`
- `modified`
- `gitBranch`
- `projectPath`
- `isSidechain`

Observed subagent metadata file:

- `subagents/agent-<agent-id>.meta.json`

Observed minimal example:

```json
{ "agentType": "Explore" }
```

Practical design takeaway:

- use parent and subagent JSONL files as the canonical transcript source
- use `sessions-index.json` as a convenience index, not the source of truth
- treat `file-history-snapshot` lines as a distinct metadata stream embedded in the transcript

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
