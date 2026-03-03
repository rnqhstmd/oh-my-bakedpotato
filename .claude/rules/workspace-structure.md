# Workspace 구조

`projects/` 하위의 코드 레포는 **workspace 모드**로 운영한다.

## 디렉토리 구조

구체적인 경로명은 `.claude/config.json`의 `workspace` 필드를 참조한다.

```
projects/{repo}/              ← workspace root
├── main/                     ← git repo 본체 (읽기 전용)
│   ├── .git/
│   └── src/...
├── worktrees/                ← 모든 워크트리
│   ├── feature-x/
│   └── feature-y/
└── CLAUDE.md                 ← main/CLAUDE.md 복사본
```

## 핵심 규칙

- `main/`은 **읽기 전용**. 코드 탐색, 구조 파악, context 문서화의 근거로만 사용한다.
- 코드 수정은 반드시 워크트리에서 수행한다 (`/worktree create`).
- workspace 구조가 아닌 프로젝트를 발견하면 `/worktree setup`을 제안한다.
- sync 시 `main/`만 pull한다. 워크트리는 개별 관리.

## 관련 스킬

- `/worktree`: workspace 구조 생성 및 워크트리 관리
- `/sync-projects`: clone 시 자동으로 workspace 구조로 변환
- `/dev`: workspace 모드를 감지하여 GIT_PREFIX/PROJECT_ROOT 설정

## 환경 파일

워크트리 생성 시 `main/`에서 복사할 환경 파일 목록은 `.claude/config.json`의 `workspace.envFiles`를 참조한다.
