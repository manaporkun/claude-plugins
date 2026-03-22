# Changelog

## [1.1.0] - 2026-03-22

### Security

- **Fix command injection in Codex invocation** — Replaced `codex exec "$(cat ...)"` with stdin-piped `cat ... | codex exec -q -` to prevent shell metacharacter expansion from prompt contents.
- **Scope `git add -N` to plan-relevant files** — Phase 4b no longer runs `git add -N .`, which could expose unrelated files (`.env`, credentials) in diffs sent to external agents.
- **Use `mktemp` for temp files** — Plan-review and code-review temp files now use randomized names via `mktemp` instead of predictable paths, with cleanup after use.

### Added

- **Input validation** — The skill now rejects empty `/do` invocations with a usage message instead of proceeding with no task.
- **Large diff truncation** — Phase 4b truncates diffs exceeding 15,000 lines to the most relevant files and notes the truncation in the review prompt.
- **Version field** in `plugin.json` for future plugin ecosystem compatibility.

## [1.0.0] - 2026-03-21

### Added

- Initial release: structured `/do` workflow with 6 phases (Plan, Analyze, Approve, Implement, QC, Present).
- Cached environment detection for Gemini, Codex, and Ollama agents.
- Per-phase agent routing via `.claude/do-config.json`.
- Prompt templates for plan review and code review.
- Symlink installer and Claude Code plugin manifest.
