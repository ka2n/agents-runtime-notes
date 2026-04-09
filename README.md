# Agent Runtime Notes

Last updated: 2026-04-09

This repository is a runtime reference for authors building tools that integrate with multiple coding agents such as Claude Code and Codex.

It focuses on:

- hook types and behavior
- hook payloads and outputs
- environment variables
- session transcript locations
- subagent detection hints
- implementation patterns for multi-agent tooling

## Recommended structure

- [`references/agent-hooks-reference.md`](references/agent-hooks-reference.md)
  - Official links and prior art
- [`references/hooks-patterns.md`](references/hooks-patterns.md)
  - Cross-agent patterns, examples, and design guidance
- [`references/session-file-paths.md`](references/session-file-paths.md)
  - Transcript and session file path reference across agents
- [`references/prior-art.md`](references/prior-art.md)
  - External reference projects for session discovery and multi-agent tooling
- [`agents/claude-code.md`](agents/claude-code.md)
  - Claude Code specific notes
- [`agents/codex.md`](agents/codex.md)
  - Codex specific notes

## Suggested information architecture

For this repository, the clean split is:

- `references/`
  - cross-agent material
  - official references
  - comparative tables
  - transcript path inventories
- `agents/`
  - one file per agent
  - only agent-specific facts and caveats

This keeps multi-agent tool authors from having to separate general patterns from vendor-specific details while reading.

## Suggested future additions

- `examples/`
  - minimal `hooks.json` and `.claude/settings.json` samples
- `schemas/`
  - normalized event and session models for tooling authors
- `notes/`
  - implementation observations that are useful but not yet stable enough to promote into the main reference
