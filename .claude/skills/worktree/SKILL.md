---
name: worktree
version: 1.0.0
argument-hint: "<setup|create|list|switch|status|done|remove> [name]"
description: "projects/ 내 코드 레포의 Git worktree 자동화. 트리거: '/worktree setup', '/worktree create', '/worktree list', '/worktree remove', '/worktree done', '/worktree switch', '/worktree status'. workspace 구조(main/ + worktrees/)로 격리된 기능 개발을 지원."
allowed-tools:
  # git - worktree 관리 핵심
  - Bash(git worktree:*)
  - Bash(git branch:*)
  - Bash(git checkout:*)
  - Bash(git rev-parse:*)
  - Bash(git -C:*)
  - Bash(git log:*)
  - Bash(git status:*)
  - Bash(git remote:*)
  # 파일시스템 - worktree 생성/이동/정리
  - Bash(pwd:*)
  - Bash(basename:*)
  - Bash(dirname:*)
  - Bash(mkdir:*)
  - Bash(cp:*)
  - Bash(mv:*)
  - Bash(ls:*)
  - Bash(test:*)
  - Bash(find:*)
  # 빌드 - worktree 생성 후 환경 세팅
  # gradlew는 worktree 내부에서 실행됨. workspace root에서는 git -C <worktree> ./gradlew 형태로 실행.
  - Bash(./gradlew:*)
  - Read
  - Glob
  - Grep
  # 사용자 확인 — setup/remove/done에서 필수
  - AskUserQuestion
---

# Worktree 스킬

Git worktree를 **workspace 구조**로 관리하여 Claude Code의 디렉토리 제약을 해결한다.

## 핵심 개념

```
workspace/              ← Claude 세션 시작점 (primary working directory)
├── main/               ← git repo 본체
└── worktrees/          ← 모든 워크트리
    ├── feature-x/      ← Read("worktrees/feature-x/src/...") ✅
    └── feature-y/      ← Glob(path="worktrees/feature-y") ✅
```

Claude Code의 파일 도구(Read/Edit/Glob/Grep)는 세션 시작 디렉토리 기준으로 동작한다. 워크트리를 하위 디렉토리에 두면 별도 `cd` 없이 모든 워크트리에 접근할 수 있다.

## 동작 규칙

### 반드시 수행
- 모든 명령 전에 환경 감지: `main/.git` 존재 → WORKSPACE_MODE / git repo 내부 → `/worktree setup` 제안
- workspace mode: `git -C main worktree add ../worktrees/<name>` 으로 생성
- 워크트리 생성 시 환경 파일을 `main/`에서 복사 (`.claude/config.json` → `workspace.envFiles` 참조)
- 워크트리 생성 시 빌드 도구를 감지하고 의존성 설치 제안 (lock 파일 기반으로 판단)
- **포커스 추적**: create → 새 워크트리 / done, remove → main / switch → 대상. 항상 갱신 후 안내
- 포커스된 워크트리 경로를 파일 도구(Read/Edit/Glob/Grep)와 Bash `cd`에 반영
- 삭제 전 uncommitted changes + unpushed commits 확인 → 경고 → 사용자 확인
- 브랜치명은 사용자 입력 그대로 사용 (prefix 추가 금지)

### 금지 사항
- `main/` 디렉토리 안에 워크트리 생성
- 사용자 확인 없이 `--force` 삭제
- orphaned 워크트리 방치
- create/switch/done/remove 후 포커스 갱신 누락
- workspace root 밖으로 파일 이동/복사/삭제
- 워크트리 디렉토리 직접 `rm -rf` (반드시 `git worktree remove` 우선. 실패 시에만 사용자 확인 후 수동 삭제)

## 명령어

### `/worktree setup`

기존 git 프로젝트를 workspace 구조로 변환한다. (프로젝트당 1회)

1. 이미 workspace면 (`main/.git` 존재) 안내 후 종료
2. git repo 내부인지 확인 (`.git` 존재). 아니면 에러
3. 기존 worktree가 있으면 경고 (이동 시 참조가 깨짐)
4. 사용자 확인 후 진행
5. **프로젝트 내부에서** workspace 구조로 변환 (부모 디렉토리 접근 금지):
   ```bash
   # 반드시 프로젝트 디렉토리 안에서 실행
   mkdir -p main worktrees
   ```

   **Phase A — 이동 대상 확인**:
   ```bash
   # .git은 별도 처리하므로 제외
   find . -maxdepth 1 ! -name '.' ! -name '..' ! -name '.git' ! -name 'main' ! -name 'worktrees' | sort
   ```
   이동 대상 목록을 사용자에게 표시하고, AskUserQuestion으로 "위 N개 항목 + `.git`을 `main/`으로 이동합니다. 계속하시겠습니까?"를 확인한다.

   **Phase B — 일반 파일/디렉토리 이동** (.git 제외):
   ```bash
   find . -maxdepth 1 ! -name '.' ! -name '..' ! -name '.git' ! -name 'main' ! -name 'worktrees' -exec mv {} main/ \;
   ```

   **Phase C — .git 이동** (마지막에 실행):
   ```bash
   mv .git main/
   ```

   **Phase D — 이동 검증**:
   ```bash
   # 잔류 항목 목록 확인 (find는 allowed-tools에 포함)
   find . -maxdepth 1 ! -name '.' ! -name '..' ! -name 'main' ! -name 'worktrees'
   ```
   위 명령의 출력이 비어있으면 `MOVE_OK`. 출력이 있으면 잔류 항목 목록을 사용자에게 표시하고 "다음 파일이 이동되지 않았습니다:\n{잔류 항목 목록}\n수동으로 `mv <file> main/`을 실행해주세요." 안내 후 **즉시 종료** (검증 단계로 넘어가지 않음).

   **주의**: `cd ..` 또는 부모 경로를 사용하면 Claude Code sandbox에 의해 차단됨
6. `main/CLAUDE.md`가 있으면 workspace root에 복사: `cp main/CLAUDE.md ./CLAUDE.md`
7. 변환 검증: `test -d main/.git && test -d worktrees && echo "OK"`. 검증 실패 시: "workspace 변환이 불완전합니다. `main/` 디렉토리와 원본 파일 위치를 확인해주세요." 안내 후 **즉시 종료**
8. Claude 재시작 안내: `cd <workspace> && claude`

### `/worktree create <name>`

새 워크트리 + 브랜치를 생성한다.

1. workspace 구조 확인
2. **프로젝트 레포 브랜치명 검증** (workspace가 `projects/` 하위인 경우):
   - `.claude/rules/issue-key.md` 규칙을 따른다. 이슈 키 정규식은 `.claude/config.json`의 `issueKey.pattern`을 참조한다.
   - 올바른 형식: `이슈키/설명` (e.g. `AFS-6/local-ddl-auto`)
   - 이슈 키가 없으면 AskUserQuestion으로 이슈 키를 입력받고, `이슈키/설명` 형식으로 조합한다
   - **Claude가 브랜치명을 임의로 생성하지 않는다** — 반드시 사용자가 지정하거나 확인해야 한다
3. 브랜치 중복 체크 (중복 시: 기존 사용 / 다른 이름 / 기존 삭제 중 선택)
4. `git -C main worktree add ../worktrees/<name> -b <name>`
5. 환경 파일 복사 + 빌드 도구 감지 후 의존성 설치 제안 (의존성 설치 Bash 명령에 `timeout: 300000` 설정)
6. 포커스를 새 워크트리로 전환

### `/worktree list`

모든 워크트리 목록 + 현재 포커스를 표시한다. (`git -C main worktree list`)

### `/worktree switch <name>`

작업 대상 워크트리를 전환한다. (`main`으로도 전환 가능)

1. 대상 디렉토리 존재 확인
2. Bash 컨텍스트 `cd` + 포커스 갱신

### `/worktree status`

현재 포커스 워크트리의 git status + 최근 커밋(`git log --oneline -n 10`)을 표시한다.

### `/worktree done`

현재 워크트리 작업을 마무리한다.

1. uncommitted changes + unpushed commits 표시
2. 미커밋/미푸시 변경이 있으면 경고하고, `/commit`과 `/pull-request`을 사용하도록 안내
3. 포커스를 main으로 전환
4. 워크트리 삭제 여부 확인

### `/worktree remove <name>`

워크트리를 삭제한다.

1. uncommitted changes + unpushed commits 확인 → 위험 시 경고
2. 사용자 확인 후 `git -C main worktree remove` (`--force`는 명시적 확인 후만)
3. 브랜치 삭제 여부 확인
4. `git -C main worktree prune`
5. 삭제 대상이 active였으면 포커스를 main으로 전환

## 안전 규칙

- **setup 전**: 디렉토리 이동 경고 → 확인 → 이동 성공 검증
- **remove 전**: uncommitted changes + unpushed commits → 경고 → 확인
- `--force`는 사용자 명시적 확인 후에만
- 브랜치는 명시적 요청 없이 삭제하지 않음
- 수동 삭제된 워크트리는 `git -C main worktree prune`으로 정리

## 에러 처리

| 상황 | 대응 |
|------|------|
| workspace도 git repo도 아님 | git 프로젝트에서 `/worktree setup` 실행 안내 |
| 이미 workspace 구조 | `/worktree create <name>` 사용 안내 |
| 브랜치명 중복 | 기존 사용 / 다른 이름 / 기존 삭제 선택지 제공 |
| 워크트리 디렉토리 이미 존재 | `git worktree list`로 확인 → orphan이면 삭제 확인 |
| 의존성 설치 실패 | 워크트리는 생성됨, 수동 해결 안내 |
| setup 중단 (Phase B — 일반 파일) | `.git`은 원래 위치에 있으므로 프로젝트는 정상. `ls main/`으로 이동된 파일 확인 → 미이동 파일을 `mv <file> main/`으로 수동 이동 → Phase C(.git 이동)부터 재시도 |
| setup 중단 (Phase C — .git 이동) | 일반 파일은 `main/`에 있고 `.git`은 루트에 남은 상태. `mv .git main/`을 수동 실행 → 검증 재실행 |
| create 실패 (부분 생성) | `git worktree list`로 orphan 확인 → `git -C main worktree prune`으로 정리 |

## 예시

```
User: /worktree create auth-refactor

Claude: 워크트리 생성 완료.
  브랜치: auth-refactor
  경로:   worktrees/auth-refactor/
  기반:   main (abc1234)
  환경 파일: .env, .nvmrc 복사
  의존성: npm install ✅

현재 포커스: worktrees/auth-refactor/

User: /worktree list

Claude: 워크트리 목록:
  ★ main/                    [main]          abc1234
    worktrees/auth-refactor/ [auth-refactor] def5678

현재 포커스: worktrees/auth-refactor/

User: /worktree done

Claude: [uncommitted changes, unpushed commits 확인 후 안내]
포커스 전환: main/
```

---

## Permission 참고

이 스킬의 `allowed-tools`는 **스킬 실행 중에만** 유효하다. 워크트리 안에서 일반 개발(테스트, 빌드 등)을 할 때는 사용자의 `settings.json` permission이 적용된다.

워크트리에서 permission 프롬프트 없이 개발하려면 자주 쓰는 명령을 `settings.json`에 추가한다:

```json
{
  "permissions": {
    "allow": [
      "Bash(npm:*)",
      "Bash(bun:*)",
      "Bash(./gradlew:*)",
      "Bash(git -C:*)",
      "Bash(git status:*)",
      "Bash(git log:*)"
    ]
  }
}
```

파일 도구(Read/Edit/Glob/Grep)는 workspace 하위 경로로 접근하므로 별도 permission 설정이 필요 없다.

**Note:** 이 스킬은 워크트리 관리만 담당한다. 커밋, 푸시 등은 `/commit`, `/pull-request` 등을 사용한다.
