# Agent Hook References

Last verified: 2026-04-09

This file is the index of upstream references and related material.

## In-repo references

- Hooks patterns reference
  - [`references/hooks-patterns.md`](hooks-patterns.md)
  - Cross-agent hook types, execution model, environment variables, input JSON examples, and implementation patterns.

- Session file paths reference
  - [`references/session-file-paths.md`](session-file-paths.md)
  - Transcript locations and filesystem discovery patterns across agents.

- Prior art
  - [`references/prior-art.md`](prior-art.md)
  - External tools and projects that are useful comparison material for session discovery and multi-agent tooling.

- Claude Code notes
  - [`agents/claude-code.md`](../agents/claude-code.md)

- Codex notes
  - [`agents/codex.md`](../agents/codex.md)

## Claude Code

- Anthropic Claude Code hooks reference
  - https://code.claude.com/docs/en/hooks
  - Primary source for hook events, matcher behavior, and structured outputs.

- Anthropic Claude Code hooks guide
  - https://code.claude.com/docs/en/hooks-guide
  - Practical examples for notifications, formatting, and logging.

## Codex

- Codex hooks
  - https://developers.openai.com/codex/hooks
  - Key current points:
    - hooks are under development
    - hooks require `[features].codex_hooks = true`
    - `PreToolUse` and `PostToolUse` currently only expose `Bash`
    - no dedicated approval-request hook is documented in the current public surface

- Codex config reference
  - https://developers.openai.com/codex/config-reference
  - Documents `features.codex_hooks` and related runtime configuration.

- Codex open source config notes
  - https://github.com/openai/codex/blob/main/docs/config.md
  - Useful for implementation-adjacent details and historical context.

## Other agents

- Gemini CLI repository
  - https://github.com/google-gemini/gemini-cli
  - Useful as a reference point for transcript-first integrations where hooks are not documented.

## Related upstream issues and implementation notes

- https://github.com/openai/codex/issues/15311
- https://github.com/openai/codex/issues/16301
- https://github.com/openai/codex/issues/16484

- https://github.com/openai/codex/pull/10096
  - Merged 2026-02-03
  - Injects `CODEX_THREAD_ID` into the terminal environment when applicable.
