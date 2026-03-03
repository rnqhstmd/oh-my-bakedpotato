---
name: sync-projects
version: 1.0.0
description: |
  GH에서 레포를 검색하고 projects/에 clone 또는 sync하는 스킬.
  이미 clone된 레포는 최신화하고, 없으면 검색 후 clone합니다.
argument-hint: "[검색어 또는 레포명]"
allowed-tools:
  - "Bash(gh *)"
  - "Bash(git *)"
  - Glob
  - Read
  - Edit
  - AskUserQuestion
---

# sync-projects

GH에서 레포를 검색하여 `projects/`에 clone하거나, 이미 있는 레포를 sync합니다.

## 사용 패턴

| 입력 | 동작 |
|------|------|
| `/sync-projects` | 현재 `projects/` 전체 상태 표시 + 일괄 sync |
| `/sync-projects {검색어}` | GH에서 검색 → 결과 표시 → 선택 → clone/sync |
| `/sync-projects {org/repo}` | 정확한 레포를 바로 clone/sync |

## 실행 절차

### 1. 워크스페이스 감지

- 현재 디렉토리 또는 상위에서 `.claude/workspace.json`이 있는 디렉토리를 찾는다.
- 해당 디렉토리의 `projects/`를 작업 대상으로 사용한다.
- `.claude/workspace.json`에서 워크스페이스 설정을 읽는다:
  ```json
  {
    "team": "...",
    "gh_host": "github.com",
    "projects": {
      "required": [
        { "repo": "xx/xx", "desc": "BE API 서버" }
      ],
      "optional": [
        { "repo": "yy/yy", "desc": "모바일 FE 앱" }
      ]
    }
  }
  ```
- 각 프로젝트의 `repo` 필드에서 `{org}/{repo}` 를 읽고, `desc`는 사용자에게 표시할 때 사용한다.
- `gh_host`를 GHE 호스트로 사용한다.
- 찾지 못하면 사용자에게 경로를 물어본다.

### 2. 인자 없이 호출된 경우 (`/sync-projects`)

1. `.claude/workspace.json`의 `projects.required` + `projects.optional` 목록과 `projects/` 하위 디렉토리를 비교한다.
2. `required` 중 아직 clone되지 않은 레포가 있으면 자동으로 clone한다.
3. `optional` 중 아직 clone되지 않은 레포가 있으면 clone 여부를 물어본다.
4. 이미 clone된 레포에 대해:
   - git repo인지 확인
   - 현재 브랜치, default branch 확인
   - `git fetch` 후 remote와의 차이 (behind/ahead) 확인
4. 상태를 테이블로 표시한다:
   ```
   | 레포 | 브랜치 | 상태 |
   |------|--------|------|
   | shopping-fep | main | ✓ 최신 |
   | shopping-order | main | ↓ 3 commits behind |
   ```
5. behind인 레포가 있으면 "전체 sync할까요?"로 물어본다.
6. 승인 시 각 레포에서 `git pull` 실행한다.

### 3. 검색어와 함께 호출된 경우 (`/sync-projects {검색어}`)

1. `projects/`에 이미 해당 이름의 디렉토리가 있는지 확인한다.
   - 있으면 → sync (git fetch + git pull) 후 상태 표시
   - 없으면 → 2단계로

2. GHE에서 검색한다:
   ```bash
   gh api search/repositories --hostname {ghe_host} -X GET -f q="{검색어}" --jq '.items[] | {full_name, description}'
   ```

3. 검색 결과를 표시하고 clone할 레포를 선택받는다.

4. 선택된 레포를 clone한다:
   ```bash
   gh repo clone https://{ghe_host}/{org}/{repo} projects/{repo}
   ```

5. clone 완료 후 상태를 표시한다.
6. clone한 레포가 `workspace.json`에 없으면 추가할지 물어보고, 승인 시 간단한 설명(desc)을 입력받아 `projects.optional` 배열에 `{ "repo": "...", "desc": "..." }` 형태로 Edit 도구를 사용해 추가한다.

### 4. 정확한 레포명으로 호출된 경우 (`/sync-projects {org/repo}`)

- `{org/repo}` 형식(슬래시 포함)이면 검색 없이 바로 clone/sync한다.
- clone 명령:
  ```bash
  gh repo clone https://{ghe_host}/{org}/{repo} projects/{repo}
  ```
- clone한 레포가 `workspace.json`에 없으면 추가할지 물어보고, 승인 시 간단한 설명(desc)을 입력받아 `projects.optional` 배열에 `{ "repo": "...", "desc": "..." }` 형태로 Edit 도구를 사용해 추가한다.

### 5. Workspace 구조 설정

clone 완료 후, 해당 프로젝트를 workspace 모드(`main/` + `worktrees/`)로 변환한다.

1. clone된 `projects/{repo}/` 디렉토리에서:
   ```bash
   cd projects/{repo}
   mkdir -p main worktrees
   # .git 제외한 파일을 main/으로 이동
   find . -maxdepth 1 ! -name '.' ! -name '..' ! -name '.git' ! -name 'main' ! -name 'worktrees' -exec mv {} main/ \;
   # .git 이동 (마지막)
   mv .git main/
   ```
2. 변환 검증: `test -d main/.git && test -d worktrees`
3. `main/CLAUDE.md`가 있으면 workspace root에 복사: `cp main/CLAUDE.md ./CLAUDE.md`

이미 workspace 구조(`main/.git` 존재)인 프로젝트는 이 단계를 건너뛴다.

### 6. Sync 시 workspace 모드 확인

인자 없이 호출(`/sync-projects`)하여 기존 프로젝트를 sync할 때:
- workspace 구조가 아닌 프로젝트(`projects/{repo}/.git` 존재)를 발견하면, 상태 테이블에 "⚠️ workspace 미설정"으로 표시하고 `/worktree setup` 실행을 제안한다.
- workspace 구조인 프로젝트는 `git -C projects/{repo}/main fetch && git -C projects/{repo}/main pull`로 sync한다.

## 주의사항

- clone 시 항상 **default branch**를 유지한다. 다른 브랜치로 체크아웃하지 않는다.
- clone 후 자동으로 workspace 구조로 변환한다. 변환 실패 시 안내하고 수동 `/worktree setup`을 제안한다.
- `projects/`는 `.gitignore` 대상이므로 워크스페이스 repo에 영향을 주지 않는다.
- sync 시 `main/` 디렉토리 기준으로 pull한다. 워크트리는 개별 관리 대상이므로 건드리지 않는다.
- 네트워크 오류 시 어떤 레포가 실패했는지 명확히 알린다.
- **타임아웃**: 네트워크 명령(GHE API 검색, clone, fetch, pull)에 `timeout: 120000` (2분)을 설정한다. 타임아웃 초과 시 해당 레포를 건너뛰고 실패 목록에 기록한다.

## GHE 호스트

- `.claude/workspace.json`의 `gh_host` 값을 사용한다.
- 검색: `gh api search/repositories --hostname {gh_host}` 사용
- clone: `gh repo clone https://{gh_host}/{org}/{repo}` (full URL) 사용
- sync: `git -C projects/{repo}/main fetch/pull` 사용 (workspace 모드 기준)
- 모든 명령이 `gh` 또는 `git`으로 시작해야 한다 (allowed-tools 제약).
