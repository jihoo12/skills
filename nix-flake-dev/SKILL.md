---
name: nix-flake-dev
description: Use Nix for all development work, with every development dependency declared in the project's flake.nix devShell. Use this skill whenever you are about to do anything that touches a development environment - starting or scaffolding a project, installing or upgrading a compiler, runtime, CLI tool, linter, formatter or system library, running builds, tests or dev servers, hitting a "command not found" or missing-library error, writing setup instructions or CI, or editing flake.nix / flake.lock / .envrc. Use it even when the user never mentions Nix, and before reaching for apt, brew, npm -g, pipx, rustup, nvm, asdf, or a curl-pipe-sh installer.
---

# Nix flake development

The development environment of a project lives in one place: the `devShells`
output of its `flake.nix`, pinned by `flake.lock`. Every tool the project needs
(compilers, runtimes, package managers, linters, CLIs, system libraries) is
declared there, and every command is run inside that shell.

The point is reproducibility. A tool installed with `apt`, `brew`, `npm -g` or a
curl installer exists only on this machine, at whatever version happened to be
current, and nobody else (a teammate, CI, the same user next month) gets the
same thing. A tool listed in `flake.nix` is the same for everyone and is
recorded in git. So when something is missing, the fix is an edit to
`flake.nix`, not an install command.

## The loop

1. **Look for `flake.nix`** at the project root before running anything.
2. **If there is none, create one.** Start from `assets/flake.nix` and fill in
   the packages this project needs (see `references/languages.md` for
   per-language starting points). Then `git add flake.nix`.
3. **Run every project command through the dev shell:**
   ```bash
   nix develop -c <command>        # e.g. nix develop -c cargo test
   ```
4. **When a tool or library is missing, add it to `packages` in `flake.nix`**
   and rerun. Do not install it any other way.
5. **Keep `flake.nix` and `flake.lock` committed together** with the change
   that needed them.

## Things that are easy to get wrong

**Each shell call starts fresh.** As an agent, your shell state does not carry
from one command to the next, so running `nix develop` on its own and then
`cargo build` in a later call does nothing useful. Prefix each command with
`nix develop -c`. For several steps, use
`nix develop -c bash -c 'step1 && step2'`. If `$IN_NIX_SHELL` is already set
(the user launched you from inside the shell, or direnv is active), the prefix
is unnecessary, but still harmless.

**Flakes only see files git knows about.** In a git repository, an untracked
`flake.nix` produces a confusing "path does not exist" error. Run
`git add flake.nix` (and `flake.lock` once it is generated) before the first
`nix develop`. Staging is enough; committing is the user's call.

**After editing `flake.nix`, the next `nix develop -c ...` picks it up.** There
is nothing to reload. A long-running process started under the old shell (a dev
server, a watcher) has to be restarted to see new tools.

**Finding the right package name.** Guessing attribute names wastes time:
```bash
nix search nixpkgs ripgrep            # search by name/description
nix eval nixpkgs#nodejs_24.version    # confirm an attribute exists and its version
```
Prefer version-pinned attributes (`nodejs_24`, `python312`, `jdk21`,
`postgresql_16`) over floating ones (`nodejs`, `python3`) so that a lock update
does not silently change a major version. Old pinned attributes are removed
from nixpkgs once the upstream version reaches end-of-life, so check that the
one you want still exists.

**Updating.** `nix flake update` moves every input and can change many tool
versions at once. Run it only when the user wants an update or a needed package
version is missing from the pinned nixpkgs, and say that you did it.

## What Nix manages and what it does not

Nix provides the **toolchain and system-level dependencies**: the things you
would otherwise install onto the machine. The project's **library dependencies**
stay with the language's own package manager and lockfile (`package.json` +
lockfile, `Cargo.toml`, `pyproject.toml` + `uv.lock`, `go.mod`), run from inside
the dev shell:

```bash
nix develop -c npm install        # fine: project-local, locked by package-lock.json
nix develop -c uv add requests    # fine: recorded in pyproject.toml / uv.lock
npm install -g typescript         # not fine: global, unrecorded -> add to flake.nix or devDependencies
pip install --user black          # not fine: same reason
```

The test: does the command change the machine outside the project directory, or
leave no trace in a committed file? If so, it belongs in `flake.nix` instead.

**One-off tools** that are not part of the project (inspecting a file with `jq`
once, trying a tool out) do not need to go into the flake. Use an ad-hoc shell,
which installs nothing permanently:

```bash
nix run nixpkgs#jq -- '.name' package.json
nix shell nixpkgs#imagemagick -c magick in.png out.jpg
```

If you find yourself using the same one-off tool repeatedly for project work,
that is the signal to move it into `flake.nix`.

## Beyond `packages`

Environment setup that the project needs also belongs in the dev shell, so that
it is reproducible too:

```nix
pkgs.mkShell {
  packages = with pkgs; [ nodejs_24 pnpm postgresql_16 ];
  env.DATABASE_URL = "postgres://localhost/dev";   # plain environment variables
  shellHook = ''
    export PATH="$PWD/node_modules/.bin:$PATH"
  '';
}
```

Keep `shellHook` short and quiet. It runs on every `nix develop -c`, so slow or
chatty hooks slow down and clutter every command. Never put secrets in
`flake.nix`; it is committed and copied into the world-readable Nix store.

**direnv (optional).** If the user uses direnv, an `.envrc` containing
`use flake` loads the shell automatically on `cd`. Add `.direnv/` to
`.gitignore`. Do not add this unprompted to a project that does not already use
it; offer it.

## Fitting into existing projects

- **A `flake.nix` already exists:** extend it in its own style (flake-parts,
  flake-utils, devenv, a custom layout). Do not rewrite it into the template's
  shape.
- **The project has `shell.nix`, devbox, or devenv without a flake:** it already
  has a Nix-managed environment. Use it as it is, and suggest moving to
  `flake.nix` rather than migrating on your own.
- **Documentation and CI:** setup instructions you write should say
  `nix develop`, not list packages to install. CI should run commands through
  `nix develop -c` as well, so that CI and local environments cannot drift.

## When Nix is not available or not working

- **`nix` is not installed:** stop and tell the user, pointing them to
  <https://nixos.org/download/>. Installing Nix changes the system (a `/nix`
  store, a daemon, shell profiles), so it is the user's decision, not something
  to do silently. Do not fall back to `apt`/`brew`/global installs to get
  unblocked; that defeats the purpose and leaves the machine in the state this
  workflow exists to avoid.
- **"experimental feature 'flakes' is disabled":** add
  `--extra-experimental-features 'nix-command flakes'` to the command, and tell
  the user they can make it permanent with
  `experimental-features = nix-command flakes` in `~/.config/nix/nix.conf`.
- **A package is missing from nixpkgs or is the wrong version:** say so, and
  propose options (another nixpkgs branch as a second input, an overlay, the
  upstream project's own flake) rather than quietly installing it another way.
- **The user explicitly asks for a non-Nix approach** for some step: do what
  they ask. This skill sets the default, not a restriction on the user.

## Reference files

- `assets/flake.nix` - minimal multi-platform dev shell template to copy.
- `references/languages.md` - starting `packages` lists and known pitfalls for
  Node.js, Python, Rust, Go, C/C++, and JVM projects, plus native libraries and
  local services. Read the section for the language at hand when creating a
  flake or debugging a build that cannot find a library.
