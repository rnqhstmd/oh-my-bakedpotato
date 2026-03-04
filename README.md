# oh-my-bakedpotato

Java Spring Boot 멀티모듈 프로젝트를 위한 Claude Code 플러그인입니다.

PRD 작성, 설계, 구현, 리뷰, 커밋/PR까지 전체 개발 사이클을 에이전트 팀이 Q&A 루프로 수행합니다. 각 단계마다 사용자 확인을 거치며, PR 생성까지만 자동화합니다.

---

## 플러그인 설치

Claude Code에서 아래 두 명령을 실행합니다:

```bash
# 1. 마켓플레이스 등록
/plugin marketplace add rnqhstmd/oh-my-bakedpotato

# 2. 플러그인 설치
/plugin install oh-my-bakedpotato@oh-my-bakedpotato
```

설치 스코프를 선택할 수 있습니다:

| 스코프 | 명령 | 설명 |
|--------|------|------|
| user (기본) | `/plugin install oh-my-bakedpotato@oh-my-bakedpotato` | 모든 프로젝트에서 사용 |
| project | Discover 탭에서 project 스코프 선택 | 해당 repo 협업자 전체 |
| local | Discover 탭에서 local 스코프 선택 | 나만 이 repo에서 사용 |

---

## 빠른 시작

이 플러그인은 대상 프로젝트의 `CLAUDE.md`(아키텍처, 컨벤션)와 플러그인의 `.claude/`(스킬, 에이전트, 규칙)가 합쳐져 동작합니다. **프로젝트 컨벤션이 플러그인의 일반 규칙보다 우선** 적용됩니다.

**1단계: 환경 확인**

```
/oh-my-bakedpotato:setup
```

OS를 자동 감지하여 Git, GitHub CLI, JDK 21, Docker 등 필수 도구와 GH 인증을 확인합니다.

**2단계: 도메인 컨텍스트 등록 (선택)**

```
/oh-my-bakedpotato:new-context 결제
```

도메인의 용어, 아키텍처를 등록하면 `/dev` 실행 시 자동으로 매칭하여 참조합니다. 없어도 모든 스킬이 동작합니다.

**3단계: 개발 시작**

```
/oh-my-bakedpotato:dev [JIRA-123] 상품 목록 정렬 기능 추가
```

PRD → 설계 → 구현 → 리뷰 → 커밋/PR까지 에이전트 팀이 단계별로 수행합니다.

---

## /dev: 전체 개발 사이클

### 호출 방법

```
/oh-my-bakedpotato:dev <기능/버그 설명> [옵션]
```

| 옵션 | 설명 |
|------|------|
| (없음) | 전체 사이클 (setup → requirements → design → implement → review → complete) |
| `--phase <name>` | 특정 Phase만 실행 (requirements, design, implement, review, complete) |
| `--hotfix` | 긴급 수정 경량 경로 (설계/리뷰 건너뜀) |
| `--base <branch>` | 베이스 브랜치 지정 (미지정 시 main/develop/master 자동 감지) |
| `--status` | 진행 상태 조회 (파이프라인 실행 없이 `.dev/state.md` 확인) |
| `--resume` | 이전 작업 재개 (중단된 Step부터 계속) |

플래그 제약: `--hotfix`와 `--phase`는 동시 사용 불가. `--resume`은 `--phase`, `--hotfix`, `--status` 및 작업 설명과 동시 사용 불가.

### Phase별 흐름

```
setup → requirements → design → implement → review → complete
         (PO)        (architect  (coder     (qa-manager (PO 인수
                     + critic)  + qa 점검)  + security)  + commit/PR)
```

| Phase | 에이전트 | 사용자 상호작용 | 산출물 |
|-------|---------|----------------|--------|
| setup | 오케스트레이터 | 없음 | 브랜치, 코드 맵, `.dev/state.md` |
| requirements | product-owner | Q&A 최대 1회 + 승인 | `.dev/prd.md` |
| design | architect + design-critic (선택적) | Q&A 최대 2회 + 승인 | `.dev/design.md` |
| implement | coder + qa-manager | 자기점검 결과 보고 | 코드, `.dev/self-check.md` |
| review | qa-manager + security-auditor | CERTAIN 확인, QUESTION 답변 (최대 2회) | `.dev/trust-ledger.md` |
| complete | product-owner | 인수 결과 확인 (재시도 최대 1회) | 커밋, PR |

**setup**: 작업 브랜치 생성, 프로젝트 타입 감지, 코드 맵 초기 생성, 도메인 컨텍스트 자동 매칭.

**requirements**: product-owner가 모호한 요청을 명확한 PRD로 변환합니다. 사용자와 Q&A 1회 후 PRD를 확정합니다.

**design**: architect가 PRD와 코드 맵을 기반으로 기술 설계를 작성합니다. 필요 시 design-critic(opus 모델)이 암묵적 가정을 도전하고 불필요한 복잡성을 식별합니다. 사용자 승인 후 확정됩니다.

**implement**: coder가 확정된 설계에 따라 코드를 구현합니다. qa-manager가 자기점검을 수행하여 스펙 충족을 확인합니다. Critical 이슈는 coder가 즉시 수정합니다.

**review**: qa-manager와 security-auditor가 병렬로 코드 리뷰와 보안 감사를 수행합니다. 확실한 문제(CERTAIN)는 수정하고, 확인이 필요한 사항(QUESTION)은 사용자에게 질문합니다. 최대 2회 반복합니다.

**complete**: product-owner가 PRD 수용 기준 대비 인수 검증을 수행합니다. 통과하면 커밋과 PR을 생성합니다.

### 일반 개발 vs --hotfix

| 항목 | 정상 경로 | --hotfix |
|------|----------|----------|
| PRD | 전체 (Q&A 포함) | 경량 (배경 + 요구사항 + 수용 기준만) |
| 설계 | architect + design-critic | 건너뜀 (coder가 PRD + 코드 맵으로 직접 구현) |
| 리뷰 | qa-manager + security-auditor 병렬 | 건너뜀 |
| 인수 검증 | 실행 | 실행 (PRD 수용 기준 대비 검증) |

```
--hotfix: setup → requirements(경량) → implement → complete
정상:     setup → requirements → design → implement → review → complete
```

### Phase 제어

특정 Phase만 실행하거나, 중단된 작업을 재개할 수 있습니다.

```bash
# PRD만 작성
/dev 로그인 기능 추가 --phase requirements

# 설계까지만 (requirements + design)
/dev 로그인 기능 추가 --phase design

# 현재 변경사항을 리뷰만
/dev --phase review

# 커밋/PR만
/dev --phase complete

# 진행 상태 확인
/dev --status

# 이전 작업 재개 (중단된 Step부터)
/dev --resume
```

`--phase implement`는 `.dev/design.md`가 필요합니다. 없으면 `--phase design`을 먼저 실행하라는 안내가 나옵니다.

### 정체 감지와 에스컬레이션

implement와 review Phase에서 반복 루프가 진전 없이 정체되면, 자동으로 에스컬레이션 에이전트를 호출합니다.

| 패턴 | 감지 기준 | 1차 대응 | 2차 대응 |
|------|----------|---------|---------|
| SPINNING | 동일 에러가 2회 연속 반복 | hacker: 제약 우회 분석 | researcher: 근본 원인 분석 |
| OSCILLATION | 접근법 A→B→A 왕복 | architect: 설계 재검토 | 사용자에게 두 접근법 제시 |
| NO_DRIFT | 코드 변경이 실질적으로 없음 (diff 비교) | hacker: 제약 식별 + 우회 | researcher: 코드베이스 탐색 |
| DIMINISHING_RETURNS | 수정 범위가 줄어드는데 테스트/리뷰 결과 미개선 | simplifier: 복잡도 분석 + 범위 축소 | 사용자에게 방향 전환 확인 |

---

## 개별 스킬 가이드

모든 스킬은 `/oh-my-bakedpotato:<스킬명>` 형식으로 호출합니다.

### commit

변경사항을 스테이징하고, 브랜치명에서 타입을 파싱하여 한국어 커밋 메시지를 생성합니다.

```
/oh-my-bakedpotato:commit              # 자동 생성
/oh-my-bakedpotato:commit 로그인 검증 수정  # 메시지 직접 지정
```

핵심 동작:
- 커밋 전 프로젝트 타입 감지 후 테스트 실행 (`build.gradle` 존재 시 `./gradlew test`, 실패 시 커밋 중단)
- 브랜치명에서 타입 파싱 (예: `feat/login` → `feat`)
- 메시지 형식: `{type}: 한국어 요약` + 변경사항 bullet (예: `feat: 로그인 기능 추가`)
- 민감 파일(`.env`, `credentials` 등) 감지 시 경고
- 변경 파일 20개 초과 시 전체 스테이징 여부 확인
- `Co-Authored-By` 라인 추가 금지

### pull-request

커밋 히스토리를 분석하여 PR을 자동 생성합니다.

```
/oh-my-bakedpotato:pull-request         # 베이스 브랜치 자동 감지
/oh-my-bakedpotato:pull-request develop  # 베이스 브랜치 지정
```

핵심 동작:
- 브랜치 타입에서 PR 제목 타입 매핑 (`feat`→`[FEATURE]`, `fix`→`[BUGFIX]` 등)
- PR 제목: `[{TYPE}] 한국어 서술형 문장` (50자 이내)
- PR 본문: Background / Summary / Changes / Checklist 구조 (문장형 서술)
- 기존 PR이 있으면 업데이트/신규 생성/취소 선택
- `gh` CLI 미설치 시 설치 안내, 미인증 시 `gh auth login` 실행
- GHE 지원: origin remote URL에서 호스트를 감지하여 `GH_HOST` 환경변수를 자동 설정

### worktree

격리된 기능 개발을 위한 Git worktree를 관리합니다.

```
/oh-my-bakedpotato:worktree create feat/login   # 워크트리 + 브랜치 생성
/oh-my-bakedpotato:worktree list                # 워크트리 목록
/oh-my-bakedpotato:worktree switch feat/login    # 워크트리 전환 안내
/oh-my-bakedpotato:worktree status              # 현재 상태 확인
/oh-my-bakedpotato:worktree done                # 작업 마무리
/oh-my-bakedpotato:worktree remove feat/login   # 워크트리 삭제
```

브랜치명은 `{type}/{description}` 형식을 따릅니다. 타입 없이 이름만 제공하면 타입을 선택받습니다. 삭제 전 미커밋 변경사항과 미푸시 커밋을 확인하고 경고합니다.

### new-context

도메인 컨텍스트를 검증 질문을 거쳐 생성합니다.

```
/oh-my-bakedpotato:new-context 결제
```

Q&A 흐름:
1. 도메인의 문제, 현재 프로세스, 안 하면 어떻게 되는가, 사용자/규모를 질문
2. 모호한 답변이 많으면 심화 질문 1라운드 추가
3. 관련 레포 확인

생성 결과 (`context/` 디렉토리가 없으면 `context/README.md`, `context/glossary.md`도 함께 초기화):
```
context/{도메인}/
├── README.md          ← 도메인 개요 (배경, 문제, 성공 기준)
├── PROJECTS.md        ← 관련 레포 매핑
├── glossary.md        ← 도메인 용어 사전
├── architecture.md    ← 아키텍처 인덱스
└── status.md          ← 구현 추적
```

### lens

코드에서 비즈니스 정책을 탐지하고, 변경 시 영향도를 PO/PD 친화적 보고서로 제공합니다. **읽기 전용** — 탐색 대상 레포의 코드를 절대 변경하지 않습니다.

```
/oh-my-bakedpotato:lens shopping에서 구매 정책 정리해줘
/oh-my-bakedpotato:lens shopping에서 구매 정책 --detail
/oh-my-bakedpotato:lens 미니혜택탭 정책 --projects shopping-* --skip-update
/oh-my-bakedpotato:lens 구매 한도 --idea "한도를 50만원으로 올리면?"
```

| 옵션 | 설명 |
|------|------|
| `--detail` | 상세 모드. 프로젝트당 더 많은 파일 탐색 |
| `--projects <glob>` | 프로젝트 필터 패턴 (예: `shopping-*`) |
| `--skip-update` | git fetch/pull 건너뜀. 반복 실행 시 토큰 절약 |
| `--idea "<아이디어>"` | 정책 보고서 후 영향도 분석까지 자동 실행 |

5 Phase 흐름: Prepare(프로젝트 감지) → Explore(정책 탐색) → Report(보고서) → Impact(영향도 분석) → Impact-Report(최종 보고서). `--idea` 미지정 시 Report 후 사용자에게 아이디어를 질문합니다.

### humanizer

AI가 쓴 티를 제거하는 글쓰기 편집 도구입니다. 40+ 가지 한국어/영어 AI 글쓰기 패턴을 감지하고 교정합니다.

```
/oh-my-bakedpotato:humanizer docs/guide.md          # 파일 지정
/oh-my-bakedpotato:humanizer "오늘날 급변하는..."     # 텍스트 직접 입력
```

| 모드 | 설명 |
|------|------|
| **audit** | 패턴을 감지하고 리포트만 출력 (텍스트 수정 없음) |
| **rewrite** | 패턴 감지 + 직접 수정 수행 |

심각도 3단계:
- **P1**: 확실한 AI 흔적 (도입부 상투어, 과장 수식어, "이를 통해" 연쇄 등) — 즉시 수정
- **P2**: 의심스러운 패턴 ("그렇다면 왜~" 자문자답, 격식체 과용 등) — 맥락 판단
- **P3**: 스타일 개선 (한자어 남용, 마크다운 남용 등) — 선택적 수정

기본값은 P2까지 수정, P3은 리포트만. 콘텐츠 유형(블로그, 기술 문서, 학술 등)에 따라 적용 기준이 달라집니다.

---

## 에이전트 팀

### 핵심 에이전트

| 에이전트 | 활동 Phase | 역할 | 모델 |
|---------|-----------|------|------|
| product-owner | requirements, complete | PRD 작성, 인수 검증 | sonnet |
| architect | design | PRD + 코드 맵 기반 기술 설계 | sonnet |
| design-critic | design | 설계 초안의 암묵적 가정 도전, 복잡성 식별 | opus |
| coder | implement | 승인된 설계 기반 코드 구현 | inherit |
| qa-manager | implement, review | 자기점검, 코드 리뷰, 스펙 충족 검증 | sonnet |
| security-auditor | review | PRD/설계/코드 교차 검증, 보안 취약점 식별 | sonnet |

### 에스컬레이션 에이전트

정체 상황 발생 시 자동으로 호출됩니다.

| 에이전트 | 역할 | 모델 |
|---------|------|------|
| researcher | 코드베이스 탐색, 버그 근본 원인 분석, 기술 비교 | sonnet |
| hacker | 반복 실패 시 제약 우회 경로 탐색 | sonnet |
| simplifier | 과도한 복잡성 → 범위 축소, 최소 해법 제시 | sonnet |

---

## 산출물 (.dev/ 디렉토리)

`/dev` 실행 시 `.dev/` 디렉토리에 단계별 산출물이 생성됩니다. 이 디렉토리는 `.gitignore`에 자동 추가됩니다.

| 파일 | 생성 시점 | 설명 |
|------|----------|------|
| `state.md` | setup | 파이프라인 진행 상태 (Phase, Step, 실행 로그). `--resume` 시 참조 |
| `prd.md` | requirements | 확정된 PRD (요구사항, 수용 기준) |
| `design.md` | design | 확정된 기술 설계서 |
| `codemap.md` | setup~ | 관련 파일 경로와 역할 매핑 (최대 25개, 누적 갱신) |
| `self-check.md` | implement | 자기점검 결과 (qa-manager 수행) |
| `trust-ledger.md` | review | security-auditor의 감사 결과 누적 |
| `diff.txt` | implement~ | 변경사항 diff (에이전트 전달용) |

---

## 도메인 컨텍스트 (context/)

`/new-context`로 생성한 도메인 지식은 `context/` 디렉토리에 저장됩니다.

```
context/
├── README.md              ← 도메인 목록 인덱스
├── glossary.md            ← 공통 용어 사전
└── 결제/
    ├── README.md          ← 도메인 개요 (배경, 문제, 성공 기준)
    ├── PROJECTS.md        ← 관련 레포 매핑
    ├── glossary.md        ← 도메인 용어 사전
    ├── architecture.md    ← 아키텍처 인덱스
    └── status.md          ← 구현 추적
```

`/dev`와의 연동:
- **자동 매칭**: setup Phase에서 `context/*/PROJECTS.md`를 탐색하여 현재 프로젝트와 관련된 도메인을 자동 매칭합니다.
- **status.md 갱신**: complete Phase에서 구현된 항목의 상태를 `⬜→✅`로 갱신합니다.
- **context 환류**: `/dev` 중 발견된 도메인 지식(정책 변경, 스키마 변경 등)은 context 문서에 반영을 제안합니다.

---

## 안전장치

| 장치 | 설명 |
|------|------|
| `settings.json` deny 규칙 | `gh pr merge`, `git push --force` 차단 |
| `pre-tool-guard.sh` 훅 | 보호 브랜치(main)에서 직접 커밋 차단 |
| 작업 범위 제한 | PR 생성까지만 — PR 머지는 사용자가 직접 수행 |
| commit 스킬 검증 | 커밋 전 테스트 실행, 민감 파일 감지, 빌드 아티팩트 tracking 해제 |
| worktree 삭제 보호 | 미커밋 변경사항/미푸시 커밋 확인 후 경고, `--force` 사용자 확인 필수 |

---

## 실제 사용 시나리오

### 시나리오 1: 신규 기능 개발 (처음부터 PR까지)

```bash
# 1. 환경 확인 (최초 1회)
/oh-my-bakedpotato:setup

# 2. 도메인 컨텍스트 등록
/oh-my-bakedpotato:new-context 주문

# 3. 전체 개발 사이클 실행
/oh-my-bakedpotato:dev [ORDER-42] 주문 취소 기능 추가

# 4. PR이 생성되면 GitHub에서 직접 머지
```

에이전트 팀이 PRD 작성 → 설계 → 구현 → 리뷰 → 커밋 → PR 생성까지 수행합니다. 각 단계에서 사용자 확인을 거칩니다.

### 시나리오 2: 긴급 수정

```bash
# hotfix 경로: 설계/리뷰 건너뜀, 경량 PRD + 인수 검증은 실행
/oh-my-bakedpotato:dev 결제 금액 소수점 절삭 오류 수정 --hotfix
```

### 시나리오 3: 개별 작업

```bash
# 수동으로 코드 수정 후 커밋만
/oh-my-bakedpotato:commit

# 커밋 후 PR만 생성
/oh-my-bakedpotato:pull-request

# 진행 상태 확인 후 재개
/oh-my-bakedpotato:dev --status
/oh-my-bakedpotato:dev --resume
```

---

## 플러그인 구조

```
oh-my-bakedpotato/
├── .claude-plugin/
│   ├── plugin.json        ← 플러그인 매니페스트
│   └── marketplace.json   ← 마켓플레이스 카탈로그
│
├── CLAUDE.md              ← 플러그인 소개 및 작업 범위
│
├── .claude/
│   ├── config.json        ← 이슈 키 패턴, 타임아웃 등 설정
│   ├── settings.json      ← 팀 공유 권한/훅 설정
│   ├── agents/            ← 에이전트 정의 (9종)
│   ├── rules/             ← 행동 규칙 (자동 로드)
│   ├── hooks/             ← 안전장치 (보호 브랜치 커밋 차단 등)
│   └── skills/            ← 스킬 정의
│
└── context/               ← 도메인 지식 예시 (/new-context로 소비 프로젝트에 생성)
```

---

## 자주 묻는 질문

**플러그인 설치 후 어떻게 시작하나요?**

프로젝트 디렉토리에서 Claude Code를 실행하면 플러그인이 자동으로 로드됩니다:
```bash
cd my-spring-project
claude
```

**context/는 꼭 만들어야 하나요?**

아니요. 없어도 모든 스킬이 동작합니다. 도메인 지식을 등록하면 에이전트가 용어와 아키텍처를 참조하여 더 정확한 결과를 생성합니다. `/oh-my-bakedpotato:new-context <도메인명>`으로 생성하세요.

**`/setup`을 다시 실행해도 되나요?**

네. 이미 설치된 것은 건너뛰고, 새로 추가된 것만 처리합니다.

**플러그인을 업데이트하려면?**

```
/plugin marketplace update oh-my-bakedpotato
```

**세션이 끊겼을 때 작업을 이어가려면?**

`/dev --resume`으로 이전 파이프라인을 재개합니다. `.dev/state.md`에 기록된 중단 Step부터 계속합니다. state.md가 없거나 이미 완료 상태이면 에러가 발생합니다.

**특정 Phase만 실행하고 싶으면?**

`/dev <설명> --phase <name>`으로 원하는 Phase만 실행합니다. 예: `--phase requirements`는 PRD만 작성하고, `--phase review`는 현재 변경사항만 리뷰합니다.

**`--hotfix`와 `--phase`를 같이 쓸 수 있나요?**

아니요. 동시 사용 불가합니다. `--hotfix`는 고정된 경량 경로(setup → requirements → implement → complete)를 사용하므로, Phase를 개별 지정할 수 없습니다.

**lens는 코드를 수정하나요?**

아니요. lens는 **읽기 전용**입니다. 탐색 대상 레포의 코드를 절대 변경하지 않으며, 보고서만 출력합니다.

**humanizer의 audit과 rewrite 차이는?**

audit은 AI 글쓰기 패턴을 감지하고 리포트만 출력합니다 (텍스트 수정 없음). rewrite는 감지된 패턴을 직접 수정까지 수행합니다. 기본값은 P2까지 수정, P3은 리포트만입니다.
