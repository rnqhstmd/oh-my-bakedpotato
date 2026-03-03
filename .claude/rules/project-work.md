---
paths:
  - "projects/**"
---

# 프로젝트 작업 규칙

- 프로젝트 목록은 `.claude/workspace.json`에서 관리합니다.
- `/sync-projects`로 clone 및 최신화하세요. clone 시 workspace 구조(`main/` + `worktrees/`)로 자동 설정됩니다.

## projects/ 작업 규칙

- Workspace 구조 규칙은 `.claude/rules/workspace-structure.md`를 따릅니다.
- 코드 탐색 시 `projects/{name}/main/` 경로를 사용하세요. 워크트리가 있더라도 최신 코드 기준은 `main/`입니다.
- **코드에 대한 질문을 받으면** 탐색 전에 해당 프로젝트의 `main/`을 먼저 pull하여 최신화하세요. 오래된 코드 기반으로 답변하면 안 됩니다.
- 코드 수정은 반드시 `/worktree create`로 워크트리를 만든 뒤 진행하세요.
- 해당 프로젝트 작업 시 `projects/{name}/main/CLAUDE.md`를 먼저 읽고 코드 컨벤션을 파악하세요.
- 코드 컨벤션을 발견하면 `projects/{name}/main/CLAUDE.md`에 반영을 제안하고, workspace root에도 재복사합니다(`cp projects/{name}/main/CLAUDE.md projects/{name}/CLAUDE.md`).

## 개발 작업 흐름

코드 변경이 수반되는 요청을 받으면 아래 순서를 따른다. **코드부터 작성하고 문서를 나중에 맞추는 것은 금지.**

1. **context 문서 확인**: 관련 도메인의 `context/{도메인}/` 문서를 읽고, 요청과 현재 설계/정책 사이의 이격을 파악한다. 해당 도메인 context가 아직 없으면 `/new-context`로 먼저 생성한다.
2. **설계 반영**: 새 필드, 정책 변경, 스키마 변경 등이 있으면 context 문서를 먼저 갱신한다. 사용자와 설계 핑퐁이 필요하면 이 단계에서 수행한다. **문서를 수정하면 즉시 수정일을 갱신한다.**
3. **status.md 확인**: 구현할 FR/BR 항목을 `status.md`에서 확인하고, 변경 범위를 인지한다.
4. **코드 구현**: 설계 문서가 확정된 후에만 코드 작업에 진입한다.
5. **문서 마무리**: 구현 완료 후 `status.md` 상태 갱신(⬜→✅) + PR 설명 업데이트.
