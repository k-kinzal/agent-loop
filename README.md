# agent-loop

Single-file Bash wrappers for running CLI-based AI agents in a restart loop.

The core idea is to terminate the agent process after each iteration and start
it again as a brand-new process. This keeps every iteration on a fresh session
while letting the agent pick up previous work from the filesystem.

## Why

Long-lived CLI agent sessions eventually accumulate context noise. Restarting
the process each iteration keeps the conversation window fresh and makes the
wrapper, not the agent, responsible for loop control.

## Included tools

- `claude-loop`: Claude Code wrapper using Claude Stop hooks
- `codex-loop`: Codex CLI wrapper for iterative process restarts
- `gemini-loop`: Gemini CLI wrapper for iterative process restarts

## Installation

Each wrapper is self-contained so it can be installed directly with `curl` or
`wget`.

```bash
curl -fsSLO https://example.com/codex-loop
chmod +x codex-loop
```

You can also clone the repository and install whichever scripts you need.

```bash
git clone <repo-url>
cd agent-loop
chmod +x claude-loop codex-loop gemini-loop
```

## Loop behavior

All wrappers follow the same high-level model:

1. Launch the target CLI as a new process.
2. Wait for the CLI to exit.
3. Stop if the maximum iteration count or completion promise was reached.
4. Otherwise start a new process for the next iteration.

This means interactive/TUI modes only advance the loop after the underlying CLI
exits. If the upstream CLI keeps an interactive session open indefinitely, the
wrapper cannot force a new iteration until that session ends.

## Wrapper options

The generic wrappers use these loop controls:

- `--max-iteration N`
- `--completion-promise TEXT`
- `--debug`

Everything else is passed through to the wrapped CLI.

If the wrapped CLI has an option name that conflicts with a wrapper option, use
`--` to stop wrapper parsing and pass the remaining arguments through verbatim.

```bash
codex-loop --max-iteration 5 -- exec --full-auto "implement feature X"
codex-loop -- --help
```

## Tool notes

### `claude-loop`

- Depends on `claude` and `jq`
- Uses Claude's Stop hook mechanism, so it can keep Claude working without
  relying on process exit alone

### `codex-loop`

- Depends on `codex`
- Uses plain process restarts rather than Codex-specific hooks
- Interactive Codex sessions launch correctly, but reliable autonomous looping
  requires a Codex mode that exits on completion, such as `codex exec`

### `gemini-loop`

- Depends on `gemini`
- Uses plain process restarts rather than Gemini-specific hooks
- Gemini positional prompts are one-shot by default, which fits the loop model
- `-i/--prompt-interactive` launches an interactive session, but the wrapper
  can only continue after that session exits

## Development

- Keep each wrapper in its own standalone script
- Do not require shared library files for runtime
- Preserve and restore user settings when a wrapper edits them
- The loop must be controlled by the wrapper restarting the agent process, not
  by continuing a single agent session in memory
