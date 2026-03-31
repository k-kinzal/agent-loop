# agent-loop

Single-file Bash wrappers for running CLI-based AI agents in restart loops,
file watches, and file-list batches.

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
- `copilot-loop`: GitHub Copilot CLI wrapper for iterative process restarts
- `cursor-loop`: Cursor Agent CLI wrapper for iterative process restarts
- `opencode-loop`: OpenCode CLI wrapper for iterative process restarts
- `cline-loop`: Cline CLI wrapper for iterative process restarts
- `claude-files`: Claude Code wrapper for newline-delimited file lists
- `codex-files`: Codex CLI wrapper for newline-delimited file lists
- `gemini-files`: Gemini CLI wrapper for newline-delimited file lists
- `copilot-files`: GitHub Copilot CLI wrapper for newline-delimited file lists
- `cursor-files`: Cursor Agent CLI wrapper for newline-delimited file lists
- `opencode-files`: OpenCode CLI wrapper for newline-delimited file lists
- `cline-files`: Cline CLI wrapper for newline-delimited file lists
- `claude-watch`: Claude Code wrapper triggered by file changes
- `codex-watch`: Codex CLI wrapper triggered by file changes
- `gemini-watch`: Gemini CLI wrapper triggered by file changes
- `copilot-watch`: GitHub Copilot CLI wrapper triggered by file changes
- `cursor-watch`: Cursor Agent CLI wrapper triggered by file changes
- `opencode-watch`: OpenCode CLI wrapper triggered by file changes
- `cline-watch`: Cline CLI wrapper triggered by file changes

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
chmod +x claude-loop claude-watch codex-loop codex-watch gemini-loop gemini-watch copilot-loop copilot-watch cursor-loop cursor-watch opencode-loop opencode-watch cline-loop cline-watch
```

You can also install the file-list wrappers:

```bash
chmod +x claude-files codex-files gemini-files copilot-files cursor-files opencode-files cline-files
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

## File-list behavior

The `*-files` wrappers read newline-delimited file paths from `--files-from
PATH` or from stdin when no file list is specified. Each matching line launches
the target CLI as a fresh process, so every file is handled in a new session.

All `*-files` wrappers support:

- `--files-from PATH` (repeatable, use `-` for stdin)
- `--include` / `--exclude` glob filters
- `--max-files N`
- `--completion-promise TEXT`
- `{{file}}` and `{{files}}` template variables

## Tool notes

### `claude-loop`

- Depends on `claude` and `jq`
- Uses Claude's Stop hook mechanism, so it can keep Claude working without
  relying on process exit alone

### `claude-files`

- Depends on `claude` and `jq`
- Reads newline-delimited file lists from `--files-from` or stdin
- Uses Claude's Stop hook mechanism so both print and interactive modes can
  return control between files
- Supports `--include`/`--exclude`, `--max-files`, and `{{file}}` templates

### `claude-watch`

- Depends on `claude`, `jq`, and optionally `fswatch`
- Watches directories for file creation and modification, then runs Claude Code
- Supports `--include`/`--exclude` glob filters and can derive watch roots from
  `--include` when `--watch-dir` is omitted
- For home-relative patterns, prefer quoted `~` like `'~/.claude/projects/*/*.jsonl'`
  or double-quoted `$HOME/...`; do not single-quote `$HOME/...`
- Uses Claude's Stop hook mechanism with SIGTERM to ensure Claude exits after
  each trigger, supporting both `-p` and interactive modes
- Falls back to `find -newer` polling when `fswatch` is not installed
- Cooldown prevents over-triggering from rapid saves (default: 30 seconds)
- Ignores `.git/`, `.claude/`, and `.DS_Store` by default

### `codex-loop`

- Depends on `codex`
- Uses plain process restarts rather than Codex-specific hooks
- Interactive Codex sessions launch correctly, but reliable autonomous looping
  requires a Codex mode that exits on completion, such as `codex exec`

### `codex-files` / `cursor-files` / `opencode-files` / `cline-files`

- Read newline-delimited file lists from `--files-from` or stdin
- Start a fresh process per file and support `--include`/`--exclude`,
  `--max-files`, and `{{file}}` templates
- Reliable autonomous batching is best with `codex exec ...`,
  `cursor-agent --print ...`, `opencode run ...`, and `cline --oneshot ...`

### `gemini-loop`

- Depends on `gemini`
- Depends on `jq`
- Uses a temporary Gemini `AfterAgent` hook to end each session and restart
  Gemini as a fresh process
- Gemini positional prompts are one-shot by default, and interactive
  `-i/--prompt-interactive` sessions are also loopable via the hook

### `gemini-files`

- Depends on `gemini` and `jq`
- Reads newline-delimited file lists from `--files-from` or stdin
- Uses a temporary Gemini `AfterAgent` hook to return control after each file
- Supports both positional prompts and interactive `-i` sessions

### `copilot-loop`

- Depends on `copilot`
- Depends on `jq`
- Uses a project-local Copilot `agentStop` hook under `.github/hooks/`
- Reads Copilot's session transcript to match the completion promise
- Works for prompt mode and interactive sessions by terminating Copilot after
  each completed agent turn

### `copilot-files`

- Depends on `copilot` and `jq`
- Reads newline-delimited file lists from `--files-from` or stdin
- Uses a project-local Copilot `agentStop` hook under `.github/hooks/`
- Terminates Copilot after each file so the wrapper can move to the next entry

### `cursor-loop`

- Depends on `cursor-agent`
- Uses plain process restarts rather than Cursor-specific hooks
- Reliable autonomous looping is best with `cursor-agent --print ...`
- Interactive Cursor Agent sessions still launch, but the wrapper cannot start
  the next iteration until the session exits

### `opencode-loop`

- Depends on `opencode`
- Uses plain process restarts rather than OpenCode-specific hooks
- Reliable autonomous looping is best with `opencode run ...`
- The default OpenCode TUI still launches, but the wrapper cannot start the
  next iteration until that session exits

### `cline-loop`

- Depends on `cline`
- Uses plain process restarts rather than Cline-specific hooks
- Reliable autonomous looping is best with `cline --oneshot ...` or other
  headless flags such as `--yolo` or `--no-interactive`
- Interactive Cline still launches, but the wrapper cannot start the next
  iteration until that session exits

### `codex-watch`

- Depends on `codex` and optionally `fswatch`
- Watches directories for file changes, then runs Codex CLI
- Supports `--include`/`--exclude` glob filters and can derive watch roots from
  `--include` when `--watch-dir` is omitted
- Reliable autonomous watching requires `codex exec ...`
- Supports `--per-file` mode and `{{file}}`/`{{files}}` template variables

### `gemini-watch`

- Depends on `gemini`, `jq`, and optionally `fswatch`
- Supports `--include`/`--exclude` glob filters and can derive watch roots from
  `--include` when `--watch-dir` is omitted
- Uses a temporary Gemini `AfterAgent` hook to end each session after a trigger
- Supports both positional prompts and interactive `-i` sessions

### `copilot-watch`

- Depends on `copilot`, `jq`, and optionally `fswatch`
- Supports `--include`/`--exclude` glob filters and can derive watch roots from
  `--include` when `--watch-dir` is omitted
- Uses a project-local Copilot `agentStop` hook under `.github/hooks/`
- Terminates Copilot after each trigger via SIGTERM

### `cursor-watch`

- Depends on `cursor-agent` and optionally `fswatch`
- Supports `--include`/`--exclude` glob filters and can derive watch roots from
  `--include` when `--watch-dir` is omitted
- Reliable autonomous watching is best with `cursor-agent --print ...`

### `opencode-watch`

- Depends on `opencode` and optionally `fswatch`
- Supports `--include`/`--exclude` glob filters and can derive watch roots from
  `--include` when `--watch-dir` is omitted
- Reliable autonomous watching is best with `opencode run ...`

### `cline-watch`

- Depends on `cline` and optionally `fswatch`
- Supports `--include`/`--exclude` glob filters and can derive watch roots from
  `--include` when `--watch-dir` is omitted
- Reliable autonomous watching is best with `cline --oneshot ...`

## Development

- Keep each wrapper in its own standalone script
- Do not require shared library files for runtime
- Preserve and restore user settings when a wrapper edits them
- The loop must be controlled by the wrapper restarting the agent process, not
  by continuing a single agent session in memory
