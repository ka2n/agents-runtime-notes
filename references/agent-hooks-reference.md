# Agent Hook References

Last verified: 2026-04-10

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

- GitHub Copilot CLI notes
  - [`agents/copilot-cli.md`](../agents/copilot-cli.md)

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

## GitHub Copilot CLI

- GitHub Copilot CLI documentation
  - https://docs.github.com/en/copilot/how-tos/use-copilot-agents/use-copilot-cli
  - Primary usage documentation for the standalone `copilot` CLI.

- GitHub Copilot CLI hooks concept
  - https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-hooks
  - Lists all 8 hook events. `agentStop` and `subagentStop` described here but lack detailed schemas.

- GitHub Copilot CLI hooks how-to
  - https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/use-hooks
  - Practical guide for creating hooks with `.github/hooks/*.json`.

- GitHub Copilot CLI hooks reference
  - https://docs.github.com/en/copilot/reference/hooks-configuration
  - Full input/output schemas for 6 of 8 events. Key points:
    - `preToolUse` supports `permissionDecision`, but only `"deny"` is currently processed
    - `toolArgs` is a JSON string (not a nested object), requiring double-parse
    - no matcher support; scripts must filter by `toolName` internally

- GitHub Copilot CLI configuration
  - https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/configure-copilot-cli
  - Configuration reference including MCP servers, permissions, and trusted folders.

- GitHub Copilot CLI custom instructions
  - https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-custom-instructions
  - Supports `.github/copilot-instructions.md`, `.github/instructions/**/*.instructions.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`.

- GitHub Copilot CLI repository
  - https://github.com/github/copilot-cli
  - Closed-source repository. Releases and issue tracking.

- Old `gh copilot` extension (archived)
  - https://github.com/github/gh-copilot
  - Deprecated 2025-10-25. The `gh copilot suggest` and `gh copilot explain` commands are no longer functional.

- GitHub Copilot CLI Chronicle (session data)
  - https://docs.github.com/en/copilot/concepts/agents/copilot-cli/chronicle
  - Session insights system: standup generation, tips, session store.

- GitHub Copilot CLI MCP servers
  - https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-mcp-servers
  - Adding and managing MCP server configurations.

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
