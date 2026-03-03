# oh-my-bakedpotato

Java Spring Boot 멀티모듈 프로젝트를 위한 Claude Code 플러그인입니다.

PRD 작성, 설계, 구현, 리뷰, 커밋/PR까지 전체 개발 사이클을 에이전트 팀이 Q&A 루프로 수행합니다.

---

## 플러그인 설치

Claude Code의 Plugin Marketplace를 통해 설치합니다:

```bash
# 1. 마켓플레이스에 추가
/plugin marketplace add bs-koo/oh-my-bakedpotato

# 2. 플러그인 설치
/plugin install oh-my-bakedpotato --scope user
```

설치 후 `.claude/settings.local.json.sample`을 참고하여 권한을 커스텀할 수 있습니다:

```bash
cd oh-my-bakedpotato/.claude
cp settings.local.json.sample settings.local.json
```

---

## 동작 원리

이 플러그인은 대상 프로젝트의 `CLAUDE.md`와 합쳐져 동작합니다:

| 출처 | 제공하는 것 |
|------|------------|
| **프로젝트의 `CLAUDE.md`** | 아키텍처, 코딩 컨벤션, 빌드 명령 등 프로젝트 고유 규칙 |
| **플러그인의 `.claude/`** | 스킬(`/dev`, `/commit` 등), 에이전트 팀, 행동 규칙, 안전장치 훅 |

플러그인의 `CLAUDE.md`에 "프로젝트의 CLAUDE.md를 먼저 읽으라"고 명시되어 있으므로, **프로젝트 컨벤션이 플러그인의 일반 규칙보다 우선** 적용됩니다.

---

## 시작하기

### 사전 요구사항

`/setup` 스킬이 OS를 자동 감지하여 필수 도구를 확인합니다 (macOS, Linux, Windows Git Bash 지원).

| 도구 | 용도 |
|------|------|
| Git | 버전 관리 |
| GitHub CLI (gh) | GH 인증, PR 생성 |
| JDK 21 | Java 빌드 |
| Docker | 인프라 (MySQL, Redis, Kafka) |

### 초기 세팅

Claude Code 실행 후:

```
/setup
```

OS에 맞는 설치 안내, GH 인증, 프로젝트 clone을 자동으로 처리합니다.

---

## 개발 워크플로우

### 전체 사이클 (`/dev`)

```
/dev [JIRA-123] 상품 목록 정렬 기능 추가
```

PRD → 설계 → 구현 → 리뷰 → 커밋/PR까지 에이전트 팀이 단계별로 수행합니다. 각 단계마다 사용자 확인을 거칩니다.

### 개별 스킬

```
/commit          # 한국어 커밋 메시지로 Git 커밋
/pull-request    # 커밋 히스토리 기반 PR 자동 생성
/worktree create # 기능 브랜치용 워크트리 생성
```

---

## 에이전트 팀

`/dev` 스킬은 다음 에이전트들을 단계별로 호출합니다:

| 에이전트 | 역할 |
|---------|------|
| **product-owner** | 모호한 요청 → 명확한 PRD 변환, 인수 검증 |
| **architect** | PRD + 코드 맵 기반 기술 설계 작성 |
| **design-critic** | 설계 초안의 암묵적 가정 도전, 불필요한 복잡성 식별 |
| **coder** | 승인된 설계 기반 코드 구현 |
| **qa-manager** | 코드 리뷰, 스펙 충족 검증 |
| **security-auditor** | PRD/설계/코드 교차 검증, 보안 취약점 식별 |

정체 상황 발생 시 자동 호출되는 에스컬레이션 에이전트:

| 에이전트 | 역할 |
|---------|------|
| **researcher** | 코드베이스 탐색, 버그 근본 원인 분석 |
| **hacker** | 반복 실패 시 제약 우회 경로 탐색 |
| **simplifier** | 과도한 복잡성 → 범위 축소, 최소 해법 제시 |

---

## 안전장치

| 장치 | 설명 |
|------|------|
| `settings.json` deny 규칙 | `gh pr merge`, `git push --force` 차단 |
| `pre-tool-guard.sh` 훅 | 보호 브랜치(main)에서 직접 커밋 차단 |
| 작업 범위 제한 | PR 생성까지만 — PR 머지는 사용자가 직접 수행 |

---

## 디렉토리 구조

```
oh-my-bakedpotato/
├── CLAUDE.md              ← 플러그인 소개 및 작업 범위
├── README.md
├── .gitignore
│
├── .claude/
│   ├── config.json        ← 이슈 키 패턴, 타임아웃 등 설정
│   ├── workspace.json     ← 워크스페이스 설정 (팀, GH, 프로젝트 목록)
│   ├── settings.json      ← 팀 공유 권한/훅 설정
│   ├── agents/            ← 에이전트 정의 (9종)
│   ├── rules/             ← 행동 규칙 (자동 로드)
│   ├── hooks/             ← 안전장치 (보호 브랜치 커밋 차단 등)
│   └── skills/            ← 스킬 정의
│
├── context/               ← 도메인 지식 (누적 자산)
│   ├── README.md          ← 도메인 목록 인덱스
│   ├── glossary.md        ← 공통 용어 사전
│   └── {도메인}/           ← 도메인별 컨텍스트
│
└── projects/              ← 코드 레포 (.gitignore 대상)
    └── {name}/
        ├── main/          ← 기본 브랜치 (읽기 전용)
        └── worktrees/     ← 기능별 작업 브랜치
```

## 스킬 목록

| 스킬 | 설명 |
|------|------|
| `/setup` | 초기 세팅 (도구 확인, GH 인증, 프로젝트 clone) |
| `/sync-projects` | GH 레포 clone 및 최신화 |
| `/new-context` | 새 도메인 컨텍스트 생성 |
| `/dev` | PRD → 설계 → 구현 → 리뷰 → 커밋/PR 전체 사이클 |
| `/commit` | 한국어 커밋 메시지로 Git 커밋 |
| `/pull-request` | 커밋 히스토리 기반 PR 자동 생성 |
| `/worktree` | projects/ 코드 레포의 Git worktree 자동화 |
| `/lens` | 코드 속 비즈니스 정책 탐지 → PO/PD 보고서 |
| `/humanizer` | AI 글쓰기 패턴 감지 및 교정 |

## 자주 묻는 질문

**다음에 다시 시작하려면?**
```bash
cd oh-my-bakedpotato
claude
```

**이전 작업을 이어서 하고 싶으면?**
Claude Code 실행 후 `/resume`을 입력하면 이전 세션 목록이 나옵니다.

**`/setup`을 다시 실행해도 되나요?**
네. 이미 설치된 것은 건너뛰고, 새로 추가된 것만 처리합니다.
