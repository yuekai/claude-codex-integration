# Claude Code Codex Plugin

## Context

The user wants a Claude Code plugin that integrates OpenAI Codex as a collaborative partner. Claude handles planning, Codex reviews the plan and performs the actual implementation. This leverages Codex's strength at sandboxed code execution while keeping Claude's orchestration and reasoning abilities in the loop.

## Plugin Structure

```
/Users/yuekai/claude-codex-plugin/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── skills/
│   └── codex/
│       └── SKILL.md             # /codex slash command - orchestrates workflow
├── agents/
│   ├── codex-plan-reviewer.md   # Subagent: sends plan to Codex for review
│   └── codex-implementer.md     # Subagent: sends task to Codex for implementation
├── scripts/
│   ├── codex-review.mjs         # Node.js bridge: Codex SDK for plan review
│   └── codex-implement.mjs      # Node.js bridge: Codex SDK for implementation
├── package.json                 # Dependencies (@openai/codex-sdk)
└── README.md
```

## Workflow (Single-Shot with Rich Context)

Each Codex invocation is a single, well-informed call. Claude gathers all necessary context upfront so Codex can act without needing clarification.

1. User runs `/codex <task description>`
2. Claude explores the codebase (Read, Grep, Glob) and creates a detailed implementation plan
3. Claude invokes `codex-plan-reviewer` — the subagent gathers relevant file contents and codebase context, bundles them with the task + plan, and sends everything to Codex for review
4. Codex returns a single comprehensive review (issues, suggestions, verdict)
5. Claude refines the plan based on Codex's feedback
6. Claude invokes `codex-implementer` — the subagent gathers relevant source files and context, bundles them with the task + refined plan, and sends everything to Codex for implementation
7. Codex implements the changes and returns summary + git diff
8. Claude reviews the results and reports to the user

## Implementation Steps

### Step 1: Initialize project
- Create `package.json` with `@openai/codex-sdk` dependency, `"type": "module"`
- Run `npm install`

### Step 2: Create plugin manifest
- **File**: `.claude-plugin/plugin.json`
- Fields: name, version, description

### Step 3: Create bridge scripts
Both scripts read JSON from **stdin** (`{task, plan, context}`) to avoid shell argument length limits.

**Input JSON schema**: `{task, plan, context?}`
- `task`: the original task description
- `plan`: the implementation plan
- `context`: optional; relevant file contents and codebase info gathered by the subagent, so Codex has everything it needs in one shot

**Output**: plain text (Codex's response). Implementation script also appends git diff.

Scripts:
- **`scripts/codex-review.mjs`**: Sends a plan review prompt with full context to Codex, returns review feedback
- **`scripts/codex-implement.mjs`**: Sends an implementation prompt with full context to Codex, then captures `git diff` and untracked files
- Both include an `extractText()` helper that handles multiple Codex SDK response shapes with fallbacks

### Step 4: Create subagent definitions

**`agents/codex-plan-reviewer.md`**
- Tools: `Bash, Read, Glob, Grep`
- Model: `inherit`
- Instructions: gather relevant file contents and codebase context using Read/Glob/Grep, construct JSON payload with `task`, `plan`, and `context`, pipe to `codex-review.mjs` via heredoc, return Codex's feedback
- Read-only — does not modify files

**`agents/codex-implementer.md`**
- Tools: `Bash, Read, Glob, Grep`
- Model: `inherit`
- Instructions: gather relevant source files mentioned in the plan using Read/Glob/Grep, construct JSON payload with `task`, `plan`, and `context`, pipe to `codex-implement.mjs` via heredoc, return summary + diff
- May verify changes with Read/Glob/Grep after Codex finishes

### Step 5: Create the skill

**`skills/codex/SKILL.md`**
- User-invocable, argument-hint: `<task description>`
- Allowed tools: `Read, Grep, Glob, Bash, Agent`
- Orchestration instructions for the single-shot workflow (plan → review → refine → implement → verify)
- Error handling: if Codex is unavailable, Claude can proceed on its own

### Step 6: Create README
- Setup instructions (OPENAI_API_KEY, npm install)
- Usage: `/codex <task>`
- Workflow overview

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Stdin for script input | Plans can exceed shell arg limits; stdin has no practical limit |
| Separate review/implement scripts | Single responsibility; different prompts and post-processing |
| `extractText()` fallback chain | Codex SDK is new; response shape may vary across versions |
| `model: inherit` on agents | Avoids hardcoding; uses whatever model the parent is using |
| Git diff capture in implementer | Lets Claude review actual changes without re-reading every file |
| Single-shot with rich context | Subagents gather relevant files before calling Codex, so it has everything it needs without back-and-forth. Simpler and more reliable than multi-turn loops. |
| Graceful degradation | Skill handles Codex failures by offering Claude-only fallback |

## Verification

1. Run `npm install` and confirm `@openai/codex-sdk` installs
2. Test review script: `echo '{"task":"test","plan":"say hello"}' | node scripts/codex-review.mjs .`
3. Test implement script similarly
4. Load plugin with `claude --plugin-dir ./` and verify `/codex` appears in skill list
5. Run `/codex <simple task>` end-to-end and verify both subagents are invoked
