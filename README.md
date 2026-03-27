# datastar-skills

An [agent skill specification](https://agentskills.io/specification) for [Datastar](https://data-star.dev).

## About

Core Datastar skill covering backend-driven architecture, signal design, `data-*` attribute usage, SSE patterns, and CQRS. Language-specific backend references live under `datastar/references/`.

For best results you may want to include your project's backend language in your prompt or `AGENTS.md` file.

## Installation

To install, use the Open Agent Skills tool from Vercel Labs:

```sh
npx skills add vimcommando/datastar-skills
```

Repository: https://github.com/vercel-labs/skills

If you prefer a manual install, copy the `datastar` skill into your agent's skills directory.

### Codex

Codex reads skills from `$CODEX_HOME/skills` (default: `~/.codex/skills`).

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
cp -R datastar "${CODEX_HOME:-$HOME/.codex}/skills/datastar"
```

Restart Codex after installing.

### Claude Code

Project-local install:

```bash
mkdir -p .claude/skills
cp -R datastar .claude/skills/datastar
```

Global install:

```bash
mkdir -p ~/.claude/skills
cp -R datastar ~/.claude/skills/datastar
```

### OpenCode

Project-local install:

```bash
mkdir -p .opencode/skills
cp -R datastar .opencode/skills/datastar
```

Global install:

```bash
mkdir -p ~/.config/opencode/skills
cp -R datastar ~/.config/opencode/skills/datastar
```

Restart OpenCode after installing.

## References

Forked from https://gist.github.com/njreid/29032a171ec88c4fe8da1b09e2bac196
