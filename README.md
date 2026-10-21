# herdr-config

Personal [Herdr](https://github.com/eliird/herdr) configuration.

- `config/` — Herdr configuration and runtime files (`config.toml`, etc.).
- `skills/` — agent skills, installed into your coding agent's skills folder.

## Install

Clone the repo somewhere permanent:

```sh
git clone https://github.com/eliird/herdr-config.git ~/herdr-config
```

### Config

Copy (or symlink) the Herdr config files into `~/.config/herdr`:

```sh
mkdir -p ~/.config/herdr
cp -r ~/herdr-config/config/. ~/.config/herdr/
```

### Skills

Install the agent skills into your coding agent's skills directory. For
opencode:

```sh
mkdir -p ~/.config/opencode/skills
cp -r ~/herdr-config/skills/. ~/.config/opencode/skills/
```

For Claude, use `~/.claude/skills/` instead.

Note: the `herd-orchestrator` skill needs git >= 2.5 (worktrees) and creates a
git-excluded `.herd/` directory per repo.

Restart Herdr / your agent so the new config and skills are picked up.
