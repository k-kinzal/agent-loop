# agent-loop

Bash scripts to run CLI-based AI agents (Claude Code, Codex CLI, Gemini CLI, Copilot CLI, OpenCode, etc.) in an automated loop.

## Design philosophy

CLI-based AI agents consume their context window during a single session. As the conversation grows longer, performance degrades. agent-loop solves this by terminating and restarting the agent process each iteration, ensuring every run starts with a fresh context.

### How the loop works

1. The wrapper script launches the agent process
2. The agent works autonomously and eventually exits on its own
3. The wrapper evaluates termination conditions such as max iteration count or completion keyword detection
4. If conditions are not met, the wrapper launches the agent again as a new process
5. If conditions are met or the agent exited abnormally, the loop ends

Each iteration starts as a brand-new session. The agent picks up previous work through the project's filesystem, config files, and git state rather than through in-memory conversation history.

## Structure

- `claude-loop` — Loop wrapper for Claude Code using Stop hooks
- `codex-loop` — Standalone loop wrapper for Codex CLI using plain process restarts
- `gemini-loop` — Loop wrapper for Gemini CLI using AfterAgent hooks
- `copilot-loop` — Standalone loop wrapper for GitHub Copilot CLI using plain process restarts
- `cursor-loop` — Standalone loop wrapper for Cursor Agent CLI using plain process restarts
- `opencode-loop` — Standalone loop wrapper for OpenCode CLI using plain process restarts
- `cline-loop` — Standalone loop wrapper for Cline CLI using plain process restarts

## Distribution requirement

Every wrapper other than `claude-loop` should be implemented as a standalone script that can be fetched directly with `curl` or `wget` and run without any companion library file.

## Argument handling

- Wrapper-level loop options must be parsed before `--`
- All remaining arguments must pass through to the target CLI unchanged
- If a target CLI flag conflicts with a wrapper flag, users should be able to pass it through with `--`

## TUI expectation

- Launching an interactive/TUI mode is valuable and should be supported where practical
- However, the loop can only continue after the target CLI exits
- If an upstream CLI keeps its TUI session open instead of exiting automatically, document that autonomous looping is only reliable in that CLI's non-interactive or one-shot modes

## Development guidelines

- Add each agent CLI loop as a separate top-level file
- Write scripts in bash with minimal external dependencies
- Keep runtime behavior self-contained inside each script
- Always backup and restore existing settings files when a wrapper edits them
- The loop is driven by the wrapper script restarting the agent process, not by agent-internal continuation mechanisms
