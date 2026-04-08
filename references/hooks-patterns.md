# Hooks Patterns Reference

Last verified: 2026-04-08

This file is a practical reference for building projects that rely on agent hooks.

Official sources:

- Claude Code hooks reference: https://code.claude.com/docs/en/hooks
- Claude Code hooks guide: https://code.claude.com/docs/en/hooks-guide
- Codex hooks: https://developers.openai.com/codex/hooks
- Codex config reference: https://developers.openai.com/codex/config-reference

## At a glance

| Topic | Claude Code | Codex |
| --- | --- | --- |
| Enablement | Built into settings | `features.codex_hooks = true` |
| Main config locations | `~/.claude/settings.json`, `.claude/settings.json`, `.claude/settings.local.json` | `~/.codex/hooks.json`, `<repo>/.codex/hooks.json` |
| Hook transport | command, HTTP, prompt | command |
| Input delivery | JSON via stdin or HTTP body | JSON via stdin |
| Matcher model | tool names or event-specific matcher values | regex-like matcher per event, but current tool hooks only emit `Bash` |
| Permission-specific event | `PermissionRequest` and `Notification(permission_prompt)` | no dedicated permission-request event in current docs |
| Environment persistence | `CLAUDE_ENV_FILE` in `SessionStart`, `CwdChanged`, `FileChanged` | no equivalent documented |

## Claude Code hook types

Current documented Claude hook events:

- `SessionStart`
- `InstructionsLoaded`
- `UserPromptSubmit`
- `PreToolUse`
- `PermissionRequest`
- `PostToolUse`
- `Notification`
- `Stop`
- `SubagentStop`
- `PreCompact`
- `PostCompact`
- `CwdChanged`
- `FileChanged`
- `SessionEnd`

Important practical distinctions:

- `PreToolUse` runs before tool execution even if no permission dialog is needed.
- `PermissionRequest` runs only when Claude is about to show a permission dialog.
- `Notification` is useful for side-channel alerts and includes `permission_prompt`, `idle_prompt`, `auth_success`, and `elicitation_dialog`.
- `CwdChanged` and `FileChanged` are useful for environment reloading patterns.

Common Claude matchers you will use:

- Tool events: `Bash`, `Edit`, `Write`, `Read`, `Glob`, `Grep`, `Agent`, `WebFetch`, `WebSearch`, `AskUserQuestion`, `ExitPlanMode`, and MCP tool names
- `SessionStart`: `startup`, `resume`, `clear`, `compact`
- `PreCompact` and `PostCompact`: `manual`, `auto`
- `Notification`: notification type such as `permission_prompt`

## Codex hook types

Current documented Codex hook events:

- `SessionStart`
- `PreToolUse`
- `PostToolUse`
- `UserPromptSubmit`
- `Stop`

Current runtime limitations called out in the official docs:

- `PreToolUse` currently only supports `Bash`
- `PostToolUse` currently only supports `Bash`
- `UserPromptSubmit` ignores `matcher`
- `Stop` ignores `matcher`
- there is no dedicated approval-request hook event in the documented surface

Codex config notes:

- hooks are loaded from `hooks.json`
- `features.codex_hooks` is documented as under development and off by default
- default hook timeout is `600` seconds when not specified

## Execution model

### Claude Code

- Command hooks receive JSON on `stdin`
- HTTP hooks receive the same JSON in the request body
- `UserPromptSubmit` and `SessionStart` can inject plain text stdout into Claude context
- `PreToolUse` can allow, deny, ask, or defer
- exit code `2` is the blocking code for policy enforcement
- matching hooks run in parallel
- identical hook commands are deduplicated

### Codex

- Hooks are command-based and receive one JSON object on `stdin`
- Commands run with the session `cwd` as working directory
- plain text stdout is only accepted as extra context for `SessionStart` and `UserPromptSubmit`
- `Stop` expects JSON on stdout when exiting `0`
- `PreToolUse` can deny or otherwise shape Bash execution, but the current tool surface is only `Bash`

## Environment variables and runtime context

### Claude Code

Useful documented environment variables and runtime behavior:

- `CLAUDE_PROJECT_DIR`: absolute project root, useful for stable hook script paths
- `CLAUDE_ENV_FILE`: available in `SessionStart`, `CwdChanged`, and `FileChanged` for persisting environment variables into later Bash commands

Practical use cases:

- append `export` lines into `CLAUDE_ENV_FILE`
- activate toolchains when entering a repo directory
- reload env when `.envrc` or similar files change

Example:

```bash
#!/usr/bin/env bash
set -euo pipefail

if [ -n "${CLAUDE_ENV_FILE:-}" ]; then
  {
    echo 'export PATH="$PATH:$CLAUDE_PROJECT_DIR/node_modules/.bin"'
    echo 'export APP_ENV=development'
  } >> "$CLAUDE_ENV_FILE"
fi
```

### Codex

Codex does not currently document a dedicated environment-persistence hook file like `CLAUDE_ENV_FILE`.

The reliable runtime fields are instead carried in the JSON payload:

- `session_id`
- `transcript_path`
- `cwd`
- `hook_event_name`
- `model`
- `turn_id` on turn-scoped events

Documented Codex environment variables relevant around hooks and runtime setup:

- `CODEX_HOME`
- `CODEX_SQLITE_HOME`
- `CODEX_CA_CERTIFICATE`

Implementation-observed Codex hook/session variables:

- `CODEX_THREAD_ID`

Notes:

- `CODEX_THREAD_ID` is worth using in hook scripts and helper tooling when you need a stable link back to the current Codex thread.
- As of 2026-04-08, this variable is easier to verify from the open-source Codex implementation history than from the public hooks docs. The merged `openai/codex` PR #10096 states that Codex injects `CODEX_THREAD_ID` into the terminal environment when applicable.

## Input JSON examples

### Claude `PreToolUse`

Representative shape from the docs:

```json
{
  "session_id": "abc123",
  "transcript_path": "/Users/.../.claude/projects/.../session.jsonl",
  "cwd": "/Users/example/project",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf node_modules",
    "description": "Remove node_modules directory"
  },
  "tool_use_id": "toolu_123"
}
```

### Claude `PermissionRequest`

```json
{
  "session_id": "abc123",
  "transcript_path": "/Users/.../.claude/projects/.../session.jsonl",
  "cwd": "/Users/example/project",
  "permission_mode": "default",
  "hook_event_name": "PermissionRequest",
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf node_modules",
    "description": "Remove node_modules directory"
  },
  "permission_suggestions": [
    {
      "type": "addRules"
    }
  ]
}
```

### Claude `Notification`

```json
{
  "session_id": "abc123",
  "transcript_path": "/Users/.../.claude/projects/.../session.jsonl",
  "cwd": "/Users/example/project",
  "hook_event_name": "Notification",
  "message": "Claude needs your permission to use Bash",
  "title": "Permission needed",
  "notification_type": "permission_prompt"
}
```

### Codex `PreToolUse`

```json
{
  "session_id": "session_123",
  "transcript_path": "/Users/.../.codex/sessions/.../rollout-123.jsonl",
  "cwd": "/Users/example/project",
  "hook_event_name": "PreToolUse",
  "model": "gpt-5.4",
  "turn_id": "turn_123",
  "tool_name": "Bash",
  "tool_use_id": "tool_123",
  "tool_input": {
    "command": "go test ./..."
  }
}
```

### Codex `PostToolUse`

```json
{
  "session_id": "session_123",
  "transcript_path": "/Users/.../.codex/sessions/.../rollout-123.jsonl",
  "cwd": "/Users/example/project",
  "hook_event_name": "PostToolUse",
  "model": "gpt-5.4",
  "turn_id": "turn_123",
  "tool_name": "Bash",
  "tool_use_id": "tool_123",
  "tool_input": {
    "command": "go test ./..."
  },
  "tool_response": "{\"exit_code\":1,\"stdout\":\"...\",\"stderr\":\"...\"}"
}
```

## Output patterns

### Claude deny dangerous commands before execution

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Destructive command blocked by policy."
  }
}
```

### Claude auto-allow after rewriting input

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow",
      "updatedInput": {
        "command": "npm run lint"
      }
    }
  }
}
```

### Codex block a Bash command

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Direct deploy commands are blocked."
  }
}
```

### Codex continue the turn after `Stop`

```json
{
  "decision": "block",
  "reason": "Run one more pass over the failing tests."
}
```

## Implementation patterns

### 1. Policy gate before shell execution

Use when:

- you want to block destructive commands
- you want to stop direct deploys from local sessions
- you want to require wrappers such as `make test` instead of raw scripts

Best events:

- Claude: `PreToolUse` or `PermissionRequest`
- Codex: `PreToolUse`

Minimal shell pattern:

```bash
#!/usr/bin/env bash
set -euo pipefail

payload="$(cat)"
command="$(printf '%s' "$payload" | jq -r '.tool_input.command // ""')"

case "$command" in
  rm\ -rf*|git\ push\ --force*)
    printf '%s\n' 'Blocked by hook policy.' >&2
    exit 2
    ;;
esac
```

### 2. Post-edit formatter or linter

Use when:

- any edit should trigger formatting
- generated files need validation before the agent continues

Best events:

- Claude: `PostToolUse` with `Edit|Write|MultiEdit`
- Codex: `PostToolUse` for Bash commands that already ran formatting or generation

Claude example config:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs prettier --write"
          }
        ]
      }
    ]
  }
}
```

### 3. Session bootstrap

Use when:

- every session should load repo conventions
- you want dynamic context from issue trackers, release notes, or local docs
- you need environment bootstrapping

Best events:

- Claude: `SessionStart`
- Codex: `SessionStart`

Typical output:

- emit plain text or `additionalContext`
- keep it short and deterministic
- avoid expensive network calls unless cached

### 4. Idle and approval notifications

Use when:

- you want desktop notifications
- you want external audit logging for approvals

Best events:

- Claude: `Notification` and `PermissionRequest`
- Codex: no direct equivalent, so rely on app behavior or transcript-side heuristics

### 5. Directory-aware environment loading

Use when:

- monorepos require different toolchains
- `direnv`, `mise`, `nix`, or language version managers must follow `cd`

Best events:

- Claude: `CwdChanged` and `FileChanged`

Pattern:

- detect `.envrc`, `flake.nix`, `.tool-versions`, or language manifests
- append exports to `CLAUDE_ENV_FILE`
- keep the hook idempotent

## Design guidance

- Prefer deterministic hooks over long prompt instructions when the rule is mandatory.
- Treat `PreToolUse` in both systems as a guardrail, not a hard sandbox boundary.
- Keep hooks fast. Blocking hooks directly affect agent latency.
- Parse JSON defensively. Missing fields and future event expansion are likely.
- Use stable absolute paths or project-root-derived paths, not relative paths from the current shell directory.
- Separate policy hooks from convenience hooks so failures in notifications do not block real work.
- For Claude, use `exit 2` for enforcement. `exit 1` is usually non-blocking.
- For Codex, document the current limitation that tool interception is Bash-only.

## Suggested repo layout for future projects

```text
.claude/
  settings.json
  hooks/
    pre_tool_policy.sh
    post_tool_format.sh
    session_bootstrap.sh

.codex/
  hooks.json
  hooks/
    pre_tool_policy.sh
    post_tool_review.sh
    session_bootstrap.sh
```

Keep shared logic in ordinary scripts and keep `hooks.json` or `settings.json` thin.
