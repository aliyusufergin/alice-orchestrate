## Facts
- The end-to-end milestone is done; see `done.md` and `journal.md`. The trial scripts are in the session scratchpad and will not last: `e2e/mk.sh` builds the sample project, `e2e/run.sh <scenario>` runs a headless step. Rebuild them from this description if needed.
- Done means 2–4 were observed, with one gap waiting for the user's decision. Done 2 in its literal form ("Codex finds the planted bug") was not seen, because Claude found the planted bug first in both runs. What was seen instead: Codex found a different bug that Claude had missed, it was labelled "Codex", and it went into the journal.
- Codex 0.159.2 was used throughout.

## Proposal
Route 1: README and version tag.
- Install the plugin through a local marketplace (`/plugin marketplace add <path>`) to check done means 1, and confirm that `bin/` is on PATH for an installed plugin (Codex's review claimed it is not).
- In the README cover: setup, Codex login, when work goes to Codex, the minimum and the tested Codex version, and that the plugin is not to be installed alongside alice-skills.
- Tag v0.1.0 locally. Publishing to GitHub waits for the user.
