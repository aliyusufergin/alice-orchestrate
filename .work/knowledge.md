# Knowledge

## Workers
- Codex: the supported scripting surface is `codex exec` (`--json` JSONL events, `-o` last message, `--output-schema`, `-s read-only|workspace-write`, `-C`, `--worktree`, `--ephemeral`, `exec resume`, `exec review`). `codex mcp-server` is removed; `app-server` is experimental and version-specific. `--full-auto` is gone (`--approve-for-me` replaces it). Needs a git repo unless `--skip-git-repo-check`.
- Codex traps: hangs if stdin stays open or the stream stalls, so close stdin, enforce our own timeout and kill the whole process tree. There is no distinct exit code for a usage limit, so treat `turn.failed`/`error` or a non-zero exit as "worker unavailable". `exec review` ignores `--output-schema`.
- Google (Gemini CLI, Antigravity): third-party orchestration on a consumer login is not permitted. Use only an API key or Vertex.
- Other vendors (Copilot, Cursor, Vibe, Qwen, Grok) officially ship ACP servers. That is the path for adding them later.
- Launcher: `bin/alice-codex` (on Bash PATH when the plugin is installed). Job dirs must be outside `/tmp`/`$TMPDIR` and the worktree, since Codex's workspace-write sandbox may write there; `<repo>/.git/alice-codex/<job>` is a safe place.
- Testing: a stub `codex` on PATH (handles `--version`, `login status`, `exec ... -o FILE -C DIR -`) exercises every failure mode without quota. End-to-end: headless `claude -p "/alice-orchestrate:step ..." --plugin-dir <repo> --settings '{"enabledPlugins":{"alice-skills@alice-skills":false}}'` on a sample project with a spec and a planted bug, one run per scenario (real, missing, fail, hang, quota). Never `pkill -f` a pattern in the test shell's own command line.

## Claude Code
- Background Bash (`run_in_background`) notifies on exit, so it serves as a job queue without an MCP server.
- Plugin subagents ignore `hooks`, `mcpServers` and `permissionMode`.
- MCP output is capped at about 25k tokens, so return a summary plus a file path.
