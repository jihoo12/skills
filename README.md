# skills

English | [한국어](./README.ko.md)

A collection of skills for AI coding agents such as Claude Code, Codex, and OpenCode.

## Skills

| Skill | Description |
| --- | --- |
| [`nix-flake-dev`](./nix-flake-dev/SKILL.md) | Makes the agent always use Nix for development and manage every development dependency in the `flake.nix` devShell. |

## Installation

Clone this repository first:

```bash
git clone https://github.com/jihoo12/skills.git ~/src/skills
```

Each skill is a folder containing a `SKILL.md`. Link or copy it into the skills directory of the agent you use.

### Claude Code

Claude Code reads skills from `~/.claude/skills/` (all projects) or from a project's `.claude/skills/` (that project only).

```bash
# Apply to all projects
mkdir -p ~/.claude/skills
ln -s ~/src/skills/nix-flake-dev ~/.claude/skills/nix-flake-dev

# Or apply to a single project
mkdir -p .claude/skills
cp -r ~/src/skills/nix-flake-dev .claude/skills/
```

### Codex

Codex reads skills from `~/.agents/skills/` (all projects) or from `.agents/skills/` in a repository (searched from the current directory up to the repository root). Symlinked skill folders are supported.

```bash
# Apply to all projects
mkdir -p ~/.agents/skills
ln -s ~/src/skills/nix-flake-dev ~/.agents/skills/nix-flake-dev

# Or apply to a single project
mkdir -p .agents/skills
cp -r ~/src/skills/nix-flake-dev .agents/skills/
```

Codex picks up skill changes automatically; if a skill does not appear, restart Codex.

### OpenCode

OpenCode reads skills from `~/.config/opencode/skills/` (all projects) or from a project's `.opencode/skills/` (that project only).

```bash
# Apply to all projects
mkdir -p ~/.config/opencode/skills
ln -s ~/src/skills/nix-flake-dev ~/.config/opencode/skills/nix-flake-dev

# Or apply to a single project
mkdir -p .opencode/skills
cp -r ~/src/skills/nix-flake-dev .opencode/skills/
```

OpenCode also reads `.claude/skills/` and `.agents/skills/` (both global and per project), so a skill installed for Claude Code or Codex works in OpenCode without a separate install.

## nix-flake-dev

With this skill, the agent:

- Creates a `flake.nix` first if the project has none, and extends the existing one in its own style if it does.
- Runs every command inside the dev shell with `nix develop -c <command>`.
- Adds missing tools and system libraries to `packages` in `flake.nix` instead of using `apt`, `brew`, `npm -g`, `pip install --user`, `rustup`, or `curl | sh`.
- Leaves library dependencies to each language's package manager and lockfile (`package.json`, `Cargo.toml`, `pyproject.toml`, and so on), while Nix provides the toolchain and system dependencies.
- Runs one-off tools that are not part of the project with `nix run nixpkgs#<pkg>` / `nix shell nixpkgs#<pkg> -c`, without installing them.
- Tells the user when Nix is not installed, instead of working around it with another install method.

Layout:

```
nix-flake-dev/
├── SKILL.md                 # the skill itself
├── assets/flake.nix         # minimal devShell template
└── references/languages.md  # per-language package lists and common pitfalls
```

## License

[Apache License 2.0](./LICENSE)
