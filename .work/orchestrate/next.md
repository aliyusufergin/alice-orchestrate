## Facts
- The launcher `bin/alice-codex` is done and tested; see `done.md` and the header of the script. Under `--plugin-dir` it is on the Bash PATH.
- The skills now use workers:
  - `step` has a Workers section. The independent check uses a worker when one is available, and a well-defined part can go to a worker as a `work` job; its `changes.patch` is reviewed and applied with `git apply`.
  - `survey` can share its reading with a worker.
  - `ask-alice` explains workers to the user.
  - About 18 lines were added in total; `git diff d6d29f5 -- skills/` shows them.
- No end-to-end run of the skills themselves has happened yet. Only the launcher has been tested.

## Proposal
MILESTONE (Route 1): an end-to-end trial on a sample git project with a planted bug, observing done means 2–4.
- A real `/step` should send its independent check to Codex, catch the bug, and record it in the journal with a "Codex" label.
- A delegated sub-task should come back as a patch that Claude applies.
- The fallback cases should end with Claude alone: no codex on PATH, a stub that fails, a timeout and a quota error. Nothing may be left behind.

The user may prefer to run this interactively with `claude --plugin-dir <this repo>`. Otherwise run it headless with `claude -p` and report what was observed.
