## Facts
- The plugin skeleton exists: `alice-orchestrate` 0.1.0, forked from alice-skills 0.3.6. The skills and hooks are byte-identical to the source; only `.claude-plugin/*.json` and the README title and install lines were changed. `claude plugin validate .` passes, with one warning about a missing icon. The scale is still tiny, as the spec intends.
- The baseline commit is on `main`. `git diff <baseline>..` shows everything added since.
- Codex 0.159.2 is logged in with ChatGPT. Its flags are listed in `.work/knowledge.md`.

## Proposal
Route item 1: build the Codex launcher, with a read-only review mode and a write mode that runs in a worktree. It needs a timeout, cleanup of processes and worktrees, and detection of when no worker is available. Exercise it against real Codex in a throwaway git repo (for example under the scratchpad), including the failure cases from "done means" item 4. Find out what an exhausted quota looks like in exec output, or record that it can't be observed and how it will be simulated. This is the riskiest piece, so run the independent check.
