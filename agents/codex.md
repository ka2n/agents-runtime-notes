# Codex Integration

Last verified: 2026-04-08

This file summarizes Codex behavior relevant to authors building tools that integrate with Codex hooks, sessions, and transcripts.

## Official References

- Codex hooks: https://developers.openai.com/codex/hooks
- Codex config reference: https://developers.openai.com/codex/config-reference
- Codex open source config notes: https://github.com/openai/codex/blob/main/docs/config.md

## Relevant External Hook Surface

The current official Codex hooks documentation describes:

- hooks are under development
- hooks are behind `[features].codex_hooks = true`
- hooks are discovered from `~/.codex/hooks.json` and `<repo>/.codex/hooks.json`
- `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, and `Stop` are turn-scoped

Current documented hook events:

- `SessionStart`
- `UserPromptSubmit`
- `PreToolUse`
- `PostToolUse`
- `Stop`

The current docs also explicitly state that:

- `PreToolUse` currently only supports Bash tool interception
- `PostToolUse` currently only supports Bash tool results
- matcher support is limited, and `PreToolUse` / `PostToolUse` currently emit only `Bash`

## Important gap

As of the verification date above, Codex does not expose a dedicated approval-request hook event in the official hook surface used here.

Practical consequence:

- `PreToolUse` tells us Codex is about to run `Bash`
- it does not tell us whether execution will proceed immediately or stop on an approval prompt

This is the main difference from Claude Code when you are implementing approval-aware tooling.

## Execution and output model

Codex hooks are command-based.

Notable current behaviors:

- hooks receive one JSON object on `stdin`
- commands run in the session `cwd`
- plain text stdout is accepted as extra context for `SessionStart` and `UserPromptSubmit`
- `Stop` expects structured JSON on stdout when exiting `0`
- `PreToolUse` and `PostToolUse` are currently Bash-only in the public docs

## Subagent Detection

Codex session IDs are opaque identifiers. From hooks alone, a tool cannot reliably determine whether a session is a main agent session or a subagent session.

Current hook-surface limitation:

- a session ID by itself is not enough to classify a session
- the current hook surface does not include a documented `is_subagent` or
  `parent_session_id` field

Codex transcript files contain richer metadata than the hook stream.

Observed local path layout:

- `~/.codex/sessions/YYYY/MM/DD/rollout-<timestamp>-<session-id>.jsonl`

Observed `session_meta` fields for spawned subagents include:

- `forked_from_id`
- `source.subagent.thread_spawn.parent_thread_id`
- `source.subagent.thread_spawn.depth`
- `agent_nickname`
- `agent_role`

Practical design takeaway:

- hook-only classification is heuristic at best
- transcript-backed classification is the reliable path for subagent linkage

Example shape seen locally:

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

The parent session transcript may also contain spawn-side events such as `collab_agent_spawn_end` with `new_thread_id`, which can be used as a secondary cross-check.

## Recommended integration strategy

For Codex, the practical design is:

1. accept hook events as the low-latency source of session lifecycle and state
2. when a new session ID is first seen, look for the corresponding Codex
   session JSONL file
3. read only the initial `session_meta` entry
4. if `forked_from_id` or `source.subagent.thread_spawn.parent_thread_id`
   exists, mark the session as a subagent and record the parent session ID

This keeps the live path lightweight while still using the richer transcript
metadata when it becomes available.

## Hook-only fallback

Without transcript access, you can still build limited heuristics around:

- session-local state keyed by `session_id`
- `tool_name`
- `tool_input.command`
- environment variables such as `CODEX_THREAD_ID` when present

But this still does not prove parent-child direction between sessions.

## Known Constraints

- there is no dedicated approval-request hook event in the current public docs
- public tool interception is Bash-only today
- transcript layout is implementation-observed, not fully specified as a stable external contract
- if Codex adds first-class subagent or approval events, transcript-side inference can be reduced

## Upstream Tracking

These open issues are directly relevant for tool authors:

- `openai/codex#15311` Add blocking PermissionRequest hook for external approval UIs
- `openai/codex#16301` add permission request event for parity with Claude Code
- `openai/codex#16484` machine-readable event surface for approvals and turn lifecycle

## Integration guidance

- If you need low-latency live state, start from hooks.
- If you need subagent relationships, enrich from transcript parsing.
- If you need approval-aware tooling, design for the current gap that approval prompts are not first-class hook events.
- If `CODEX_THREAD_ID` is present in the runtime environment, use it as an additional correlation key, but not as a full replacement for transcript metadata.
