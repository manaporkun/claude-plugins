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
| 2. ANALYZE|  External agent (Gemini/Codex) reviews the plan
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

### From GitHub

```bash
git clone <repo> /tmp/do-skill
cp -r /tmp/do-skill/do ~/.claude/skills/do
```

### Manual

Copy the `do/` directory to `~/.claude/skills/`.

## Usage

```
/do Add a dark mode toggle to the settings page
/do Fix the race condition in the WebSocket handler
/do Refactor the auth middleware to use JWT
```

## Configuration

Optionally create `.claude/do-config.json` in your project root:

```json
{
  "reviewAgents": ["gemini"],
  "reviewAgentCommands": {
    "gemini": "cat {file} | gemini -p \"Review the content via stdin.\" -o text",
    "codex": "codex exec \"$(cat {file})\""
  },
  "qc": {
    "test": "npm test",
    "build": "npm run build",
    "lint": "eslint ."
  },
  "maxIterations": 3
}
```

### Config Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `reviewAgents` | string[] | auto-detect | External agents for plan/code review |
| `reviewAgentCommands` | object | built-in | Custom invocation commands per agent. Use `{file}` as placeholder for the prompt file path. |
| `qc.test` | string | auto | Test command |
| `qc.build` | string | auto | Build command |
| `qc.lint` | string | auto | Lint command |
| `maxIterations` | number | 3 | Max QC fix iterations |

Without a config file, the skill auto-detects available agents and project type.

## Requirements

- Claude Code
- At least one external agent CLI (optional but recommended):
  - [Gemini CLI](https://github.com/google-gemini/gemini-cli) — `npm i -g @google/gemini-cli`
  - [Codex CLI](https://github.com/openai/codex-cli) — `brew install codex`

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
do/
├── SKILL.md              # Main workflow orchestrator
├── README.md             # This file
└── prompts/
    ├── plan-review.md    # Template: external agent plan review
    └── code-review.md    # Template: external agent code review
```

## How It Works

1. **Plan**: Claude researches the codebase and creates a step-by-step plan, saved to `.claude/plans/`
2. **Analyze**: The plan is sent to an external agent (Gemini/Codex) for independent review. If issues are found, the plan is revised.
3. **Approve**: You review the plan and external feedback, then approve.
4. **Implement**: Claude spawns subagents to implement the plan in isolated contexts.
5. **QC**: Automated tests/builds run first. Then an external agent reviews the code diff for plan compliance and quality.
6. **Present**: A summary of all changes, QC results, and outstanding items is presented for your final approval.

## License

MIT
