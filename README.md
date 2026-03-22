# /do — Structured Task Workflow

A Claude Code skill that turns any task into a systematic, quality-controlled workflow with external agent review.

## Workflow

```
/do <task description>
     |
     v
+-----------+
| 1. PLAN   |  Research codebase, create implementation plan
+-----+-----+
      |
      v
+-----------+
| 2. ANALYZE|  External agent (Gemini/Codex/Ollama) reviews the plan
+-----+-----+
      |
      v
+-----------+
| 3. APPROVE|  User reviews plan + analysis, approves
+-----+-----+
      |
      v
+-----------+
| 4. BUILD  |  Subagents implement the approved plan
+-----+-----+
      |
      v
+-----------+
| 5. QC     |  Automated tests + external code review
+-----+-----+
      |
      v
+-----------+
| 6. PRESENT|  Summary of changes, QC results, final approval
+-----------+
```

## Installation

### As a Claude Code Plugin

```bash
claude plugin add manaporkun/task-workflow-skill
```

### Direct (symlink)

```bash
git clone https://github.com/manaporkun/task-workflow-skill.git ~/Documents/Projects/task-workflow-skill
cd ~/Documents/Projects/task-workflow-skill
./install.sh
```

This creates a symlink from `~/.claude/skills/do` to the repo's `skills/do/` directory, so updates via `git pull` take effect immediately.

## Usage

```
/do Add a dark mode toggle to the settings page
/do Fix the race condition in the WebSocket handler
/do Refactor the auth middleware to use JWT
```

### Environment Cache

The skill caches detected agent availability (Gemini, Codex, Ollama) at `~/.claude/do-env.json` so it doesn't re-run `which` checks on every invocation. The cache is created automatically on first run.

To force a re-detection (e.g. after installing or removing an agent CLI):

```
/do --refresh-env <task description>
```

Or delete the cache manually: `rm ~/.claude/do-env.json`

## Configuration

Optionally create `.claude/do-config.json` in your project root:

```json
{
  "agents": {
    "planReview": ["ollama:qwen2.5-coder", "gemini"],
    "codeReview": ["gemini", "codex"]
  },
  "agentCommands": {
    "gemini": "cat {file} | gemini -p \"Review the content via stdin.\" -o text",
    "codex": "cat {file} | codex exec -q -",
    "ollama": "cat {file} | ollama run {model}"
  },
  "qc": {
    "test": "npm test",
    "build": "npm run build",
    "lint": "eslint ."
  },
  "maxIterations": 3
}
```

### Agent Routing

Each phase can use a different review agent. The `agents` field maps phases to an ordered list of agents — the skill tries the first available and falls back to the next.

Agent format:
- `"gemini"` — Gemini CLI (cloud)
- `"codex"` — Codex CLI (cloud)
- `"ollama:<model>"` — Ollama with a local model (e.g. `"ollama:qwen2.5-coder"`)

This lets you use fast local models for plan review and more capable cloud models for code review.

### Config Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `agents.planReview` | string[] | auto-detect | Ordered agent list for plan review (Phase 2) |
| `agents.codeReview` | string[] | auto-detect | Ordered agent list for code review (Phase 4) |
| `agentCommands` | object | built-in | Custom invocation commands per agent. `{file}` = prompt file path, `{model}` = model name. |
| `qc.test` | string | auto | Test command |
| `qc.build` | string | auto | Build command |
| `qc.lint` | string | auto | Lint command |
| `maxIterations` | number | 3 | Max QC fix iterations |

Without a config file, the skill auto-detects available agents and project type.

## Requirements

- Claude Code
- At least one external agent CLI (optional but recommended):

| Agent | Install | Type | Best for |
|---|---|---|---|
| [Gemini CLI](https://github.com/google-gemini/gemini-cli) | `npm i -g @google/gemini-cli` | Cloud | Deep analysis, code review |
| [Codex CLI](https://github.com/openai/codex-cli) | `brew install codex` | Cloud | Code review, built-in `codex review` |
| [Ollama](https://ollama.com) | `brew install ollama` | Local | Fast plan review, small tasks |

### Recommended Ollama Models for Code Review

| Model | Size | Good for |
|---|---|---|
| `qwen2.5-coder` | 7B | Fast plan/code review |
| `deepseek-coder-v2` | 16B | Thorough code review |
| `codellama` | 7B | Lightweight review |
| `llama3` | 8B | General-purpose review |

Install a model: `ollama pull qwen2.5-coder`

## Supported Project Types

Auto-detection for:
- **Node.js** — package.json
- **iOS/macOS** — Podfile, *.xcodeproj
- **Python** — pyproject.toml, requirements.txt
- **Go** — go.mod
- **Rust** — Cargo.toml

For other project types, specify QC commands in `.claude/do-config.json`.

## File Structure

```
task-workflow-skill/
├── .claude-plugin/
│   └── plugin.json           # Plugin manifest
├── skills/
│   └── do/
│       ├── SKILL.md          # Main workflow orchestrator
│       └── prompts/
│           ├── plan-review.md    # Template: external agent plan review
│           └── code-review.md    # Template: external agent code review
├── install.sh                # Symlink installer for direct use
├── CHANGELOG.md              # Version history
├── README.md                 # This file
└── LICENSE
```

## How It Works

1. **Plan**: Claude researches the codebase and creates a step-by-step plan, saved to `.claude/plans/`
2. **Analyze**: The plan is sent to an external agent (Gemini/Codex/Ollama) for independent review. If issues are found, the plan is revised.
3. **Approve**: You review the plan and external feedback, then approve.
4. **Implement**: Claude spawns subagents to implement the plan in isolated contexts.
5. **QC**: Automated tests/builds run first. Then an external agent reviews the code diff for plan compliance and quality.
6. **Present**: A summary of all changes, QC results, and outstanding items is presented for your final approval.

## License

MIT
