# Prior Art

Last verified: 2026-04-09

This file collects external projects that are useful reference points for building tooling around coding-agent sessions, transcripts, hooks, and multi-agent orchestration.

Use it for:

- session discovery patterns
- transcript browsing patterns
- multi-agent coordination ideas
- storage and indexing tradeoffs
- implementation comparisons before documenting new claims here

## agent-sessions

Description: Session discovery and transcript path survey tool across multiple coding agents.
URL: https://github.com/jazzyalex/agent-sessions

- transcript path conventions across agents
- session discovery normalization
- supports Claude Code, Codex, Copilot CLI, Gemini, OpenCode, Droid, Cursor, and OpenClaw
- Copilot CLI: dedicated `CopilotSessionDiscovery`, `CopilotSessionIndexer`, and `CopilotSessionParser` classes
- confirms Copilot CLI session path as `~/.copilot/session-state/<sessionId>.jsonl`

## CodeIsland

Description: Multi-agent management and visualization tool for coding-agent workflows.
URL: https://github.com/wxtsky/CodeIsland

- parent-child session handling
- agent coordination UX
- multi-agent workspace and runtime presentation
- supports 9 AI tools including Copilot CLI
- Copilot CLI: reads model info and chat messages from JSONL transcripts
- Copilot CLI: installs hooks via `~/.copilot/hooks/codeisland.json` (a global hooks path, separate from the per-repo `.github/hooks/`)

## Notes

- Treat prior art as inspiration and comparison material, not as an authority on vendor-internal behavior.
- Prefer upstream vendor docs and source code when confirming exact field names, schemas, or storage guarantees.
