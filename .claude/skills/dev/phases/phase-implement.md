# phase-implement: 구현 + 자기점검

## Hotfix 모드 분기

오케스트레이터가 `--hotfix` 모드이면:
- Step 0에서 설계서(`design.md`) 로드를 건너뛴다. PRD(`prd.md`)는 로드한다.
- 구현 Task에서 설계서 대신 PRD와 코드 맵을 전달한다.
  - prompt: "다음 PRD를 참고하여 최소한의 변경으로 구현하라: {PRD 내용}. 코드 맵을 참고하라."
- 자기점검은 동일하게 실행한다.

hotfix가 아닌 경우 아래 정상 플로우를 따른다.

## 구현

**Task A**: coder 구현.

**Step 0**: 문서 로드.
- `${PROJECT_ROOT}/.dev/design.md`를 Read하여 설계서를 로드한다.
- `${PROJECT_ROOT}/.dev/prd.md`를 Read하여 PRD를 로드한다 (자기점검에서 "요구사항"+"수용 기준" 섹션 사용).

`Task(subagent_type="coder")` — prompt에 다음을 포함:
- 확정된 설계서 (Step 0에서 로드한 설계 문서 전체)
- 코드 맵 (누적된 상태)
- 프로젝트 타입 및 구조
- 프로젝트 루트 경로 (작업 경로 기준 참조)
- "구현 순서" 섹션에 따라 순서대로 구현할 것

## Task 완료 후

**Step 1**: coder 결과를 받은 후:
- 설계서 "구현 순서"의 항목 수와 coder의 보고 단계 수(`[N/M]`의 M)를 비교한다. 불일치 시 누락 항목을 명시하고 사용자에게 확인한다.
- **요약만** 사용자에게 보고한다 (Agent 전문 출력 금지. 코드는 파일에 이미 작성됨):
  ```
  구현 완료: M단계
  - [1/M] <파일> - <변경 요약>
  - [2/M] <파일> - <변경 요약>
  - ...
  특이사항: (설계 불일치 등, 있으면)
  ```
- Agent가 설계에서 벗어난 판단을 했다면 해당 내용을 특이사항에 포함하고 사용자 확인을 받는다.

## 자기점검 (1회 패스, 루프 없음)

사용자 리뷰(phase-review) 전에 명백한 실수를 자동으로 잡는다. **1회만 실행하고 루프하지 않는다.**

**조건**: `${GIT_PREFIX} diff --stat`으로 변경 규모와 대상 파일을 확인한다 (unstaged 변경 기준). 다음 **두 조건을 모두** 만족할 때만 자기점검을 건너뛰고 phase-review로 직행한다:
1. 총 변경이 **10줄 미만**
2. 변경된 파일이 **설정 파일만**으로 구성 (e.g., `.yml`, `.yaml`, `.json`, `.toml`, `.properties`, `.env`, `.md`). 코드 파일(`.kt`, `.java`, `.ts`, `.js`, `.py` 등)이 하나라도 포함되면 규모와 무관하게 자기점검을 실행한다.

**Step 1**: 변경사항 수집 및 파일 저장 (작업 경로 기준에 따라 GIT_PREFIX를 붙여 실행). 이 스테이징은 diff 추출 목적이며, 커밋은 phase-complete의 commit이 별도로 수행한다. 스테이징 상태는 이후 phase-review의 diff 수집과 phase-complete의 commit까지 유지된다 (각 단계에서 `git add -A`를 재실행하므로 중간에 coder가 수정한 파일도 포함됨).

1. `${GIT_PREFIX} add -A`로 스테이징한다.
2. **Diff 수집 규칙**에 따라 diff를 `DIFF_FILE`에 리다이렉트한다 (`git diff --cached`를 Bash 단독 실행하지 않는다).

**Step 2**: qa-manager agent로 자동 리뷰.
`Task(subagent_type="qa-manager")` — prompt에 다음을 포함:
- 변경사항 diff 파일 경로 (`DIFF_FILE`) + Read 지시
- PRD의 "요구사항" + "수용 기준" 섹션만 (Context Slicing 규칙 참조: 자기점검 모드). `--hotfix`이면 PRD만 전달 (설계서 없음).
- "자기점검 단계이므로 CERTAIN 문제만 자동 수정 대상으로 취급할 것. QUESTION은 보고하되 수정하지 않고 phase-review로 이월한다."

**Step 3**: 결과 판단.
- **Critical이 있으면**: `Task(subagent_type="coder")` — Critical 항목 목록, qa-manager가 제시한 수정 방안, 코드 맵, 프로젝트 루트 경로를 prompt에 포함하여 자동 수정 (Context Slicing: coder 수정 모드). 수정 실패 시(coder가 해결하지 못했거나 수정 후에도 문제가 남는 경우) 미해결 Critical을 `SELF_CHECK_FINDINGS`에 `[Critical/미해결]`로 추가하고 phase-review로 이월한다. 자기점검은 1회만 시도하므로 재시도하지 않는다.
- **Critical이 없으면**: 자기점검 완료.
- Warning/Info는 `SELF_CHECK_FINDINGS` 변수에 저장한다. 형식: `[Warning] 파일:라인 - 설명` (한 줄씩). phase-review에서 qa-manager 프롬프트에 포함하여 중복 보고를 방지한다.
- QUESTION은 `SELF_CHECK_QUESTIONS` 변수에 저장한다. phase-review에서 qa-manager 프롬프트에 포함하여 사용자 확인을 받는다.
- 자기점검 결과(SELF_CHECK_FINDINGS + SELF_CHECK_QUESTIONS)를 `${PROJECT_ROOT}/.dev/self-check.md`에 Write한다. `--resume`으로 phase-review에서 재개할 때 이 파일을 Read하여 복원한다.

자기점검 결과를 사용자에게 **요약만** 보고한다 (Agent 전문 출력 금지):
```
자기점검 완료:
- Critical: N건 (자동 수정 완료/실패)
- Warning/Info: N건 (phase-review로 이월)
- QUESTION: N건 (phase-review로 이월)
```
이후 phase-review로 진행.
