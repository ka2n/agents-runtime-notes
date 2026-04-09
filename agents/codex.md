# Codex Integration

Last verified: 2026-04-09

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

## Session File Format

Codex session transcripts are JSONL files.

Observed local layout:

- first line is typically a `session_meta` record
- later lines are mixed rollout items such as `event_msg`, `response_item`, `turn_context`, `function_call`, and `function_call_output`

Observed first-line shape:

```json
{
  "timestamp": "2026-04-09T01:05:35.033Z",
  "type": "session_meta",
  "payload": {
    "id": "<thread-id>",
    "forked_from_id": "<optional-parent-thread-id>",
    "timestamp": "2026-04-09T01:02:54.781Z",
    "cwd": "/path/to/repo",
    "originator": "codex-tui",
    "cli_version": "0.118.0",
    "source": "cli",
    "model_provider": "openai",
    "base_instructions": { "text": "..." }
  }
}
```

Fields confirmed in the open-source code for `SessionMeta` include:

- `id`
- `forked_from_id`
- `timestamp`
- `cwd`
- `originator`
- `cli_version`
- `source`
- `agent_nickname`
- `agent_role`
- `agent_path`
- `model_provider`
- `base_instructions`
- `dynamic_tools`
- `memory_mode`

Important practical details:

- `agent_role` accepts `agent_type` as a serde alias in the OSS protocol type
- model and reasoning effort are not stored on `session_meta`; the state extractor derives those from `turn_context`
- title and `first_user_message` are derived from `event_msg.user_message`, not from `session_meta`

## Additional Local Metadata

Codex stores more than transcript JSONL files under `~/.codex/`.

## Storage Direction

The current open-source implementation appears to be in a hybrid phase, not a full SQLite-only design.

What the code makes clear:

- rollout JSONL files are still persisted under `~/.codex/sessions/...`
- the `codex-state` crate describes SQLite as metadata extracted from JSONL rollouts and mirrored locally
- startup and recovery include a backfill pass from rollout files into SQLite
- some runtime paths still explicitly read persisted rollout JSONL, including fork-related flows

Practical interpretation:

- JSONL is still the canonical conversation history today
- SQLite is a derived local index and state cache for fast listing, resume, linkage, and auxiliary metadata
- Codex is moving toward stronger SQLite usage, but the current code does not support claiming that JSONL has already been fully replaced

This is also hinted by comments in the rollout code that refer to parity checks during the SQLite migration phase.

### State SQLite

Observed local file:

- `~/.codex/state_5.sqlite`

The open-source runtime creates a versioned state DB filename as:

- `state_<STATE_DB_VERSION>.sqlite`

Observed responsibilities of the state DB:

- index threads for listing and resume
- cache normalized metadata extracted from rollout JSONL
- store dynamic tool specs
- track backfill progress
- track parent-child spawn edges

Observed tables of highest relevance for session tooling:

- `threads`
- `thread_dynamic_tools`
- `thread_spawn_edges`
- `stage1_outputs`
- `backfill_state`

Observed `threads` columns include:

- `id`
- `rollout_path`
- `created_at`
- `updated_at`
- `source`
- `model_provider`
- `cwd`
- `title`
- `sandbox_policy`
- `approval_mode`
- `tokens_used`
- `archived`
- `git_sha`
- `git_branch`
- `git_origin_url`
- `cli_version`
- `first_user_message`
- `agent_nickname`
- `agent_role`
- `memory_mode`
- `model`
- `reasoning_effort`
- `agent_path`

Observed `thread_spawn_edges` shape:

```text
parent_thread_id TEXT NOT NULL
child_thread_id  TEXT NOT NULL PRIMARY KEY
status           TEXT NOT NULL
```

This is the clearest local metadata source for parent-child linkage when it exists.

### Logs SQLite

Observed local file:

- `~/.codex/logs_1.sqlite`

The open-source runtime creates a separate versioned logs DB filename as:

- `logs_<LOGS_DB_VERSION>.sqlite`

Observed responsibilities of the logs DB:

- store runtime logs separately from thread metadata
- reduce lock contention against the main state DB
- index logs by timestamp, `thread_id`, and `process_uuid`

Observed `logs` columns include:

- `ts`
- `ts_nanos`
- `level`
- `target`
- `feedback_log_body`
- `module_path`
- `file`
- `line`
- `thread_id`
- `process_uuid`
- `estimated_bytes`

Practical design takeaway:

- use JSONL for canonical conversation history
- use `state_*.sqlite` for fast lookup, listing, and spawn linkage
- use `logs_*.sqlite` only for diagnostics and correlation, not as the primary conversation source

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
