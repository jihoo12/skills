# skills

[English](./README.md) | 한국어

AI 코딩 에이전트(Claude Code, Codex, OpenCode 등)를 위한 스킬 모음입니다.

## 스킬 목록

| 스킬 | 설명 |
| --- | --- |
| [`nix-flake-dev`](./nix-flake-dev/SKILL.md) | 개발할 때 항상 Nix를 사용하고, 모든 개발 의존성을 `flake.nix`의 devShell로 관리하도록 합니다. |

## 설치

먼저 이 저장소를 클론합니다.

```bash
git clone https://github.com/jihoo12/skills.git ~/src/skills
```

각 스킬은 `SKILL.md`가 들어 있는 폴더입니다. 사용하는 에이전트의 스킬 디렉터리에 링크하거나 복사하면 됩니다.

### Claude Code

Claude Code는 `~/.claude/skills/`(모든 프로젝트) 또는 프로젝트의 `.claude/skills/`(해당 프로젝트만)에 있는 스킬을 읽습니다.

```bash
# 모든 프로젝트에 적용
mkdir -p ~/.claude/skills
ln -s ~/src/skills/nix-flake-dev ~/.claude/skills/nix-flake-dev

# 또는 특정 프로젝트에만 적용
mkdir -p .claude/skills
cp -r ~/src/skills/nix-flake-dev .claude/skills/
```

### Codex

Codex는 `~/.agents/skills/`(모든 프로젝트) 또는 저장소의 `.agents/skills/`(현재 디렉터리부터 저장소 루트까지 탐색)에 있는 스킬을 읽습니다. 심볼릭 링크로 연결한 스킬 폴더도 지원합니다.

```bash
# 모든 프로젝트에 적용
mkdir -p ~/.agents/skills
ln -s ~/src/skills/nix-flake-dev ~/.agents/skills/nix-flake-dev

# 또는 특정 프로젝트에만 적용
mkdir -p .agents/skills
cp -r ~/src/skills/nix-flake-dev .agents/skills/
```

Codex는 스킬 변경을 자동으로 감지합니다. 스킬이 보이지 않으면 Codex를 다시 시작하세요.

### OpenCode

OpenCode는 `~/.config/opencode/skills/`(모든 프로젝트) 또는 프로젝트의 `.opencode/skills/`(해당 프로젝트만)에 있는 스킬을 읽습니다.

```bash
# 모든 프로젝트에 적용
mkdir -p ~/.config/opencode/skills
ln -s ~/src/skills/nix-flake-dev ~/.config/opencode/skills/nix-flake-dev

# 또는 특정 프로젝트에만 적용
mkdir -p .opencode/skills
cp -r ~/src/skills/nix-flake-dev .opencode/skills/
```

OpenCode는 `.claude/skills/`와 `.agents/skills/`도(전역, 프로젝트 모두) 읽으므로, Claude Code나 Codex용으로 설치한 스킬을 따로 설치하지 않고 OpenCode에서도 쓸 수 있습니다.

## nix-flake-dev

이 스킬이 적용되면 에이전트는 다음과 같이 동작합니다.

- 프로젝트에 `flake.nix`가 없으면 먼저 만들고, 있으면 기존 구조를 그대로 확장합니다.
- 모든 명령을 `nix develop -c <명령>`으로 dev shell 안에서 실행합니다.
- 도구나 시스템 라이브러리가 없으면 `apt`, `brew`, `npm -g`, `pip install --user`, `rustup`, `curl | sh` 대신 `flake.nix`의 `packages`에 추가합니다.
- 라이브러리 의존성은 각 언어의 패키지 매니저와 lockfile(`package.json`, `Cargo.toml`, `pyproject.toml` 등)에 그대로 두고, Nix는 툴체인과 시스템 의존성을 맡습니다.
- 프로젝트에 속하지 않는 일회성 도구는 `nix run nixpkgs#<pkg>` / `nix shell nixpkgs#<pkg> -c`로 설치 없이 실행합니다.
- Nix가 설치되어 있지 않으면 다른 설치 방법으로 우회하지 않고 사용자에게 알립니다.

구성:

```
nix-flake-dev/
├── SKILL.md                 # 스킬 본문
├── assets/flake.nix         # 최소 devShell 템플릿
└── references/languages.md  # 언어별 packages 예시와 자주 겪는 문제
```

## 라이선스

[Apache License 2.0](./LICENSE)
