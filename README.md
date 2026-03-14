# Claude Code ↔ Codex Integration Plugin

A Claude Code plugin that integrates OpenAI Codex as a plan reviewer and code implementer.

## How It Works

1. You invoke `/codex <task>` in Claude Code
2. Claude explores the codebase and creates an implementation plan
3. The plan is sent to Codex for expert review via the `codex-plan-reviewer` subagent
4. Claude refines the plan based on Codex's feedback
5. The refined plan is sent to Codex for implementation via the `codex-implementer` subagent
6. Claude reviews the results and reports back

## Setup

1. Install the Codex SDK globally (if not already installed):
   ```bash
   npm install -g @openai/codex-sdk
   ```

2. Authenticate with Codex (one of the following):
   - **Already logged in**: If you've run `codex login` or are otherwise authenticated with the Codex CLI, no extra setup is needed — the SDK inherits your existing credentials.
   - **API key**: Alternatively, set `export CODEX_API_KEY=your-key-here` in your environment.

3. Load the plugin in Claude Code:
   ```bash
   claude --plugin-dir /path/to/claude-codex-plugin
   ```

## Usage

```
/codex Add a REST API endpoint for user authentication
```

## Components

| Component | Path | Purpose |
|-----------|------|---------|
| Skill | `skills/codex/SKILL.md` | `/codex` slash command — orchestrates the workflow |
| Plan Reviewer | `agents/codex-plan-reviewer.md` | Gathers context and sends plan to Codex for review |
| Implementer | `agents/codex-implementer.md` | Gathers context and sends task to Codex for implementation |
| Review Script | `scripts/codex-review.mjs` | Node.js bridge to Codex SDK for plan review |
| Implement Script | `scripts/codex-implement.mjs` | Node.js bridge to Codex SDK for code implementation |

## Requirements

- Node.js 18+
- Codex CLI authentication (via `codex login`) or `CODEX_API_KEY` env var
- Claude Code with plugin support
