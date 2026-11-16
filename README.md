# alice-orchestrate

[alice-skills](https://github.com/aliyusufergin/alice-skills) with Claude as the orchestrator and other models as its workers. The first worker is OpenAI Codex, reached only through its official non-interactive CLI and your own ChatGPT login.

Intent-first Claude Code skills for working on code or anything else. They state what good looks like and leave the how to the model. They stay firm only at the boundaries: who may change what, and what one agent hands to the next.

```text
/clarify → /survey → /spec → /step → /step → ... → done
                               ↑
                             /steer
```

Every skill is optional. Small, clear work needs none of them.

| Skill | Purpose |
|---|---|
| `clarify` | Understand and widen the intent, asking only what matters |
| `survey` | Learn from existing tools and neighbouring problems |
| `spec` | Write the smallest spec a fresh agent can work from safely |
| `step` | Continue the work in verified steps, keeping the next one ready |
| `steer` | Fold a new idea or change of mind into work underway |
| `ask-alice` | Map of the flow, the record and how to explain both to the user |

The skills hold no domain knowledge. What the work learns about its domain, including how work there is checked, goes to `.work/knowledge.md`.

Plans are rewritten as the work teaches something. The user owns intent, commitments, "not doing" and "done means". Agents own the route.

Before adding a line to a skill, ask whether the model would go wrong without it.

## Workers

Claude stays in charge of the work and its record. When Codex is installed and logged in, Claude can give it:

- **the independent check** at milestones, at the end and at risky steps, since a different model misses different things;
- **a second opinion** on a decision that would change the outcome;
- **a well-defined part of a step.** Codex works in a temporary git worktree and returns a patch, which Claude reviews and applies. Your working tree and `.work/` are never written by the worker.

`survey` can also share its reading with Codex. Talking with you (`clarify`, `spec`, `steer`) stays with Claude.

Every job uses your ChatGPT plan's limits, so Claude uses Codex where a second model adds something, not on every step. If Codex is missing, not logged in, fails, times out (15 minutes by default) or is out of quota, Claude does the work itself and tells you why. Everything works without Codex.

Claude reaches Codex through `bin/alice-codex`, which runs `codex exec` read-only for reviews and in a temporary worktree for work, and leaves no worktree or process behind. Run it without arguments for its interface. The timeout is set with `ALICE_CODEX_TIMEOUT` (in seconds) and the model with `ALICE_CODEX_MODEL`.

The skills do not change your Claude model. Use the strongest one available as the orchestrator.

## Requirements

- Claude Code on Linux or macOS. Windows is not supported.
- For workers: [Codex CLI](https://github.com/openai/codex) 0.159 or later, logged in with `codex login`. Tested with codex-cli 0.159.2.
- Git, for delegated work. In a project without git, Codex only reviews.

## Install

In Claude Code:

```text
/plugin marketplace add aliyusufergin/alice-orchestrate
/plugin install alice-orchestrate@alice-orchestrate
```

Install this or alice-skills, not both: they share skill names, so which one runs would be unpredictable. Projects can switch between them, because the `.work/` record is the same.

## Hooks

The plugin runs a small shell script (`hooks/compact.sh`) around each compaction. When the project has work in `.work/`, it reminds the summary and the agent that the record there is the source of truth; otherwise it prints nothing. It needs `sh`, so on Windows it works only with Git Bash or WSL.

## Recommended setting

Add this to `~/.claude/settings.json` so that `step` can suggest compacting at the end of a step, once more than 150,000 tokens of context are in use:

```json
"totalTokensReminder": "countdown"
```

It shows the agent how many tokens are left in the context window. The setting is undocumented and may change; without it, `step` simply never suggests compacting.

## License

MIT
