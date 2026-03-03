---
name: setup
version: 2.0.0
argument-hint: "없음"
description: |
  Command Center 워크스페이스 초기 세팅.
  필수 도구 확인, GH 인증, 프로젝트 clone을 단계별로 수행합니다.
allowed-tools:
  - "Bash(curl *)"
  - "Bash(brew *)"
  - "Bash(winget *)"
  - "Bash(apt *)"
  - "Bash(yum *)"
  - "Bash(gh *)"
  - "Bash(git *)"
  - "Bash(which *)"
  - "Bash(java *)"
  - "Bash(docker *)"
  - "Bash(uname *)"
  - "Bash(command *)"
  - Read
  - Glob
  - AskUserQuestion
---

# setup

Command Center 워크스페이스 초기 세팅을 단계별로 수행한다.

## 설정 파일

워크스페이스 루트의 `.claude/workspace.json`에서 설정을 읽는다.

워크스페이스 루트 감지: 현재 디렉토리 또는 상위에서 `.claude/workspace.json`이 있는 디렉토리를 찾는다.

## 실행 절차

아래 단계를 **순서대로** 실행한다. 각 단계 완료 시 `{항목} : 완료 ✅` 형식으로 출력한다.

### 0단계: OS 감지

`uname -s`로 운영체제를 감지한다:
- `Darwin` → macOS
- `Linux` → Linux
- `MINGW*` / `MSYS*` → Windows (Git Bash)

감지된 OS를 이후 설치 명령 분기에 사용한다.

### 1단계: 필수 도구 확인

아래 도구를 하나씩 확인하고, 없으면 OS별 설치 안내를 제공한다:

| 도구 | 확인 | macOS | Linux (apt) | Windows |
|------|------|-------|-------------|---------|
| git | `which git` | `brew install git` | `sudo apt install git` | `winget install Git.Git` 또는 https://git-scm.com |
| gh | `which gh` | `brew install gh` | `sudo apt install gh` | `winget install GitHub.cli` 또는 https://cli.github.com |
| JDK 21 | `java -version` | `brew install openjdk@21` | `sudo apt install openjdk-21-jdk` | https://adoptium.net 에서 JDK 21 설치 안내 |
| Docker | `docker info` | Docker Desktop 설치 안내 | `sudo apt install docker.io` | Docker Desktop 설치 안내 |

각 도구마다:
1. `which {도구}` 또는 해당 확인 명령 실행
2. 있으면 → `{도구} : 완료 ✅` 출력
3. 없으면 → OS에 맞는 설치 명령 또는 설치 링크를 안내. macOS에서는 `brew`가 있으면 자동 설치, 없으면 brew 먼저 설치 안내.

**JDK 버전 확인**: `java -version` 출력에서 버전 번호를 파싱한다. 21 이상이면 통과. 21 미만이면 업그레이드를 안내한다.

**타임아웃**: 설치 Bash 명령에 `timeout: 300000` (5분)을 설정한다.

### 2단계: GH 인증

1. `.claude/workspace.json`에서 `gh_host` 값을 읽는다.
2. `gh auth status --hostname {gh_host}` 로 인증 상태 확인
3. 인증됨 → `GH 인증 ({gh_host}) : 완료 ✅` 출력
4. 미인증 → `gh auth login --hostname {gh_host}` 실행 (`timeout: 120000`). 브라우저 인증을 안내.

### 3단계: 프로젝트 clone

1. `.claude/workspace.json`에서 `projects` 객체를 읽는다.
2. projects가 비어있으면 → "등록된 프로젝트가 없습니다. workspace.json에 프로젝트를 추가하거나 `/sync-projects`를 사용하세요." 출력 후 건너뜀.
3. projects가 있으면 각 프로젝트에 대해 clone 또는 확인.

### 완료

모든 단계가 끝나면:

```
=== 세팅 완료 ===
```

## 주의사항

- 각 단계를 **하나씩** 실행하고, 실패하면 원인을 파악하여 사용자에게 안내한다.
- 설치 도중 에러가 나면 멈추고 사용자에게 상황을 설명한다.
- 이미 완료된 항목은 재실행하지 않고 `완료 ✅` 만 출력한다.
