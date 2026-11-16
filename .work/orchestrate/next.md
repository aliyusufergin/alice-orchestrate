## Facts
- `bin/alice-codex` (about 150 lines of bash) provides three commands: `check`, `review <job> [repo]` (read-only) and `work <job> <repo>` (detached temporary worktree, produces `changes.patch`).
  - Every job writes `status` (ok, unavailable, failed, timeout or quota) and uses exit codes 0, 3, 4 and 5. Usage is documented in the script header.
  - It was tested with a stub codex in every failure mode and with real Codex 0.159.2: a review found a planted bug in 15 s, and a work run returned a patch that applies in about 25 s. No worktree, branch or process was left behind. Re-run the stub tests after changing the script; see `.work/knowledge.md`.
- Skills are still byte-identical to alice-skills 0.3.6. Nothing calls the launcher yet.
- It is unverified that `bin/` lands on the Bash PATH once the plugin is installed. Check that before skills rely on it, with `ls` or `${CLAUDE_PLUGIN_ROOT}` as the fallback.
- Job directories must not be in `/tmp`, because Codex can write there. Use `<repo>/.git/alice-codex/<job>`.

## Proposal
Route item 1: wire the worker into `step` (the independent check goes to Codex when available; well-defined sub-tasks can be delegated, and their patch is reviewed and applied by Claude), `survey` (reading can be shared) and `ask-alice` (explain the workers). Fallback and telling the user must follow the spec's commitments. Keep the additions as short as the rest of the skills; the skill text should say what good looks like and leave the how to the model.
