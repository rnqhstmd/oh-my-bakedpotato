# Command Center

Java Spring Boot 프로젝트를 위한 AI 기반 개발 플러그인입니다.

PRD 작성, 설계, 구현, 리뷰, 커밋/PR까지 전체 개발 사이클을 에이전트 팀이 수행합니다.

---

## 플러그인 설치

이 레포지토리를 Claude Code Plugin으로 추가합니다:

1. GitHub에서 이 레포를 clone하거나 fork
2. Claude Code 설정에서 Plugin으로 로컬 경로 추가
3. `settings.local.json.sample`을 `settings.local.json`으로 복사 후 필요에 맞게 수정

```bash
cd project-command-center/.claude
cp settings.local.json.sample settings.local.json
```

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

OS에 맞는 설치 안내와 GH 인증을 자동으로 처리합니다.

---

## 프로젝트와 함께 사용하기

이 플러그인은 Java Spring Boot 멀티모듈 프로젝트와 함께 사용합니다.

1. 프로젝트 루트에 `CLAUDE.md`를 작성하여 아키텍처와 컨벤션을 정의
2. 플러그인이 프로젝트의 `CLAUDE.md`를 자동으로 읽어 컨벤션에 맞게 작업
3. `./gradlew spotlessApply`로 코드 포맷팅, `./gradlew test`로 테스트

### 개발 워크플로우

```
/dev [JIRA-123] 상품 목록 정렬 기능 추가
```

PRD → 설계 → 구현 → 리뷰 → 커밋/PR까지 자동 수행합니다.

### 커밋 & PR

```
/commit
/pull-request
```

---

## 디렉토리 구조

```
project-command-center/
├── CLAUDE.md              ← 플러그인 소개 및 작업 범위
├── README.md              ← 지금 이 파일
├── .gitignore
│
├── .claude/
│   ├── workspace.json     ← 워크스페이스 설정 (팀, GH, 프로젝트 목록)
│   ├── settings.json      ← 팀 공유 권한/훅 설정
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
cd project-command-center
claude
```

**이전 작업을 이어서 하고 싶으면?**
Claude Code 실행 후 `/resume`을 입력하면 이전 세션 목록이 나옵니다.

**`/setup`을 다시 실행해도 되나요?**
네. 이미 설치된 것은 건너뛰고, 새로 추가된 것만 처리합니다.
