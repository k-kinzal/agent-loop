# agent-loop

Bash scripts to run CLI-based AI agents (Claude Code, Codex CLI, Gemini CLI, Copilot CLI, OpenCode, etc.) in an automated loop.

## Design philosophy

CLI-based AI agents consume their context window during a single session. As the conversation grows longer, performance degrades. agent-loop solves this by **terminating and restarting the agent process each iteration**, ensuring every run starts with a fresh context.

### How the loop works

1. The wrapper script launches the agent process
2. The agent works autonomously and eventually exits on its own
3. The wrapper evaluates termination conditions (max iteration count, completion keyword detected)
4. If conditions are not met, the agent is **launched as a new process** (not a session continuation)
5. If conditions are met or the agent exited abnormally, the loop ends

Each iteration starts the agent as a brand-new session with no prior conversation history. The agent picks up previous work through the project's filesystem (code, docs, git history, etc.).

## Structure

- `claude-loop` — Loop wrapper for Claude Code, using its Stop hook mechanism to detect agent completion

## Usage

```bash
claude-loop --max-iteration 5 "implement feature X"
claude-loop --completion-promise "DONE" "work until done"
claude-loop --debug --max-iteration 3 -p "say hello"
```

## Dependencies

- `claude` CLI
- `jq`

## Development guidelines

- Add each agent CLI loop as a separate file (e.g. `codex-loop`, `gemini-loop`)
- Write scripts in bash with minimal external dependencies
- Always backup and restore existing settings files
- The loop is driven by the wrapper script restarting the agent process, not by agent-internal continuation mechanisms
