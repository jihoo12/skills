# Per-language dev shell notes

Starting points for the `packages` list in `flake.nix`, and the problems that
come up most often. Confirm attribute names against the pinned nixpkgs with
`nix eval nixpkgs#<attr>.version` before relying on them; names change between
releases.

## Contents

- [Node.js / TypeScript](#nodejs--typescript)
- [Python](#python)
- [Rust](#rust)
- [Go](#go)
- [C / C++](#c--c)
- [JVM (Java, Kotlin, Scala)](#jvm-java-kotlin-scala)
- [Native libraries and pkg-config](#native-libraries-and-pkg-config)
- [Local services (databases etc.)](#local-services-databases-etc)

## Node.js / TypeScript

```nix
packages = with pkgs; [ nodejs_24 pnpm ];   # or yarn-berry, bun, deno
```

- Library dependencies stay in `package.json` and its lockfile. CLIs the project
  uses (`tsc`, `eslint`, `vite`) belong in `devDependencies` and are run with
  `npx`/`pnpm exec`, not installed globally.
- Pick the Node.js major version this way:
  - An existing project: use the `nodejs_<major>` attribute that matches
    `engines` / `.nvmrc` / `.node-version`.
  - A new project: use an LTS line. nixpkgs currently has `nodejs_24` and
    `nodejs_26`; `nodejs_24` is the safe default for broad library
    compatibility, and `nodejs_26` is fine when the dependencies support it.
    Avoid odd-numbered majors (never LTS) and `nodejs_22` or older for new
    work. Check what exists before writing it down:
    `nix eval nixpkgs#nodejs_26.version`.
- Use the `nodejs_<major>` attribute, not the floating `nodejs`. Remove
  `.nvmrc`-based instructions from docs once the flake is the source of truth,
  or keep the two in agreement.
- npm packages that download prebuilt binaries (Prisma engines, Playwright
  browsers, some `sharp`/`esbuild` setups) may fail on NixOS because the
  binaries expect a conventional Linux layout. Prefer the nixpkgs build
  (`prisma-engines`, `playwright-driver.browsers`) and point the tool at it
  with the environment variable its docs describe.

## Python

```nix
packages = with pkgs; [ python314 uv ];
```

- Pick the Python version this way:
  - An existing project: use the `python3<minor>` attribute that matches
    `requires-python` in `pyproject.toml` / `.python-version`.
  - A new project: use `python314`. Versions in security-fix-only mode
    (`python312` and older) are not for new work. A just-released version
    (`python315` at the moment) is fine once the dependencies ship wheels for
    it. Check what exists before writing it down:
    `nix eval nixpkgs#python314.version`.
- Let `uv` (or poetry/pdm) own the virtualenv and `pyproject.toml` + lockfile;
  Nix supplies the interpreter and the system libraries. Tell uv to use the Nix
  interpreter instead of downloading its own:
  ```nix
  env.UV_PYTHON_DOWNLOADS = "never";
  env.UV_PYTHON = "${pkgs.python314}/bin/python";
  ```
- Never `pip install` outside a virtualenv; the Nix store is read-only and it
  will fail or land in `~/.local`.
- Wheels with compiled extensions (numpy, torch, opencv) may fail at import on
  NixOS with `libstdc++.so.6: cannot open shared object file` or similar. Add
  the libraries they need to the loader path:
  ```nix
  env.LD_LIBRARY_PATH = pkgs.lib.makeLibraryPath [ pkgs.stdenv.cc.cc.lib pkgs.zlib ];
  ```
  On non-NixOS Linux and on macOS this is usually unnecessary.
- For small scripts with few dependencies, a Nix-built interpreter is simpler:
  `(python314.withPackages (ps: with ps; [ requests pyyaml ]))`.

## Rust

```nix
packages = with pkgs; [ cargo rustc clippy rustfmt rust-analyzer ];
env.RUST_SRC_PATH = "${pkgs.rustPlatform.rustLibSrc}";
```

- This gives the stable toolchain from nixpkgs. Do not add `rustup` alongside
  it. If the project needs a specific version, nightly, or extra targets
  (it has a `rust-toolchain.toml`), add `fenix` or `rust-overlay` as a flake
  input and build the toolchain from that file, so the file stays the single
  source of truth.
- Crates linking C libraries need `pkg-config` plus the library (commonly
  `openssl`). See the native libraries section.

## Go

```nix
packages = with pkgs; [ go gopls golangci-lint ];
```

- Modules stay in `go.mod` / `go.sum`. Tools installed with `go install` land in
  `~/go/bin` and are unrecorded; take them from nixpkgs instead when available.
- cgo needs a C compiler; `mkShell` provides one by default.

## C / C++

```nix
packages = with pkgs; [ cmake ninja pkg-config gdb clang-tools ];
buildInputs = with pkgs; [ openssl zlib ];   # libraries to link against
```

- `mkShell` already includes a C/C++ compiler (gcc on Linux, clang on macOS).
  Use `pkgs.mkShell.override { stdenv = pkgs.clangStdenv; }` to switch.
- Put build tools in `packages` and link-time libraries in `buildInputs`; the
  compiler wrapper finds headers and libraries from `buildInputs`
  automatically.
- `clang-tools` must come from the same LLVM as the compiler for `clangd` to
  find standard headers.

## JVM (Java, Kotlin, Scala)

```nix
packages = with pkgs; [ jdk21 gradle ];   # or maven, sbt, kotlin
env.JAVA_HOME = "${pkgs.jdk21}";
```

- Gradle/Maven wrappers (`./gradlew`) download their own build tool, which is
  fine, but the JDK should come from the flake. Disable Gradle toolchain
  auto-download if it tries to fetch another JDK.

## Native libraries and pkg-config

When a build fails with "could not find `<lib>`", "Package `<x>` was not found
in the pkg-config search path", or a missing header:

```nix
packages = with pkgs; [ pkg-config ];
buildInputs = with pkgs; [ openssl sqlite libpq ];
```

When a *prebuilt* binary fails at runtime with a missing `.so`, the library has
to be on `LD_LIBRARY_PATH` (see the Python section). Find which package ships a
file with `nix-locate` from `nix-index` (`nix run nixpkgs#nix-index` builds the
index once) or by searching <https://search.nixos.org/packages>.

macOS frameworks are provided by the default SDK in current nixpkgs; adding
`darwin.apple_sdk.frameworks.*` is no longer needed and those attributes have
been removed.

## Local services (databases etc.)

Put the server binaries in `packages` (`postgresql_16`, `redis`, `mysql84`) and
keep their data inside the project, ignored by git:

```nix
packages = with pkgs; [ postgresql_16 ];
shellHook = ''
  export PGDATA="$PWD/.dev/postgres"
  export PGHOST="$PWD/.dev"
'';
```

Then `initdb` once and start with `pg_ctl -o "-k $PGHOST" start`. For several
services at once, consider `process-compose` or `services-flake` rather than a
growing `shellHook`. If the project already uses Docker Compose for services,
keep that and take the `docker`/`docker-compose` clients from the flake.
