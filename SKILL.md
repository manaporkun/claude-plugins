---
name: do
description: >
  Structured task execution: plan, external analysis, approval, implement,
  quality control, present. Use /do <task description> for any feature, bug fix, or task.
disable-model-invocation: true
argument-hint: <task description>
effort: high
---

# /do — Structured Task Workflow

Execute the following phases strictly in order. Do not skip phases.
Present output to the user at each checkpoint marked with STOP.

## Environment

Available external agents: !`(which gemini >/dev/null 2>&1 && echo "gemini") ; (which codex >/dev/null 2>&1 && echo "codex")`
Project config: !`cat .claude/do-config.json 2>/dev/null || echo "none"`
Project indicators: !`ls -1 package.json Podfile *.xcodeproj pyproject.toml go.mod Cargo.toml Makefile 2>/dev/null || echo "unknown project type"`

### Configuration

If `.claude/do-config.json` exists, use its values. Schema:

```json
{
  "reviewAgents": ["gemini"],
  "reviewAgentCommands": {
    "gemini": "cat {file} | gemini -p \"Review the content provided via stdin. Respond in plain text.\" -o text",
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

The `reviewAgentCommands` field is optional — if omitted, use the default commands listed in Phase 2.
If no config file exists, auto-detect everything from the environment output above.

---

## Phase 1: PLAN

1. Analyze the task: **$ARGUMENTS**
2. Explore the codebase to understand relevant files, patterns, and architecture
   - Use the Explore subagent for broad research if the task scope is unclear
   - Use Grep/Glob for targeted lookups
3. Create an implementation plan containing:
   - **Task Summary**: What needs to be done and why
   - **Files to Modify/Create**: Each with a brief description of changes
   - **Implementation Steps**: Numbered, ordered, actionable
   - **Testing Strategy**: How to verify correctness
   - **Risks & Edge Cases**: Potential issues to watch for
4. Save the plan to `.claude/plans/<descriptive-slug>.md`

---

## Phase 2: EXTERNAL ANALYSIS

> If no external agent is available, skip to the CHECKPOINT below.

1. Read the prompt template from `${CLAUDE_SKILL_DIR}/prompts/plan-review.md`
2. Build the full review prompt by replacing the template placeholders:
   - `{TASK}` → the task description ($ARGUMENTS)
   - `{PLAN}` → the full plan content
   - `{CONTEXT}` → brief codebase context (key file paths, interfaces involved)
3. Write the assembled prompt to `/tmp/do-plan-review-${CLAUDE_SESSION_ID}.md`
4. Call the first available external agent (with a 120-second timeout):
   - **Gemini**: `timeout 120 sh -c 'cat /tmp/do-plan-review-${CLAUDE_SESSION_ID}.md | gemini -p "Review the implementation plan provided via stdin. Respond in plain text." -o text'`
   - **Codex**: `timeout 120 codex exec "$(cat /tmp/do-plan-review-${CLAUDE_SESSION_ID}.md)"`
   - **Custom**: If `reviewAgentCommands` is defined in config, use that command pattern with `{file}` replaced by the temp file path
   - If the agent call times out or fails, note the failure and continue to the checkpoint
5. Capture and analyze the feedback
6. If the feedback suggests significant improvements:
   - Revise the plan
   - Update the saved plan file
   - Note what changed and why

### CHECKPOINT — User Approval

Present to the user:
- The implementation plan (revised if applicable)
- External agent feedback summary (if available)
- Changes made based on feedback (if any)

**STOP. Ask the user to approve, reject, or request changes to the plan.**

- **If approved**: proceed to Phase 3.
- **If rejected with feedback**: revise the plan based on user feedback, update the saved plan file, re-run external analysis if the changes are significant, then present the revised plan again. Repeat until approved.
- **If rejected without feedback**: ask the user what they'd like changed before proceeding.

---

## Phase 3: IMPLEMENT

1. Review the approved plan
2. For each implementation unit:
   - Spawn an **Agent** (using the Agent tool) with a complete task description
   - Include the relevant plan section, file paths, and expected changes
   - The agent must verify its changes compile/parse correctly
3. For large tasks with independent units, spawn multiple agents in parallel using worktree isolation
4. After all agents complete, verify the full set of changes is coherent

---

## Phase 4: QUALITY CONTROL

### 4a — Automated Testing

Run QC commands based on project type (or config overrides):

| Type | Commands |
|---|---|
| Node.js | `npm test`, `npx playwright test` (if installed), `npm run build` |
| iOS/macOS | `xcodebuild -scheme <scheme> -destination 'platform=iOS Simulator,name=iPhone 16' build` |
| Python | `pytest`, `ruff check .` |
| Go | `go test ./...`, `go vet ./...` |
| Rust | `cargo test`, `cargo clippy` |
| Other | Check Makefile, CI config, or `.claude/do-config.json` for commands |

On failure: analyze the error, fix the issue, re-run the failing command.
Maximum **3 iterations** per failing command. If still failing after 3 attempts, report the failure and continue.

### 4b — External Code Review

> If no external agent is available, skip to Phase 5.

1. Generate a diff of all changes including new untracked files:
   - `git add -N .` (intent-to-add untracked files so they appear in diff)
   - `git diff HEAD` (captures both staged, unstaged, and new files)
2. Read the prompt template from `${CLAUDE_SKILL_DIR}/prompts/code-review.md`
3. Build the review prompt by replacing placeholders:
   - `{TASK}` → the task description ($ARGUMENTS)
   - `{PLAN}` → the approved plan content
   - `{DIFF}` → the full diff output
4. Write to `/tmp/do-code-review-${CLAUDE_SESSION_ID}.md`
5. Call external agent (same method and timeout as Phase 2)
6. For **Codex** specifically, you may also try `codex review` as an alternative
7. Analyze the feedback:
   - Fix **CRITICAL** issues immediately
   - Note **WARNING**s and fix if straightforward
   - Log **SUGGESTION**s but do not necessarily act on all
8. Re-run affected tests after fixes
9. Maximum **2 code review iterations**

---

## Phase 5: PRESENT

Present to the user:

1. **Changes Summary** — files modified/created with brief descriptions
2. **Plan Compliance** — checklist of plan items with completion status
3. **QC Results** — test pass/fail, build status, external review summary
4. **Outstanding Items** — any warnings, suggestions, or follow-up tasks

Ask the user for final approval.
