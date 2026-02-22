# datastar-skills

An [agent skill specification](https://agentskills.io/specification) for [Datastar](https://data-star.dev).

## About

This repository organizes Datastar guidance using skill's progressive disclosure:

- Core cross-language guidance stays in `SKILL.md`.
- Language-specific backend implementation details live under `datastar/references/{language}.md`.

For best results you may want to include your project's backend language in your prompt or `AGENTS.md` file.

## Installation

From this repository root (`/Users/reno/Development/datastar-skills`), install the `datastar` skill into your agent's skills directory.

### Codex

Codex reads skills from `$CODEX_HOME/skills` (default: `~/.codex/skills`).

From the Codex App you can use the `skill-installer` skill:
```
$skill-installer install https://github.com/VimCommando/datastar-skills/datastar
```

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

Built from https://gist.github.com/njreid/29032a171ec88c4fe8da1b09e2bac196
