# tbright — personal Claude Code plugin

A small [Claude Code](https://claude.com/claude-code) plugin carrying Tim Bright's personal skills under the `tbright:` namespace. The repo is both the marketplace and the plugin: install it once and the skills show up as slash commands.

## Install

From a terminal:

```bash
claude plugin marketplace add tab58/skills
claude plugin install tbright@tbright
```

Or inside a Claude Code session:

```
/plugin marketplace add tab58/skills
/plugin install tbright@tbright
```

Restart Claude Code (or run `/reload-plugins` if you have it) to pick up the skills.

## Skills

### `/tbright:explain <sentence describing what to explain>`

Writes a ~750-word opinionated article explaining a topic — or reasoning Claude did earlier in the conversation. The style is modeled on ["The Repository Pattern with Rich Domain Models"](https://dev.to/40percentironman/the-repository-pattern-with-rich-domain-models-21bh): open with the pain, show the naive version first, short code snippets, "what this affords you" instead of adjectives, one honest tension, and a closing section on how the ideas lock together.

```
/tbright:explain Explain why we split the searcher into a retrieval phase and a ranking phase
```

### `/tbright:plan <task description>`

Plans a task before any implementation. Enters plan mode, explores the codebase, and asks questions until it is 95%+ confident it understands the task — with multiple-choice options deliberately sampled from different areas of the probability distribution (the likely pick, a genuinely different alternative, and a long-shot) rather than four variants of the same answer. Ends by presenting the plan for approval.

```
/tbright:plan Add rate limiting to the document search endpoint
```

## Development

1. Edit skill files under `skills/<name>/SKILL.md` (new skills: add a new directory there).
2. Bump `version` in `.claude-plugin/plugin.json`.
3. `claude plugin update tbright@tbright`, then restart or `/reload-plugins`.

Validate structure with `claude plugin validate .` from the repo root.

## License

[MIT](LICENSE)
