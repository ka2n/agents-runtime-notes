# AGENTS.md

## Purpose

This repository is a runtime reference for coding-agent integrations.

When editing docs here, optimize for:

- separating observed local behavior from vendor-documented behavior
- separating transcript paths from transcript format details
- separating canonical history from derived metadata caches

## Documentation Policy

- Put cross-agent comparisons in `references/`.
- Put agent-specific behavior in `agents/<agent>.md`.
- Use `agents/<agent>.md` for session-file format details, field notes, caveats, and integration guidance.

## Evidence Policy

- Prefer direct local observation for file layout and sample payload shapes.
- Prefer upstream OSS code for confirming field names, schema, migrations, and storage responsibilities.
- Explicitly label statements as `Observed`, `Confirmed in OSS`, or practical interpretation when certainty differs.
- Do not present implementation-observed paths or fields as stable vendor contracts unless officially documented.

## Editing Style

- Keep sections short and reference-oriented.
- Prefer bullets over long prose.
- Add concrete example shapes only when they clarify structure.
- Keep examples minimal and redact incidental noise when the schema is the important part.
- Update `Last verified` when re-checking local behavior or upstream code.

## Research Workflow

1. Check existing docs first to avoid duplicating sections.
2. Inspect local files for real examples.
3. Inspect upstream code for schema, migrations, and storage intent.
4. Inspect relevant agent-management or agent-observability tools for prior art when comparison is useful.
5. Write down observed facts first, then interpretation.
6. Keep uncertain claims narrow.

## External References

- Use `external-docs/` for cloned upstream repositories and other reference materials needed for investigation.
- Clone upstream agent implementations there when source inspection is useful for confirming schemas, migrations, storage roles, or behavior.
- Clone agent-management or agent-observability tools there when they are useful as prior art or comparison material.
- Prefer shallow clones unless deeper history is required.
- Treat `external-docs/` as supporting evidence, not as the primary authored content location.
