# GitHub Copilot CLI Integration

Last verified: 2026-04-10

This file summarizes GitHub Copilot CLI behavior relevant to authors building tools that integrate with Copilot sessions, hooks, and transcripts.

## Product History

GitHub Copilot CLI has two distinct generations:

- **`gh copilot`** (old): a `gh` CLI extension (`github/gh-copilot`), deprecated 2025-10-25 (source: archived repository)
- **`copilot`** (new): a standalone agentic coding assistant (`github/copilot-cli`), GA 2026-02-25 (source: GitHub changelog)

This document focuses on the new standalone `copilot` CLI.

## Official References

- GitHub Copilot CLI docs: https://docs.github.com/en/copilot/how-tos/use-copilot-agents/use-copilot-cli
- Copilot CLI configuration: https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/configure-copilot-cli
- Copilot CLI hooks concept: https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-hooks
- Copilot CLI hooks how-to: https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/use-hooks
- Copilot CLI hooks reference: https://docs.github.com/en/copilot/reference/hooks-configuration
- Copilot CLI custom instructions: https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-custom-instructions
- Copilot CLI repository: https://github.com/github/copilot-cli
- Old `gh copilot` repository (archived): https://github.com/github/gh-copilot

## Relevant External Hook Surface

Confirmed in vendor docs: Copilot CLI exposes a hook system with 8 documented events.

The concept page lists all 8 events:

- `sessionStart`
- `sessionEnd`
- `userPromptSubmitted`
- `preToolUse`
- `postToolUse`
- `agentStop`
- `subagentStop`
- `errorOccurred`

Important caveat: the hooks reference page provides full input/output schemas for only 6 of these events. `agentStop` and `subagentStop` are listed in the concept page but lack detailed schema documentation.

## Hook Configuration

Confirmed in vendor docs: hooks are configured via JSON files in `.github/hooks/`.

Config file location:

- `.github/hooks/*.json` (any `.json` filename, per-repo)
- for Copilot cloud agent: must be on the repository's default branch
- for Copilot CLI: loaded from the current working directory's `.github/hooks/` directory

Observed in prior art (CodeIsland): `~/.copilot/hooks/` is used as a global hooks directory. This user-level path is not documented in official docs but is used by third-party tools.

Configuration format:

```json
{
  "version": 1,
  "hooks": {
    "preToolUse": [
      {
        "type": "command",
        "bash": "./scripts/policy-check.sh",
        "powershell": "./scripts/policy-check.ps1",
        "cwd": "scripts",
        "env": { "KEY": "value" },
        "timeoutSec": 30
      }
    ]
  }
}
```

Hook object properties:

| Property | Required | Description |
| --- | --- | --- |
| `type` | yes | must be `"command"` |
| `bash` | yes (Unix) | bash script path or inline command |
| `powershell` | yes (Windows) | PowerShell script path or inline command |
| `cwd` | no | working directory relative to repo root |
| `env` | no | additional environment variables merged with existing env |
| `timeoutSec` | no | max execution time in seconds (default: 30) |

Multiple hooks per event execute sequentially in array order.

## Execution and Output Model

Copilot CLI hooks are command-based only (no HTTP or prompt hooks).

Notable behaviors:

- hooks receive JSON on `stdin`
- hooks output JSON on `stdout` (only meaningful for `preToolUse`)
- hooks run synchronously and block agent execution
- default timeout is 30 seconds, configurable per hook via `timeoutSec`

Important `toolArgs` detail: unlike Claude Code and Codex where `tool_input` is a nested object, Copilot CLI's `toolArgs` is a **JSON string** that requires double-parsing.

### `preToolUse` deny decision

`preToolUse` is the only event where hook output affects agent behavior.

Confirmed in vendor docs: `permissionDecision` supports `"allow"`, `"deny"`, and `"ask"`, but **only `"deny"` is currently processed**.

Output format:

```json
{
  "permissionDecision": "deny",
  "permissionDecisionReason": "Destructive operations blocked by policy."
}
```

All other hook events ignore stdout output.

## Input JSON Examples

### `sessionStart`

```json
{
  "timestamp": 1704614400000,
  "cwd": "/path/to/project",
  "source": "new",
  "initialPrompt": "Create a new feature"
}
```

`source` values: `"new"`, `"resume"`, `"startup"`.

### `sessionEnd`

```json
{
  "timestamp": 1704618000000,
  "cwd": "/path/to/project",
  "reason": "complete"
}
```

`reason` values: `"complete"`, `"error"`, `"abort"`, `"timeout"`, `"user_exit"`.

### `userPromptSubmitted`

```json
{
  "timestamp": 1704614500000,
  "cwd": "/path/to/project",
  "prompt": "Fix the authentication bug"
}
```

Output is ignored. Prompt modification is not supported.

### `preToolUse`

```json
{
  "timestamp": 1704614600000,
  "cwd": "/path/to/project",
  "toolName": "bash",
  "toolArgs": "{\"command\":\"rm -rf dist\",\"description\":\"Clean build directory\"}"
}
```

Note: `toolArgs` is a JSON string, not a nested object.

### `postToolUse`

```json
{
  "timestamp": 1704614700000,
  "cwd": "/path/to/project",
  "toolName": "bash",
  "toolArgs": "{\"command\":\"npm test\"}",
  "toolResult": {
    "resultType": "success",
    "textResultForLlm": "All tests passed (15/15)"
  }
}
```

`resultType` values: `"success"`, `"failure"`, `"denied"`.

### `errorOccurred`

```json
{
  "timestamp": 1704614800000,
  "cwd": "/path/to/project",
  "error": {
    "message": "Network timeout",
    "name": "TimeoutError",
    "stack": "TimeoutError: Network timeout\n    at ..."
  }
}
```

## Custom Instructions

Confirmed in vendor docs: Copilot CLI discovers custom instructions from multiple sources.

- **Repository-wide**: `.github/copilot-instructions.md`
- **Path-specific**: `.github/instructions/**/*.instructions.md` (note the `.instructions.md` suffix)
- **Root-level instruction files**: `AGENTS.md`, `CLAUDE.md`, `GEMINI.md` at repository root
- **Custom agents**: `.github/agents/` (repo), `~/.copilot/agents/` (user), `.github-private/agents/` (org)
- **Local instructions**: `$HOME/.copilot/copilot-instructions.md`
- **Environment variable**: `COPILOT_CUSTOM_INSTRUCTIONS_DIRS` for additional directories

## MCP Server Integration

Copilot CLI supports MCP servers:

- configuration: `~/.copilot/mcp-config.json`
- built-in: GitHub MCP server (repository operations, issues, PRs)
- management: `/mcp show|add|edit|delete|disable|enable`

## Session File Format

Confirmed in vendor docs: Copilot CLI stores session data locally.

Session storage locations:

```text
~/.copilot/session-state/<sessionId>.jsonl           # legacy: flat JSONL
~/.copilot/session-state/<sessionId>/events.jsonl    # current: per-session directory
~/.copilot/session-state/<sessionId>/workspace.yaml  # current: session metadata (e.g. renamed title)
~/.copilot/session-store.db                          # SQLite session store (derived index, Chronicle data)
```

Source: path layouts observed in `agent-sessions` CopilotSessionParser and CopilotSessionDiscovery.

The `COPILOT_HOME` environment variable can override the base directory (`~/.copilot/`). The `--config-dir` flag takes precedence over `COPILOT_HOME`.

Important: `session-state/` is the canonical session history. `session-store.db` is a derived cross-session index used by the Chronicle system.

### JSONL Line Schema

Observed in prior art (`agent-sessions` CopilotSessionParser): each JSONL line is an event envelope.

Observed envelope fields:

- `type`: event type string
- `data`: event-specific payload object
- `id`: message identifier
- `timestamp`: ISO 8601 or Unix timestamp
- `parentId`: reference to parent message

Observed event types:

| Type | Description | Key `data` fields |
| --- | --- | --- |
| `session.start` | session begins | `sessionId` |
| `session.model_change` | model switched | `newModel` |
| `session.info` | informational | `infoType`, `message` |
| `session.truncation` | context truncated | |
| `user.message` | user prompt | `content` |
| `assistant.message` | assistant response | `content`, `toolRequests[]` |
| `assistant.turn_start` | turn boundary | |
| `assistant.turn_end` | turn boundary | |
| `tool.execution_start` | tool invoked | `toolCallId`, `toolName`, `arguments` |
| `tool.execution_complete` | tool finished | `toolCallId`, `success`, `result.content` |

Observed tool call flow:

1. `assistant.message` with `toolRequests[]` containing `toolCallId`, `name`, `arguments`
2. `tool.execution_start` with `toolCallId`, `toolName`, `arguments`
3. `tool.execution_complete` with `toolCallId`, `success` (bool), `result.content`

Observed `session.info` subtypes:

- `infoType: "folder_trust"` with `message: "Folder /path/to/dir has been added to trusted folders."`

### `workspace.yaml`

In the current directory-based layout (`<sessionId>/events.jsonl`), a `workspace.yaml` file may exist alongside `events.jsonl`. Observed to contain a top-level `name` field, written when the user runs `/rename`.

### Comparison with Claude Code and Codex session formats

| Aspect | Claude Code | Codex | Copilot CLI |
| --- | --- | --- | --- |
| Format | JSONL | JSONL | JSONL |
| First line | mixed (no fixed type) | `session_meta` | `session.start` |
| Event envelope | `type`, `uuid`, `parentUuid`, `timestamp`, `sessionId` | `timestamp`, `type`, `payload` | `type`, `data`, `id`, `timestamp`, `parentId` |
| Tool call pattern | `tool_use` / `tool_result` in message content | `function_call` / `function_call_output` | `assistant.message.toolRequests` / `tool.execution_start` / `tool.execution_complete` |
| File snapshots | `file-history-snapshot` lines | not documented | not observed |
| Subagent linkage | separate subagent directory | `forked_from_id` in `session_meta` | not observed |

Configuration and metadata locations:

```text
~/.copilot/config.json              # global configuration, trusted folders
~/.copilot/mcp-config.json          # MCP server configurations
~/.copilot/lsp-config.json          # LSP configurations
~/.copilot/agents/                  # user-level custom agent definitions
```

Session persistence features:

- `copilot --continue`: resume most recent session
- `copilot --resume`: interactive session picker
- `/resume [session-id]`: resume from within an active session

## Subagent Detection

Copilot CLI's subagent model differs from Claude Code and Codex:

- built-in agents (Explore, Task, Plan, Code-review) are internal
- `subagentStop` hook event fires when a subagent completes, before returning results to the parent agent
- `/delegate` creates an external coding agent that works asynchronously and opens a draft PR
- `/fleet` dispatches multiple agents in parallel (PR behavior not confirmed from primary sources)
- there is no documented transcript-based parent-child linkage comparable to Codex's `forked_from_id` or Claude Code's subagent directory structure
- session format and `session-store.db` schema are not publicly documented

Practical design takeaway:

- use `subagentStop` hook for live subagent completion events
- for `/delegate`, track output via the GitHub PR API
- transcript-based subagent linkage is not available for Copilot CLI

## Authentication

Confirmed in vendor docs: token lookup order:

1. `COPILOT_GITHUB_TOKEN` environment variable
2. `GH_TOKEN` environment variable
3. `GITHUB_TOKEN` environment variable
4. Stored OAuth token (obtained via `/login`)
5. GitHub CLI (`gh`) token fallback

Fine-grained PATs with "Copilot Requests" permission are a supported token type, usable in any of the above methods.

## Integration Guidance

- Use `.github/hooks/*.json` for event-based integration. `preToolUse` supports deny decisions for policy enforcement.
- For context injection, Copilot CLI hooks do not support stdout-based context injection like Claude Code's `SessionStart`. Use custom instructions files instead.
- For session discovery, `~/.copilot/session-state/` is the canonical session source. `session-store.db` is a derived index.
- For multi-agent coordination, `/delegate` outputs are tracked as GitHub PRs, making the GitHub API a natural integration point.
- Note that `toolArgs` is a JSON string requiring double-parse, unlike Claude Code and Codex which provide tool input as a nested object.
- `agentStop` and `subagentStop` are listed as hook events but lack detailed schema documentation. Use with caution.

## Known Constraints

- only `"deny"` is currently processed in `preToolUse` output; `"allow"` and `"ask"` are defined but not functional
- no context injection via hook stdout (unlike Claude Code's `SessionStart` and `UserPromptSubmit`)
- no dedicated approval-interception hook (unlike Claude Code's `PermissionRequest`)
- no environment persistence mechanism (unlike Claude Code's `CLAUDE_ENV_FILE`)
- `agentStop` and `subagentStop` lack detailed input/output schema documentation
- `toolArgs` is a JSON string, not a nested object, requiring double-parse in hook scripts
- session storage schema is not publicly documented
- the tool is closed-source (`github/copilot-cli` is not an open-source repository)
